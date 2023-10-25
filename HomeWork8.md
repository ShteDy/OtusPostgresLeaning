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
## Также включим параметр log_lock_waits, он необходим для записи в журнал события дольше чем deadlock_timeout
применим конфигурацию

```bash
SELECT pg_reload_conf();
postgres=# 
```
## смоделируем запись в журнале, создадим таблицу, добавим в нее данные, проапдейтим ее строку в одной сессии, и ту же строку в другой
```bash
CREATE DATABASE locks

postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
CREATE TABLE
INSERT 0 3
locks=# BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*# SELECT pg_sleep(1);
COMMIT;
 pg_sleep 
----------
 
(1 row)

COMMIT
locks=# 
'''

вторая сессия
```bash
locks=# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*# COMMIT;
COMMIT
locks=# 
''

проверим журнал сервера
```bash
ubuntu@otuscoursepostgre2:~$ cat /var/log/postgresql/postgresql-15-main.log | grep deadlock
2023-10-24 07:55:17.601 UTC [114085] LOG:  parameter "deadlock_timeout" changed to "200"
2023-10-25 17:53:39.852 UTC [123137] postgres@locks ERROR:  deadlock detected
ubuntu@otuscoursepostgre2:~$ 
''
