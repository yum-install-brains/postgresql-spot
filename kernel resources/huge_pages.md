## Huge pages

Huge pages есть смысл включать в тех случаях, когда под PostgreSQL выделены десятки и сотни гигабайт оперативной памяти. Они помогают сократить накладные расходы на адресацию памяти и поддержание страничных таблиц.

### Включаем huge pages
Сперва проверим, сколько нам нужно huge pages:
 ```
# определяем PID postmaster'а
head -1 $PGDATA/postmaster.pid
1640

# определяем пик выделенной оперативной памяти
grep ^VmPeak /proc/1640/status
VmPeak:   344340 kB
 ```

По умолчанию 1 huge page имеет размер 2048 kB. Таким образом нам нужно  344340/2048 = 169 больших страниц. Зададим это значение в файле `/etc/sysctl.conf`.

Далее отключим transparent huge pages – механизм, который автоматически выделяет huge pages по запросу. Transparent huge pages может приводить к деградации системы из-за постепеннной фрагментации памяти.
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```


Теперь проверим, сколько у нас доступно huge pages:
```
cat /proc/meminfo | grep -i huge
AnonHugePages:      4096 kB
HugePages_Total:     170
HugePages_Free:      170
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

В PostgreSQL за использование huge pages отвечает параметр `huge_page = [off|on|try]`. Установим значение этого параметра в on и запустим кластер БД.

### Ошибки
```
FATAL: could not map anonymous shared memory: Cannot allocate memory
HINT: This error usually means that PostgreSQL's request for a shared memory segment exceeded available memory, swap space or huge pages. To reduce the request size (currently 148324352 bytes), reduce PostgreSQL's shared memory usage, perhaps by reducing shared_buffers or max_connections.
```
Если при старте появляется такая ошибка, значит скорее всего процессу не хватило свободных больших страниц (проверить HugePages_Free). Если при установке параметра `huge_pages=off` проблема уходит, то дело именно в этом.

### Источники
- [Блог](https://blog.dbi-services.com/configuring-huge-pages-for-your-postgresql-instance-redhatcentos-version/) DBI;
- [Статья](https://habr.com/post/228793/) Алексея Лесовского на хабре
