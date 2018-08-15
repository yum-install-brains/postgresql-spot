## Приемы разной степени сомнительности и полезности

### Временные переменные в рамках транзакции
Значение `true` в `set_config` говорит о том, что переменная устанавливается локально (в рамках жизни транзакции).
```sql
BEGIN;

SELECT set_config('test.test_var', (SELECT col FROM some_table LIMIT 1), True);
SELECT set_config('test.another_test_var', (SELECT 'some value'), True);
SELECT current_setting('test.test_var');
 current_setting
-----------------
 a
SELECT current_setting('test.another_test_var');
 current_setting
-----------------
some value

COMMIT;
SELECT current_setting('test.test_var');
 current_setting
-----------------

(1 row)
SELECT current_setting('test.another_test_var');
 current_setting
-----------------

(1 row)
```

### Использование переменных (decorated literal expression) в psql
```sql
\set start '2017-02-01'
SELECT date:'start';
    date
------------
 2017-02-01
(1 row)
SELECT date_col
FROM test_table
WHERE date_col >= date :'start'
  AND date_col < date :'start' + interval '1 month';
```

### ALTER типа поля вместе с нормализацией значений
```sql
ALTER TABLE test_table
  ALTER col_one
  type bigint
  using replace(col_one, ',', '')::bigint,

  ALTER col_two
  type bigint
  using replace(col_two, ',', '')::bigint,

  ALTER col_three
  type bigint
  using substring(replace(col_three, ',', '') from 2)::numeric;
```
