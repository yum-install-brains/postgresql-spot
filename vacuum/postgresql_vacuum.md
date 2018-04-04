## VACUUM

- [VACUUM](#VACUUM)
	- [Ссылки](#)
	- [Введение](#)
	- [VACUUM vs VACUUM FULL](#VACUUM-vs-VACUUM-FULL)
	- [Неблокирующий VACUUM](#-VACUUM)
		- [Блок 1](#-1)
		- [Блок 2](#-2)
		- [Блок 3](#-3)
	- [Visibility Map](#visibility-map)
		- [Ленивый режим](#-)
		- [Жадный режим](#-)
		- [Запрос для получения pg_class.relfrozenxid и pg_database.datfrozenxid](#-pgclassrelfrozenxid-pgdatabasedatfrozenxid)
	- [AUTOVACUUM](#AUTOVACUUM)

<!-- /TOC -->

#### Ссылки
[VACUUM processing](http://www.interdb.jp/pg/pgsql06.html)

#### Введение
VACUUM – это процесс обслуживания PostgreSQL, двумя основными задачами коготоро являются **удаление мертвых строк** и **заморозка идентификаторов транзакций**. Процесс можно запустить руками, но также он запускается и автоматически демоном AUTOVACUUM.

Есть две разновидности VACUUM:
- Неблокирующий VACUUM (или просто VACUUM)
  - удаляет мертвые строки на всех страницах файла таблицы,
  - удаляет строки индекса, ссылающиеся на мертвые строки таблицы,
  - дефрагментирует живые строки в рамках каждой страницы,
  - замораживает старые идентификаторы транзакций по необходимости,
  - обновляет замороженный идентификатор транзакции (frozen txid),
  - обновляет FSM и VM,
  - обновляет статистику (pg_stat_all_tables и тд).

  При этом не блокируя другие транзакции к этой таблице.

- Блокирующий VACUUM (VACUUM FULL):
  - удаляет мертвые строки и индексы, ссылающиеся на мертвые строки,
  - дефрагментирует живые строки во всем файле (т.е. дефрагментирует сами страницы в файле, убирая пустые или не до конца заполненные страницы и высвобождая память системе. Все дефрагментированные страницы записываются в новый файл таблицы, а старый файл после операции удаляется),
  - перестраивает все индексы по таблице,
  - обновляет FSM и VM,
  - обновляет статистику.

  При этом блокируя все транзакции к этой таблице.

При работе VACUUM должен просканировать всю таблицу, это довольно дорогая операция. В версии 8.4 была добавлена Visibility Map, которая позволила VACUUM пропускать страницы, не содержащие мертвых строк. В версии 9.6 такая же оптимизация на основе Visibility Map была сделана для процесса заморозки транзакций.

#### VACUUM vs VACUUM FULL
Псевдокод процесса VACUUM FULL
```java
(1)  FOR each table
(2)       Acquire AccessExclusiveLock lock for the table
(3)       Create a new table file
(4)       FOR each live tuple in the old table
(5)            Copy the live tuple to the new table file
(6)            Freeze the tuple IF necessary
            END FOR
(7)        Remove the old table file
(8)        Rebuild all indexes
(9)        Update FSM and VM
(10)      Update statistics
            Release AccessExclusiveLock lock
       END FOR
(11)  Remove unnecessary clog files and pages if possible
```

Псевдокод процесса неблокирующего VACUUM
```java
(1)  FOR each table
(2)       Acquire ShareUpdateExclusiveLock lock for the target table

          /* The first block */
(3)       Scan all pages to get all dead tuples, and freeze old tuples if necessary
(4)       Remove the index tuples that point to the respective dead tuples if exists

          /* The second block */
(5)       FOR each page of the table
(6)            Remove the dead tuples, and Reallocate the live tuples in the page
(7)            Update FSM and VM
           END FOR

          /* The third block */
(8)       Truncate the last page if possible
(9)       Update both the statistics and system catalogs of the target table
           Release ShareUpdateExclusiveLock lock
       END FOR

        /* Post-processing */
(10)  Update statistics and system catalogs
(11)  Remove both unnecessary files and pages of the clog if possible
```  
#### Неблокирующий VACUUM
##### Блок 1
Первый блок производит операции заморозки идентификаторов транзакций и удаления строк индексов, ссылающихся на мертвые строки таблицы.

Сперва, PostgreSQL сканирует таблицу для составления списка мертвых строк и замораживает транзакции старых строк, если возможно. Список хранится в maintenance_work_mem в оперативной памяти. Если maintenance_work_mem переполняется, PostgreSQL переходит к шагам (4)-(7), затем опять возвращается к шагу (3) и возобновляет сканирование.

После сканирования производится операция удаления строк индексов, ссылающихся на мертвые строки таблицы.

##### Блок 2
Во втором блоке происходит постраничное удаление мертвых строк и постраничное обновление FSM, VM. Процесс продолжается, пока не будут обработаны все страницы.  
![alt text](http://www.interdb.jp/pg/img/fig-6-01.png "Page structure")

##### Блок 3
После окончания VACUUM процесса, PostgreSQL обновляет статистические таблицы и таблицы системного каталога, связанные с процессом VACUUM.

##### Visibility Map
VM есть у каждой таблицы и содержит информацию по видимости страниц таблицы (1 - все строки страницы видимы, 0 - не все строки страницы видимы). В страницы, отмеченные полностью видимыми, VACUUM может не заходить (тк в них нет мертвых строк).

Начиная с версии 9.6, в VM также добавилась информация о том, заморожены ли строки на странице или нет.

Процесс заморозки может работать в двух режимах в зависимости от условий – ленивом режиме и жадном режиме. В основном заморозка происходит в ленивом режиме.

В ленивом режиме процесс заморозки сканирует только страницы, содержащие мертвые строки (используя VM).

В жадном режиме происходит полное сканирование всех страниц, независимо от наличия мертвых строк.

##### Ленивый режим
В начале процесса заморозки, PostgreSQL вычисляет

    freezeLimit_txid = (OldestXmin − vacuum_freeze_min_age)  
OldestXmin - самая старая из активных транзакций на момент запуска VACUUM. vacuum_freeze_min_age - конфигурационный параметр.

Далее, все транзакции, t_xmin которых меньше, чем freezeLimit_txid, замораживаются.

Если в момент запуска VACUUM больше нет активных транзакций, OldestXmin будет равна транзакции, в которой запустился VACUUM.  
![alt text](http://www.interdb.jp/pg/img/fig-6-03.png "Freeze")  
В примере с картинки выше Tuple 1,2,3,7,8 будут заморожены, а tuple 1,7 вдобавок почищены, т.к. они являютя мертвыми строками. Tuple 4,5,6 будут пропущены в связи с VM.

##### Жадный режим
Как было показано выше, ленивый режим может пропускать некоторые строки, требующие заморозки. Жадный режим нужен для того, чтобы компенсировать это. Жадный режим сканирует все страницы во всей таблице.

Жадный режим применяется, если удовлетворяется следующее условие

    pg_database.datfrozenxid < (OldestXmin − vacuum_freeze_table_age)  
pg_database.datfrozenxid представляет собой значение из pg_database и хранит самый старый замороженный txid для каждой базы данных.

После заморозки каждой таблицы, значение pg_class.relfrozenxid обновляется для этой таблицы. Также, после заморозки таблицы по необходимости обновляется значение pg_database.datfrozenxid.  
![alt text](http://www.interdb.jp/pg/img/fig-6-05.png "Freeze database")

Заморозку конкретной таблицы можно выполнить явно, запустив команду

    VACUUM Freeze
В таком случае будут заморожены все строки, чей t_xmin меньше OldestXmin (а не OldestXmin - vacuum_freeze_min_age в случае автовакуума).

В версии 9.6 жадный режим был оптимизирован. Теперь в VM содержится информация о страницах, содержащих только замороженные строки (они могли быть уже заморожены в ленивом режиме). Такие страницы пропускаются при заморозке.

![alt text](http://www.interdb.jp/pg/img/fig-6-06.png "eager database")

##### Запрос для получения pg_class.relfrozenxid и pg_database.datfrozenxid  
```sql
testdb=# VACUUM table_1;
VACUUM

testdb=# SELECT n.nspname as "Schema", c.relname as "Name", c.relfrozenxid
             FROM pg_catalog.pg_class c
             LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
             WHERE c.relkind IN ('r','')
                   AND n.nspname <> 'information_schema' AND n.nspname !~ '^pg_toast'
                   AND pg_catalog.pg_table_is_visible(c.oid)
                   ORDER BY c.relfrozenxid::text::bigint DESC;
   Schema   |            Name         | relfrozenxid
------------+-------------------------+--------------
 public     | table_1                 |    100002000
 public     | table_2                 |         1846
 pg_catalog | pg_database             |         1827
 pg_catalog | pg_user_mapping         |         1821
 pg_catalog | pg_largeobject          |         1821

...

 pg_catalog | pg_transform            |         1821
(57 rows)

testdb=# SELECT datname, datfrozenxid FROM pg_database WHERE datname = 'testdb';
 datname | datfrozenxid
---------+--------------
 testdb  |         1821
(1 row)
```

#### AUTOVACUUM
Автовакуум - демон автоматического запуска процедуры VACUUM. Он запускается каждые autovacuum_naptime секунд и порождает autovacuum_max_works число воркеров. Воркеры производят неблокирующий VACUUM.
