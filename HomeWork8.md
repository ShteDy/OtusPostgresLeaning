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
```

вторая сессия

```bash
locks=# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*# COMMIT;
COMMIT
locks=# 
```

проверим журнал сервера
```bash
ubuntu@otuscoursepostgre2:~$ cat /var/log/postgresql/postgresql-15-main.log | grep deadlock
2023-10-24 07:55:17.601 UTC [114085] LOG:  parameter "deadlock_timeout" changed to "200"
2023-10-25 17:53:39.852 UTC [123137] postgres@locks ERROR:  deadlock detected
ubuntu@otuscoursepostgre2:~$ 
```
в журнале сервера увидим событие об изменении параметра deadlock_timeout, а также обнаруженную блокировку

## 2. Смоделируем обновление отдной строки в разных сеансах
завустим обновление строки в трех разных сеансах
```bash
locks=# BEGIN;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
BEGIN
UPDATE 1
locks=*# 

locks=# begin; UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
BEGIN

locks=# begin ;UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
BEGIN
```

пока в первой сессии транзакция не завершена, в остальные не менаяют данные. возникает циклическое ожидание
посмотрим ощидания при помощи запроса к pg_locks и pg_stat_activity

```bash
locks=*#  SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid

    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.granted;

```

 blocked_pid | blocked_user | blocking_pid | blocking_user |                       blocked_statement                        |                               current_statement_in_blocking_process                                
-------------+--------------+--------------+---------------+----------------------------------------------------------------+----------------------------------------------------------------------------------------------------
      123353 | postgres     |       123143 | postgres      | UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;  | UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
      123143 | postgres     |       123137 | postgres      | UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2; | SELECT blocked_locks.pid     AS blocked_pid,                                                      +

(2 rows)


## vfv

