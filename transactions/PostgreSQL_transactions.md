## Transaction isolation (I in ACID)

### Виды нарушения изоляции
https://en.wikipedia.org/wiki/Isolation_%28database_systems%29#Dirty_reads

- Dirty read – Транзакция 1 делает update данных в таблице. Параллельная транзакция 2 сразу видит изменения в таблице, сделанные транзакцией 1 (даже не смотря на то, что транзакция 1 еще на сделала commit).
- Non-repeatable read – два чтения в транзакции 1 по одной и той же строке могут вернуть два разных результата. Допустим, транзакция 1 читает строку первый раз. Затем, параллельная транзакция 2 изменяет эначения полей в этой строке. При повторном чтении, транзакция 1 уже читает измененное значение строки (получаем non-repeatable чтение).
- Phantom read – В рамках одной транзакции, один и тот же запрос возвращает разный результирующий набор строк.

Non-repeatable чтение изменяет значение в одной и той же строке при повторном выполнении запроса, в то время как фантомное чтение приводит к тому, что в результате выполнения запроса второй раз, читается другой набор строк (сами значения строк не изменяются, но к предыдущему результату выполнения добавляются новые строки или удаляются старые).

#### Сопоставление уровней изоляции транзакции и видов нарушения изоляции
| Уровень изоляции | Dirty read | Non-repeatable read | Phantom read |
| :------------- | :------------- | :------------- | :------------- |
| Read uncommitted | Может возникнуть | Может возникнуть | Может возникнуть |
| Read committed | Не возникает | Может возникнуть | Может возникнуть |
| Repeatable read | Не возникает | Не возникает | Может возникнуть |
| Serializable | Не возникает | Не возникает | Не возникает |

#### Примеры
```sql
create table test (id int, city text);


insert into test values(1, 'Moscow');
insert into test values(2, 'Piter');
```

```sql
-- Пример Non-repeatable read нарушения изоляции
begin transaction isolation read committed;
select city from test where id = 1;
city
--------
Moscow
                                                begin transaction isolation read committed;
                                                update test set city = 'Msk' where id = 1;
                                                commit;
select city from test where id = 1;
city
------
Msk
commit;                                                
```

```sql
-- Пример защиты уровня изоляции от Non-repeatable read
begin transaction isolation level repeatable read;
select city from test where id = 1;
city
--------
Msk
                                                begin transaction isolation level repeatable read;
                                                update test set city = 'Moscow' where id = 1;
                                                commit;
select city from test where id = 1;
city
------
Msk
commit;

select city from test where id = 1;
city
------
Moscow
```
