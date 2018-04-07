## psql

- Информация про таблицу  
```
\d+ pgbench_accounts
                                   Table "public.pgbench_accounts"
   Column    |     Type      | Collation | Nullable | Default | Storage  | Stats target | Description
-------------+---------------+-----------+----------+---------+----------+--------------+-------------
 aid         | integer       |           | not null |         | plain    |              |
 bid         | integer       |           |          |         | plain    |              |
 abalance    | integer       |           |          |         | plain    |              |
 filler      | character(84) |           |          |         | extended |              |
 test_column | text          |           |          |         | extended |              |
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
Options: fillfactor=100
```

- Информация про индекс  
```
\di+ pgbench_accounts_pkey
                                      List of relations
 Schema |         Name          | Type  |  Owner   |      Table       |  Size  | Description
--------+-----------------------+-------+----------+------------------+--------+-------------
 public | pgbench_accounts_pkey | index | postgres | pgbench_accounts | 107 MB |
```

- Информация про представление  
```
\dv+ pg_user_mappings
                            List of relations
   Schema   |       Name       | Type |  Owner   |  Size   | Description
------------+------------------+------+----------+---------+-------------
 pg_catalog | pg_user_mappings | view | postgres | 0 bytes |
```

Дополнительные команды  

| Команда | Описание     |
| :------------- | :------------- |
| \dt+ *.*      | Список всех таблиц и краткая информация по ним во всех схемах БД       |
| \di+ *.*      | Список всех индексов и краткая информация по ним во всех схемах БД       |
| \dv+ *.*      | Список всех представлений и краткая информация по ним во всех схемах БД       |
| \dp schema_name.table_name     | Список всех прав доступа        |
| \df+ *.*      | Список всех функций и краткая информация по ним во всех схемах БД       |
| \l      | Список всех БД кластера       |
