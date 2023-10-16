#Создадим виртуальную машину 2 ядра 4 ОЗУ + 10 гигов диск
#Поставим 15й Postgres
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

#Инициализируем PGBENCH
postgres@otuscoursepostgre2:/home/ubuntu$  pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.14 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.52 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.28 s, vacuum 0.07 s, primary keys 0.16 s).

#Запустим с параметрами
postgres@otuscoursepostgre2:/home/ubuntu$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 504.7 tps, lat 15.735 ms stddev 8.568, 0 failed
progress: 12.0 s, 499.7 tps, lat 16.018 ms stddev 8.038, 0 failed
progress: 18.0 s, 499.5 tps, lat 16.016 ms stddev 8.153, 0 failed
progress: 24.0 s, 499.2 tps, lat 16.020 ms stddev 8.147, 0 failed
progress: 30.0 s, 497.2 tps, lat 16.099 ms stddev 8.609, 0 failed
progress: 36.0 s, 498.7 tps, lat 16.036 ms stddev 7.964, 0 failed
progress: 42.0 s, 499.7 tps, lat 16.014 ms stddev 8.644, 0 failed
progress: 48.0 s, 499.5 tps, lat 16.012 ms stddev 8.005, 0 failed
progress: 54.0 s, 475.8 tps, lat 16.819 ms stddev 12.819, 0 failed
progress: 60.0 s, 502.7 tps, lat 15.911 ms stddev 8.640, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29867
number of failed transactions: 0 (0.000%)
latency average = 16.065 ms
latency stddev = 8.847 ms
initial connection time = 31.626 ms
tps = 497.872270 (without initial connection time)
postgres@otuscoursepostgre2:/home/ubuntu$ 

#применим параметры инстанса постгреса согласно файлу прикрепленном к заданию
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

#применим изменения и перезапустим службу postgresql
ubuntu@otuscoursepostgre2:~$ sudo nano /etc/postgresql/15/main/postgresql.conf 
ubuntu@otuscoursepostgre2:~$ sudo systemctl restart postgresql
ubuntu@otuscoursepostgre2:~$ 
#запустим pgbench
postgres@otuscoursepostgre2:/home/ubuntu$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 502.0 tps, lat 15.835 ms stddev 7.154, 0 failed
progress: 12.0 s, 495.3 tps, lat 16.121 ms stddev 6.688, 0 failed
progress: 18.0 s, 499.3 tps, lat 16.023 ms stddev 5.881, 0 failed
progress: 24.0 s, 499.5 tps, lat 16.001 ms stddev 6.771, 0 failed
progress: 30.0 s, 499.7 tps, lat 16.007 ms stddev 5.446, 0 failed
progress: 36.0 s, 499.0 tps, lat 16.017 ms stddev 5.543, 0 failed
progress: 42.0 s, 498.3 tps, lat 16.043 ms stddev 6.198, 0 failed
progress: 48.0 s, 499.7 tps, lat 15.998 ms stddev 6.304, 0 failed
progress: 54.0 s, 499.5 tps, lat 15.999 ms stddev 6.540, 0 failed
progress: 60.0 s, 498.8 tps, lat 16.031 ms stddev 6.810, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29955
number of failed transactions: 0 (0.000%)
latency average = 16.009 ms
latency stddev = 6.358 ms
initial connection time = 26.136 ms
tps = 499.282079 (without initial connection time)
