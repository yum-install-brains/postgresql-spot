## Monitoring
Более подробная информация доступна в документации – https://postgrespro.ru/docs/postgrespro/10/monitoring-stats.

- Состояние запросов в динамике (выборка по 5 самым долгим)  
```sql
select datname, usename, pid as process_pid,
coalesce(client_hostname, client_addr::text) as client,
backend_type, now() - query_start as query_duration, state, wait_event, wait_event_type,
query
from pg_stat_activity order by 6 desc nulls last limit 5;
```

| Столбец | Описание     |
| :------------- | :------------- |
| backend_type       | Тип текущего серверного процесса. Возможные варианты: autovacuum launcher, autovacuum worker, background worker, background writer, client backend, checkpointer, startup, walreceiver, walsender и walwriter.       |
| state | Общее текущее состояние этого серверного процесса. |
| wait_event | Имя ожидаемого события, если обслуживающий процесс находится в состоянии ожидания, а в противном случае — NULL. |
|wait_event_type | Тип события, которого ждёт обслуживающий процесс, если это имеет место; в противном случае — NULL. |

Более подробно про возможные значения и их описание можно прочитать в документации - https://postgrespro.ru/docs/postgrespro/10/monitoring-stats#PG-STAT-ACTIVITY-VIEW.

- Мониторинг состояния реплики (куда стримим, лаг в секундах и мегабайтах)  
```sql
select coalesce(client_hostname, client_addr::text) as client,
         state,
         replay_lag as replication_lag,
         pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / (1024 * 2) as replication_lag_megabytes
from pg_stat_replication;
```

- функции для получения lsn текущего WAL и последнего переданного на standby WAL
```sql
select pg_current_wal_lsn();
select pg_last_wal_receive_lsn();
```

- Получаем список WAL файлов на сервере  
```sql
select name, size, modification
from pg_ls_waldir()
order by modification desc limit 5;
```

- Подсчет числа транзакций в базе данных  
```sql
select pg_stat_reset();
-- ждем 1 час или сколько нужно
select datname, xact_commit
from pg_stat_database; -- это и есть число транзакций за 1 час (или выжданное время)
```

- Смотрим как используется таблица
```sql
select
seq_scan, seq_tup_read,
idx_scan, idx_tup_fetch,
n_tup_ins, n_tup_upd, n_tup_del,
n_tup_hot_upd,
n_live_tup, n_dead_tup,
last_autovacuum
from pg_stat_all_tables where schemaname = 'schema_name' and relname = 'rel_name';
```

| Столбец |Описание     |
| :------------- | :------------- |
| seq_scan       | Количество последовательных чтений, запущенных по этой таблице       |
| seq_tup_read       | Количество "живых" строк, прочитанных при последовательных чтениях       |
| idx_scan       | Количество сканирований по индексу, запущенных по этой таблице       |
| idx_tup_fetch       | Количество "живых" строк, отобранных при сканированиях по индексу       |
| n_tup_ins       | Количество вставленных строк       |
| n_tup_upd       | Количество изменённых строк (включая изменения по схеме HOT)       |
| n_tup_del       | Количество удалённых строк       |
| n_tup_hot_upd       | Количество строк, обновлённых в режиме HOT (т. е. без отдельного изменения индекса)       |
| n_live_tup       | Оценочное количество "живых" строк       |
| n_dead_tup       | Оценочное количество "мёртвых" строк|
| last_autovacuum       | Время последней очистки таблицы фоновым процессом автоочистки       |

- Cмотрим, как используется индекс  
```sql
select
schemaname, relname,
idx_scan,
idx_tup_read, -- возвращено строк
idx_tup_fetch -- возвращено "живых" строк только при простом сканировании (без учета bitmap и тд)
from pg_stat_all_indexes
where indexrelname = 'index_name';
```

- Смотрим, как работает таблица с диском  
```sql
select
heap_blks_read, heap_blks_hit,
idx_blks_read, idx_blks_hit,
toast_blks_read, toast_blks_hit,
tidx_blks_read as toast_idx_blcks_read,
tidx_blks_hit as toast_idx_blcks_hit
from pg_statio_all_tables
where schemaname = 'schema_name'
and relname = 'rel_name';
```

- Смотрим, как работает индекс с диском  
```sql
select
schemaname, relname,
idx_blks_read, idx_blks_hit
from pg_statio_all_indexes
where indexrelname = 'index_name';
```
