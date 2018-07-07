## Онлайн операции на больших данных
### Добавление ограничений CHECK и FOREIGN KEY на таблицу
При добавлении ограничений на таблицу PostgreSQL берет `AccessExclusiveLock`. Этот лок гарантирует, что кроме транзакции, получившей эту блокировку, никакая другая транзакция не может обращаться к таблице каким-либо способом. Соответственно, все остальные транзакции повиснут до момента завершения транзакции с ограничением (для этого она должна просканировать всю таблицу).
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

Но можно этого избежать и изменить уровень блокировки на менее строгий. Для этого сперва нужно добавить `NOT VALID` (проверка существующих в таблице данных не осуществляется, при новых операциях с таблицей проверка уже будет работать) ограничение и затем запустить операцию `VALIDATE`. На данный момент `NOT VALID` работает только с `CHECK` и `FOREIGN KEY`. В таком случае лок будет уже `ShareUpdateExclusiveLock` (блокирует `VACUUM`, создание индексов и многие операции `ALTER`) на время проверки и `AccessExclusiveLock` на время размещения самого ограничения.
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

### Добавление нового поля в таблицу
Операция происходит на уровне системного каталога и выполняется мгновенно. Тем не менее, на время выполнения операции берется `AccessExclusiveLock`, поэтому необходимо следить за блокировками.

Особым случаем является добавление нового поля с `DEFAULT` значением. На больших объемах данных эта операция может занять продолжительное время, т.к. PostgreSQL должен записать дефолтное значение во все строки таблицы. Оба варианта операции ниже берут `AccessExclusiveLock` на все время выполнения операции:
```sql
BEGIN;
ALTER TABLE clients ADD COLUMN test_col text DEFAULT 'test';
SELECT locktype, mode FROM pg_locks WHERE pid = pg_backend_pid() AND relation = 'clients'::regclass;
┌──────────┬─────────────────────┐
│ locktype │        mode         │
├──────────┼─────────────────────┤
│ relation │ ShareLock           │
│ relation │ AccessExclusiveLock │
└──────────┴─────────────────────┘
COMMIT;
```
```sql
BEGIN;
ALTER TABLE clients ALTER COLUMN test_col SET DEFAULT 'text';
SELECT locktype, mode FROM pg_locks WHERE pid = pg_backend_pid() AND relation = 'clients'::regclass;
┌──────────┬─────────────────────┐
│ locktype │        mode         │
├──────────┼─────────────────────┤
│ relation │ AccessExclusiveLock │
└──────────┴─────────────────────┘
COMMIT;
```
Для того, чтобы не держать `AccessExclusiveLock` долго, лучше в отдельной транзакции добавить новое поле, затем командой `UPDATE` (с `ROW EXCLUSIVE` локом, который значительно слабее) заполнить его дефолтными значениями и после этого применить операцию `SET DEFAULT`, которая также пройдет быстро.

---
**NOTE**

В версии PostgreSQL 11 оптимизирована работа операции `DEFAULT`, теперь она происходит также быстро, как и добавление нового поля и выполняется на уровне системного каталога (верно для не `VOLATILE` значений). Подробнее в статье [Depesz](https://www.depesz.com/2018/04/04/waiting-for-postgresql-11-fast-alter-table-add-column-with-a-non-null-default/).

---

### Изменение типа поля
Изменение типа поля может также надолго заблокироавть таблицу. В данном случае все зависит от изменяемого типа. Например, расширение `varchar` пройдет быстро. Но для типов, требующих перезаписи значения, придется изменить все значения поля в таблице.

### Удаление поля
Удаление поля происходит на уровне каталога и является быстрой операцией. Однако, надо помнить, что до выполнения `VACUUM FULL` высвобожденное место не возвращается системе.

### pg_repack
pg_repack – утилита, которая позволяет проводить операции `VACUUM FULL`, `CLUSTER`, `REINDEX` и `SET TABLESPACE` в онлайне. При этом нужно помнить, что pg_repack требует примерно x2 таблицы + всех ее индексов свободного места для выполнения.

Подробнее про механизм работы pg_repack можно прочитать на [официальной странице расширения](http://reorg.github.io/pg_repack/) в разделе Details.

### Источники
- [Блог](https://blog.2ndquadrant.com/how-to-check-the-lock-level-taken-by-operations-in-postgresql/) 2ndquadrant.com
- [Блог](https://leopard.in.ua/2016/09/20/safe-and-unsafe-operations-postgresql#.Wz963JL4ksl) leopard.in.ua
- [Документация](https://www.postgresql.org/docs/current/static/explicit-locking.html) PostgreSQL
