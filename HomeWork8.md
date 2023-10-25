## Работа с блокировками
## настроим сервер для фиксации блокировок более 200 мс

```bash
postgres=# ALTER SYSTEM SET deadlock_timeout TO 200;
ALTER SYSTEM
postgres-# ;
 deadlock_timeout 
------------------
 200ms
(1 row)
```
##Также включим параметр log_lock_waits, он необходим для записи в журнал события дольше чем deadlock_timeout
## применим конфигурацию

```bash
SELECT pg_reload_conf();
postgres=# 
```
# смоделирем запись в журнале, создадим таблицу, добавим в нее данные, проапдейтим их в другой сесии,а в третей создадим индекс. 
CREATE DATABASE
postgres=# \c
You are now connected to database "postgres" as user "postgres".
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# 
