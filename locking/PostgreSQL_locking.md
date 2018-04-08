## Explicit locking
https://postgrespro.ru/docs/postgrespro/10/explicit-locking
#### Режимы блокировки на уровне таблицы
Разные транзакции могут в любой момент времени владеть блокировками неконфликтующих типов, но не могут владеть блокировками конфликтующих типов на одну и ту же таблицу.

| Режим          | С кем конфликтует| Пример операции |
| :------------- | :-------------   |:------------- |
| ACCESS SHARE   | ACCESS EXCLUSIVE |SELECT         |
| ROW SHARE   | EXCLUSIVE, ACCESS EXCLUSIVE |SELECT FOR UPDATE, SELECT FOR SHARE         |
| ROW EXCLUSIVE   | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |SELECT         |
| SHARE UPDATE EXCLUSIVE   | SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY, CREATE STATISTICS, ALTER TABLE VALIDATE         |
| SHARE   | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |CREATE INDEX         |
| SHARE ROW EXCLUSIVE   | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |CREATE COLLATION, CREATE TRIGGER, многие формы ALTER TABLE         |
| EXCLUSIVE   | ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE (т.е. совместим только с чтением таблицы) | REFRESH MATERIALIZED VIEW CONCURRENTLY         |
| ACCESS EXCLUSIVE   | Конфликтует со всеми режимами блокировок и сам с собой | DROP TABLE, TRUNCATE, REINDEX, CLUSTER, VACUUM FULL, REFRESH MATERIALIZED VIEW, многие формы ALTER TABLE         |

#### Режимы блокировки на уровне строки
| Режим          | С кем конфликтует| Пример операции |
| :------------- | :-------------   |:------------- |
| FOR UPDATE   | Конфликтует со всеми режимам и сам с собой. Конфликтует с операциями UPDATE, DELETE, SELECT FOR UPDATE, SELECT FOR NO KEY UPDATE, SELECT FOR SHARE или SELECT FOR KEY SHARE |SELECT FOR UPDATE         |
| FOR NO KEY UPDATE   | FOR SHARE, FOR NO KEY UPDATE, FOR UPDATE | Запрашивается любой командой UPDATE, которая не требует блокировки FOR UPDATE. |
| FOR SHARE   | Не позволяет другим транзакциям выполнять с этими строками UPDATE, DELETE, SELECT FOR UPDATE или SELECT FOR NO KEY UPDATE, но допускает SELECT FOR SHARE и SELECT FOR KEY SHARE. Конфликтует с режимами FOR NO KEY UPDATE, FOR UPDATE | SELECT FOR SHARE и SELECT FOR KEY SHARE |
| FOR KEY SHARE   | Блокируется SELECT FOR UPDATE, но не SELECT FOR NO KEY UPDATE. не позволяет другим транзакциям выполнять команды DELETE и UPDATE, только если они меняют значение ключа (но не другие UPDATE), и при этом допускает выполнение команд SELECT FOR NO KEY UPDATE, SELECT FOR SHARE и SELECT FOR KEY SHARE. | SELECT FOR NO KEY UPDATE, SELECT FOR SHARE и SELECT FOR KEY SHARE |
