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

#### Пример 1
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

#### Пример 2 (phantom read и сериализация)
https://vladmihalcea.com/a-beginners-guide-to-the-phantom-read-anomaly-and-how-it-differs-between-2pl-and-mvcc/  
Допустим, Элис хочет повысить зарплату всем сотрудникам отдела на 10%, в это же время Боб нанимает нового сотрудника в отдел. Бюджет отдела ограничен 370 пиастрами. Текущие сотрудники в сумме получают 300 пиастр. Новый сотрудник должен получать 70 пиастр.  
```sql
create table employee (id int, salary int, department int);

insert into employee
  values (1, 100, 1), (2, 150, 1), (3, 50, 1);
```

Элис и Боб одновременно проверяю, сколько осталось свободных денег в бюджете. Затем Боб подписывает соглашение с новым сотрудником, а через несколько секунд Элис увеличивает зарплату всем старым сотрудникам. В итоге получают перерасход бюджета 30 пиастр.  
```sql
begin transaction isolation level Serializable;
-- Alice хочет увеличить зарплату всех сотрудников на 10%
select sum(salary)
from employee
where department = 1;
--
300
                                                      begin transaction isolation level Serializable;
                                                      -- Bob в это же время нанимает нового сотрудника
                                                      select sum(salary)
                                                      from employee
                                                      where department = 1;
                                                      --
                                                      300

                                                      insert into employee
                                                        values (4, 70, 1);
                                                      commit;
update employee set salary = salary * 1.1 where department = 1;
--
UPDATE 3
commit;
select sum(salary) from employee;
sum
--
400
```

Для предотвращения такой ситуации можно использовать Serializable уровень изоляции транзакций, при котором транзакции могут успешно завершиться в том и только том случае, если они бы смогли успешно завершиться ровно с тем же итогом, выполняясь последовательно.  
```sql
drop table employee;
create table employee (id int, salary int, department int);

insert into employee
  values (1, 100, 1), (2, 150, 1), (3, 50, 1);
```

Транзакция Элис теперь не сможет завершиться, так как в последовательном выполнении транзакций Элис и Боба друг за другом Боб получил бы остаток бюджета равный 40 пиастрам, а он уже получил остаток равный 70 пиастрам. В этом случае транзакция Элис получит ошибку сериализации и не выполнится. Бюджет останется неперерасходованным.  
```sql
begin transaction isolation level Serializable;
-- Alice хочет увеличить зарплату всех сотрудников на 10%
select sum(salary)
from employee
where department = 1;
--
300
                                                      begin transaction isolation level Serializable;
                                                      -- Bob в это же время нанимает нового сотрудника
                                                      select sum(salary)
                                                      from employee
                                                      where department = 1;
                                                      --
                                                      300

                                                      insert into employee
                                                        values (4, 70, 1);
                                                      commit;
update employee set salary = salary * 1.1 where department = 1;
--
ERROR:  40001: could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during write.
HINT:  The transaction might succeed if retried.
LOCATION:  OnConflict_CheckForSerializationFailure, predicate.c:4677
rollback;
select sum(salary) from employee;
sum
--
370
```
