## Онлайн операции на больших данных
### Добавление ограничений на таблицу
При добавлении ограничений на таблицу PostgreSQL берет AccessExclusiveLock. Этот лок гарантирует, что кроме транзакции, получившей эту блокировку, никакая другая транзакция не может обращаться к таблице каким-либо способом. Соответственно, все остальные транзакции повиснут до момента завершения транзакции с ограничением (для этого она должна просканировать всю таблицу).
```sql
BEGIN;
ALTER TABLE clients ADD CONSTRAINT clients_id_check CHECK (id > 0);
SELECT locktype, mode FROM pg_locks WHERE pid = pg_backend_pid() AND relation = 'clients'::regclass;
┌──────────┬─────────────────────┐
│ locktype │        mode         │
├──────────┼─────────────────────┤
│ relation │ AccessExclusiveLock │
└──────────┴─────────────────────┘
COMMIT;
```

Но можно этого избежать и изменить уровень блокировки на менее строгий. Для этого сперва нужно добавить NOT VALID (проверка существующих в таблице данных не осуществляется, при новых операциях с таблицей проверка уже будет работать) ограничение и затем запустить операцию VALIDATE. В таком случае лок будет уже ShareUpdateExclusiveLock (блокирует VACUUM, создание индексов и многие операции ALTER) на время проверки и AccessExclusiveLock на время размещения самого ограничения.
```sql
BEGIN;
ALTER TABLE clients ADD CONSTRAINT clients_id_check CHECK (id > 0) NOT VALID;
SELECT locktype, mode FROM pg_locks WHERE pid = pg_backend_pid() AND relation = 'clients'::regclass;
┌──────────┬─────────────────────┐
│ locktype │        mode         │
├──────────┼─────────────────────┤
│ relation │ AccessExclusiveLock │
└──────────┴─────────────────────┘
COMMIT;

BEGIN;
ALTER TABLE clients VALIDATE CONSTRAINT clients_id_check;
SELECT locktype, mode FROM pg_locks WHERE pid = pg_backend_pid() AND relation = 'clients'::regclass;
┌──────────┬──────────────────────────┐
│ locktype │           mode           │
├──────────┼──────────────────────────┤
│ relation │ ShareUpdateExclusiveLock │
└──────────┴──────────────────────────┘
COMMIT;
```
