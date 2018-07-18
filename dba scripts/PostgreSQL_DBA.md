## Полезные запросы
- Вставляем метаданные Liquibase  
```sql
delete from databasechangelog
where id = 'JIRA-000' and author = 'yum_install_brains' and filename = 'dir/file.xml';
--
insert into databasechangelog (id,author,filename,dateexecuted,orderexecuted,exectype,md5sum,description,comments,tag,liquibase)
values ('JIRA-000','yum_install_brains','dir/file.xml',now(),1,'EXECUTED',null,'sql','',null,'3.5.3');
```

- Получаем лог выполнения pgagent джобов  
```sql
select *
from pgagent.pga_joblog
where jlgjobid
  in (select jobid
    from pgagent.pga_job
    where jobname = 'job_name')
order by jlgstart desc
limit 1;
```

- Какие и кому привелегии выданы на таблицу
```sql
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name='pgbench_accounts';
┌───────────┬────────────────┐
│  grantee  │ privilege_type │
├───────────┼────────────────┤
│ postgres  │ INSERT         │
│ postgres  │ SELECT         │
│ postgres  │ UPDATE         │
│ postgres  │ DELETE         │
│ postgres  │ TRUNCATE       │
│ postgres  │ REFERENCES     │
│ postgres  │ TRIGGER        │
│ test_user │ INSERT         │
│ test_user │ SELECT         │
└───────────┴────────────────┘
```

- COPY шаблон  
```sql
COPY (
  sql_text
)
TO '/dir/copy.csv' WITH (format csv, header true, delimiter ';', null 'null', encoding 'WIN1251');
```

- Мониторинг блокирующих запросов  
```sql
SELECT relation::regclass, * FROM pg_locks WHERE NOT GRANTED;
```

- Удаляем все символы переноса строки из текстового поля (для корректной выгрузки в Excel)  
```sql
select regexp_replace(field, E'[\\n\\r]+', ' ', 'g' )
```

- Обнуление статистики pg_stat_statements  
```sql
select pg_stat_statements_reset();
```

- Получение пути к файлу таблицы   
```sql
select pg_relation_filepath('pgbench_accounts');
pg_relation_filepath
--
 base/12994/16504
```

- Получение имени таблицы по OID tablespace + OID таблицы  
```sql
select pg_filenode_relation(0, 16504);
pg_filenode_relation
--
pgbench_accounts
```

- Читаем конфигурационный файл (или любой другой) из директории PGDATA или PGLOG
```sql
select pg_read_file('postgresql.conf');
```

- Создание внешнего сервера и пользовательских маппингов  
```sql
DROP SERVER IF EXISTS foreign_db CASCADE;
CREATE SERVER foreign_db FOREIGN DATA WRAPPER dblink_fdw OPTIONS (host 'host-name', dbname 'foreign_db');
CREATE USER MAPPING FOR user_name SERVER foreign_db OPTIONS (USER 'foreign_user', PASSWORD 'foreign_pass');
GRANT USAGE ON FOREIGN SERVER foreign_db TO user_name;
```

- Получаем текст функции  
```sql
select pg_get_functiondef(select oid from pg_proc where proname = 'func_name');
```

- Множественные условия в LIKE запросе  
```sql
SELECT * FROM test_table
WHERE attr_name LIKE
  ANY (ARRAY['foo%', '%bar%', 'ba z.', 'daz%'])
```

- Создание таблицы с использованием INTO  
```sql
SELECT attr1, attr2, attr3
INTO new_table
FROM old_table;
```

- Скрипт пересоздания первичного ключа "на лету"  
```sql
BEGIN;
CREATE UNIQUE INDEX CONCURRENTLY tab_pkey_idx2 ON tab(id);
ALTER TABLE tab
   DROP CONSTRAINT tab_pkey CASCADE,
   ADD CONSTRAINT tab_pkey PRIMARY KEY USING INDEX tab_pkey_idx2;
ALTER TABLE second_tab
   ADD CONSTRAINT second_tab_fkey FOREIGN KEY (tab_id) REFERENCES tab(id) NOT VALID;
COMMIT;
```

- Распаковываем массивы в таблицы и проставляем каждой строке номер  
```sql
SELECT *
FROM unnest('{1,2,3}'::int[], '{4,5,6,7}'::int[])
WITH ORDINALITY AS t(a1, a2, num)
ORDER BY t.num DESC;
  a1  | a2 | num
------+----+-----
 null |  7 |   4
    3 |  6 |   3
    2 |  5 |   2
    1 |  4 |   1
```

- Убиваем подключения к БД  
```sql
select pg_terminate_backend(pid)
from pg_stat_activity
where pid <> pg_backend_pid();
```

- Создаем JSON  
```sql
select
  row_to_json(t1)
from (
select
  'joe' as username,
  (select project from (values(1, 'prj1')) as project(project_id, project_name)) as project
) t1;
--
row_to_json
{"username":"joe","project":{"project_id":1,"project_name":"prj1"}}
```

- Действия с массивами  
```sql
WITH my_table (ID, year, any_val) AS (
     VALUES (1, 2017,56)
     ,(2, 2017,67)
     ,(3, 2017,12)
     ,(4, 2017,30)
     ,(5, 2020,8)
     ,(6, 2030,17)
     ,(7, 2030,50)
     )
SELECT year
,array_agg(any_val) -- собираю данные (по каждому году) в массив
,array_agg(any_val ORDER BY any_val) AS sort_array_agg -- порядок элементов можно отсортировать (с  9+ версии Postgres)
,array_to_string(array_agg(any_val),';') -- преобразовываю массив в строку
,ARRAY['This', 'is', 'my' , 'array'] AS my_simple_array -- способ создания массива
FROM my_table
GROUP BY year; -- группируем данные по каждому году
--
year |   array_agg   | sort_array_agg | array_to_string |  my_simple_array
------+---------------+----------------+-----------------+--------------------
 2017 | {56,67,12,30} | {12,30,56,67}  | 56;67;12;30     | {This,is,my,array}
 2020 | {8}           | {8}            | 8               | {This,is,my,array}
 2030 | {17,50}       | {17,50}        | 17;50           | {This,is,my,array}
```

## Немного bash
- Создание дампа схемы (без данных)  
```
pg_dump -t 'schema_name.table_name' --schema-only database_name
```

- Восстановление из дампа  
```
psql -d database_name < table.dump
```  
или  
```
pg_restore --table=table_name table.dump
```

- Мониторинг размера темповых файлов  
```
watch df -h /dir/PG_version_oid/pgsql_tmp
```

- Флаг для остановки группы скриптов при ошибке в одном из скриптов  
```
psql -v ON_ERROR_STOP=1
```  
или  
```
\set ON_ERROR_STOP on
```

- Информация по объектам и их правам  
```
\dfS+
```  
или  
```
\dtS+
```

- Выводим сумму в байтах темповых файлов, сгенеренных в логе  
```
cat postgresql_file.log | grep "LOG:  temporary file:" > temps.txt && awk '{s+=$15}END{print s}' temps.txt
```

- Массовое выполнение скриптов по цифровому порядку в их именах (1.sql, 2.sql, 3.sql)  
```
for i in $(ls -1 /dir/*.sql|sort -n) ;do psql -d database_name -v ON_ERROR_STOP=ON -f $i ;done
```

## Напоминалки
- порядок выполнения операций в запросе SELECT  
```sql
FROM
WHERE
GROUP BY
HAVING
SELECT
ORDER BY
```

- Drop column на самом деле не удаляет данные в колонке и колонку  
The DROP COLUMN form does not physically remove the column, but simply makes it invisible to SQL operations.
Subsequent insert and update operations in the table will store a null value for the column. Thus, dropping a column is quick but it will not immediately reduce
the on-disk size of your table, as the space occupied by the dropped column is not reclaimed. The space will be reclaimed over time as existing rows are updated.

  To force an immediate rewrite of the table, you can use VACUUM FULL, CLUSTER or one of the forms of ALTER TABLE that forces a rewrite.
  This results in no semantically-visible change in the table, but gets rid of no-longer-useful data.

- Использование лидирующих % в запросах LIKE  
For this you need to install pg_trgm and create GIN/GIST index on field.
https://stackoverflow.com/questions/1566717/postgresql-like-query-performance-variations
