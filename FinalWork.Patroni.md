
## Настроим отказоустойчивый кластер на базе демона Patroni

## ETCD
первым этапом инсталляции будет настройка распеределенного хранилища etcd
оно позволит контролировать доступность сервиса Patroni,в случае выхода из строя одного из узлов
подготовим три виртуальные машины,  моем случае это ubuntu версии 20.04.1. База etcd это согласованное хранилище ключей. Её задача оценивать доступность сервиса (в нашем случае Patroni), и хранить настройки конфигурации. Количество узлов должно быть нечетным, т.к. каждый из узлов считается голосом, для создания кворума для работы сервиса.
Поскольку etcd надежная и не особо требовательная ресурсам, подготовим с параметрами 1 CPU + 2 RAM. Однко, посколько узлы etcd постоянно обмениваются healthcheck'ами, то рекомендуется использовать быстрые диски.

### Установка и настройка ETCD
Итак имеем 3 виртуальные машины с адресами:
etcd1 10.0.0.29
etcd2 10.0.0.31
etcd3 10.0.0.20

```bash
установка etcd простая

ubuntu@etcd1-restored:~$ sudo -i
root@etcd1-restored:~# apt update
root@etcd1-restored:~# apt install etcd -y

настройка представляет собой создание файла конфигурации

nano /etc/default/etcd 

по ссылке описание переменных - https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample в этом файле
в моем файле они такие:
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_HEARTBEAT_INTERVAL="1000"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_LISTEN_PEER_URLS="http://10.0.0.29:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.29:2379,http://localhost:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.29:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.0.0.29:2380,etcd2=http://10.0.0.31:2380,etcd3=http://10.0.0.20:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="postgrescluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.29:2379"
ETCD_ENABLE_V2="true"

на остальных серверах кластера меняются значения ETCD_NAME, ETCD_LISTEN_PEER_URLS, ETCD_LISTEN_CLIENT_URLS, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS
ETCD_INITIAL_CLUSTER_TOKEN на всех сервера кластера должен быть одинаковый

после запуска etcd каждая из нод будет чувствовать себя отдельным кластером
root@etcd1-restored:/var/lib/etcd/default# systemctl start etcd
root@etcd1-restored:/var/lib/etcd/default# etcdctl cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://localhost:2379
cluster is healthy

бля того чтоб их собрать воедино, нужно почистить каталог /var/lib/etcd/default/member/*
root@etcd1-restored:/var/lib/etcd/default# rm -rf /var/lib/etcd/default/member/*
а после перезапустить etcd
root@etcd1-restored:/var/lib/etcd/default# systemctl restart etcd
root@etcd1-restored:/var/lib/etcd/default# systemctl status etcd
● etcd.service - etcd - highly-available key value store
     Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2023-12-09 13:28:44 UTC; 3s ago
       Docs: https://github.com/coreos/etcd
             man:etcd
   Main PID: 3625 (etcd)
      Tasks: 9 (limit: 2282)
     Memory: 3.8M
     CGroup: /system.slice/etcd.service
             └─3625 /usr/bin/etcd

проверить количество узлов можно узлов кластера можно при помощи команды etcdctl, в нашем случае их должно быть 3
root@etcd1:~# etcdctl cluster-health
member 24c242139c616766 is healthy: got healthy result from http://10.0.0.20:2379
member 2ed06313262270a9 is healthy: got healthy result from http://10.0.0.31:2379
member 5f245f2264a3f06c is healthy: got healthy result from http://10.0.0.29:2379
cluster is healthy

```
## Установка Postgres
cоздадим 2 виртуальные машины, на которые поставим кластер PostgreSQL 16, а потом демон Patroni
```bash
ubuntu@patroni1:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
ubuntu@patroni1:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
OK
ubuntu@patroni1:~$ sudo apt update
ubuntu@patroni1:~$ sudo apt install postgresql-16 -y

после установки останавливаем сервис Postgres и очищаем каталог с базой. далее данные отдаеются под управление демона Patroni
ubuntu@patroni1:~$ sudo systemctl stop postgresql
ubuntu@patroni1:~$ sudo rm -rf /var/lib/postgresql/16/main/*
```
## Установка Python
Поскольку демон Patroni написан на питоне, необходимо утсновить питон и некоторые библиотеки
```bash
ubuntu@patroni1:~$ sudo apt install python3-pip -y
ubuntu@patroni1:~$ sudo apt install python3-dev -y
ubuntu@patroni1:~$ sudo apt install libpq-dev -y
ubuntu@patroni1:~$ sudo pip install psycopg2
```
## Установка Patroni
```bash
ubuntu@patroni1:~$ sudo pip install patroni[etcd]
ubuntu@patroni1:~$ sudo pip3 install patroni
```
создадим в ОС сервис patroni.service  пропишем его конфиг
```bash
ubuntu@etcd1-restored:~$ sudo systemctl edit --full --force patroni.service

записываем конфиг
# This is an example systemd config file for Patroni
# You can copy it to "/etc/systemd/system/patroni.service",
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf
WorkingDirectory=~
# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
#StandardOutput=syslog
# Pre-commands to start watchdog device
# Uncomment if watchdog is part of your patroni setup
#ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
#ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog
# Start the patroni process
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process
# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30
# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no
[Install]
WantedBy=multi-user.target

перезагружаем сервисы
ubuntu@patroni1:~$ sudo systemctl daemon-reload
Добавляем Патрони в запуск при старте ОС
ubuntu@patroni1:~$ sudo systemctl enable patroni
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.

Создаем конфигурационный файл patroni.yml
Sudo nano /etc/patroni.yml

scope: pg-ha-cluster
name: pp_pg_1

log:
  level: WARNING
  format: '%(asctime)s %(levelname)s: %(message)s'
  dateformat: ''
  max_queue_size: 1000
  dir: /var/log/postgresql
  file_num: 4
  file_size: 25000000
  loggers:
    postgres.postmaster: WARNING
    urllib3: DEBUG

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.25:8008

etcd:
  hosts: 
  - 10.0.0.29:2379
  - 10.0.0.31:2379
  - 10.0.0.20:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 0
    synchronous_mode: true
    synchronous_mode_strict: false
    postgresql:
#      recovery_conf:
#        restore_command: /usr/local/bin/restore_wal.sh %p %f
#        recovery_target_time: '2021-06-11 13:20:00'
#        recovery_target_action: promote
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 200
        shared_buffers: 2GB
        effective_cache_size: 6GB
        maintenance_work_mem: 512MB
        checkpoint_completion_target: 0.7
        wal_buffers: 16MB
        default_statistics_target: 100
        random_page_cost: 1.1
        effective_io_concurrency: 200
        work_mem: 2621kB
        min_wal_size: 1GB
        max_wal_size: 4GB
        max_worker_processes: 40
        max_parallel_workers_per_gather: 4
        max_parallel_workers: 40
        max_parallel_maintenance_workers: 4

        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        wal_level: replica
        wal_log_hints: on
        track_commit_timestamp: off
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_segments: 8
        logging_collector: on
        log_destination: csvlog
        log_directory: pg_log
        log_min_messages: warning
        log_min_error_statement: error
        log_min_duration_statement: 1000
        log_duration: off
        log_statement: all

  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host all postgres all md5
  - host replication repl all md5

  users:
    postgres:
      password: mypassword
      options:
        - createrole
        - createdb
    repl:
      password: mypassword
      options:
        - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.25:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  config_dir: /var/lib/postgresql/16/main
  pgpass: /var/lib/postgresql/.pgpass
  pg_hba:
    - local all all trust
    - host all postgres all md5
    - host replication repl all md5
  authentication:
    replication:
      username: repl
      password: mypassword
    superuser:
      username: postgres
      password: mypassword
  parameters:
#    archive_mode: on
#   archive_command: /usr/local/bin/copy_wal.sh %p %f
#    archive_timeout: 600
#    unix_socket_directories: '/var/run/postgresql'
#    port: 5432![image](https://github.com/ShteDy/OtusPostgresLeaning/assets/124609480/1ee22627-5e35-49c6-a00b-e4f89515aa59)



нюансы данного конфига:
1. это yml, потому критичен к синтаксису
2. этот конфиг прописывается на всех узлах, где установлен Patroni
3. переменная name уникальна для каждого узла Patroni, Scope - одинаковая
4. фактически этот файл заменяет конфиги postgressql.conf  и pg_hba.conf (смотри параметры сервера, учетных записей, доступа, логирования и т.д.)
более подробное описание тут
https://patroni.readthedocs.io/en/latest/patroni_configuration.html

после этого стартуем сервис Patroni
ubuntu@patroni1:~$ sudo service patroni start
проверяем его статус
ubuntu@patroni1:~$ service patroni status
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
     Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-12-11 12:32:04 UTC; 4s ago
   Main PID: 147816 (patroni)
      Tasks: 14 (limit: 9451)
     Memory: 104.7M
     CGroup: /system.slice/patroni.service
             ├─147816 /usr/bin/python3 /usr/local/bin/patroni /etc/patroni.yml
             ├─147848 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main --config-file=/var/lib/postgresql/16/main/postgresql.conf --listen_addresses>
             ├─147850 postgres: pg_ha_cluster: logger
             ├─147851 postgres: pg_ha_cluster: checkpointer
             ├─147852 postgres: pg_ha_cluster: background writer
             ├─147853 postgres: pg_ha_cluster: startup recovering 000000080000000000000003
             ├─147854 postgres: pg_ha_cluster: walreceiver
             ├─147859 postgres: pg_ha_cluster: postgres postgres 127.0.0.1(36732) idle
             └─147862 postgres: pg_ha_cluster: postgres postgres 127.0.0.1(36734) idle

Dec 11 12:32:04 patroni1 systemd[1]: Started Runners to orchestrate a high-availability PostgreSQL.
Dec 11 12:32:05 patroni1 patroni[147849]: localhost:5432 - no response
Dec 11 12:32:05 patroni1 patroni[147848]: 2023-12-11 12:32:05.827 UTC [147848] LOG:  redirecting log output to logging collector process
Dec 11 12:32:05 patroni1 patroni[147848]: 2023-12-11 12:32:05.827 UTC [147848] HINT:  Future log output will appear in directory "pg_log".
Dec 11 12:32:06 patroni1 patroni[147855]: localhost:5432 - accepting connections
Dec 11 12:32:06 patroni1 patroni[147857]: localhost:5432 - accepting connections

также проверям каталог /var/lib/postgresql/16/main/
ubuntu@patroni1d:~$ sudo ls /var/lib/postgresql/16/main/
PG_VERSION  global		  pg_commit_ts	pg_logical    pg_notify    pg_serial	 pg_stat      pg_subtrans  pg_twophase	pg_xact		      postmaster.opts
base	    patroni.dynamic.json  pg_dynshmem	pg_multixact  pg_replslot  pg_snapshots  pg_stat_tmp  pg_tblspc    pg_wal	postgresql.auto.conf  standby.signal
ubuntu@patroni1:~$ 

видим, что Patroni создал файлы базы данных. Если необходимо развернуть имеющуюся базу из резервной копии - можно развернуть также как обычно.
Сервис Patroni заработал. Осталось предоставить к нему доступ
```
## HAProxy
```bash
для организации доступа к кластеру Patroni установим HAProxy     

ubuntu@patronidemo-haproxy1:~$sudo apt install haproxy -y
правим конфиг
ubuntu@patronidemo-haproxy1:~$sudo nano /etc/haproxy/haproxy.cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patronidemo1 10.0.0.25:5432 maxconn 100 check port 8008
    server patronidemo2 10.0.0.14:5432 maxconn 100 check port 8008


в параметрах listen postgres прописываем сервера patroni, и порт 5000 по еотрому будем подключаться к кластеру через HAProxy
сам сервис HAProxy будет доступен по порту 7000, если необходимо ssl совдинения в конфиге указывается еще путь до сертификатов.

перзагружаем HAProxy
ubuntu@patronidemo-haproxy1:~$ sudo service haproxy restart

после этого доступ к высокодоступному кластеру Patroni на базе СУБД PostgreSQL осуществляется через через HAProxy 10.0.0.47:5000


```






```




