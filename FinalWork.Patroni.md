
## Настроим отказоустойчивый кластер на базе демона Patroni

## ETCDhttps://github.com/ShteDy/OtusPostgresLeaning/blob/main/FinalWork.Patroni.md
первым этапом инсталляции будет настройка распеределенного хранилища etcd
оно позволит контролировать доступность сервиса Patroni,в случае выхода из строя одного из узлов
подготовим три виртуальные машины,  моем случае это ubuntu версии 20.04.1. База etcd это согласованное хранилище ключей. Её задача оценивать доступность сервиса (в нашем случае Patroni), и хранить настройки конфигурации. Количество узлов должно быть нечетным, т.к. каждый из узлов считается голосом, для создания кворума для работы сервиса.
Поскольку etcd надежная и не особо требовательная ресурсам, подготовим с параметрами 1 CPU + 2 RAM. Однко, посколько узлы etcd постоянно обмениваются healthcheck'ами, то рекомендуется использовать быстрые диски.

### Установка и настройка ETCD
Итак имеем 3 виртуальные машины с адресами:
#etcd1 10.0.0.29
#etcd2 10.0.0.31
#etcd3 10.0.0.20

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
роздадим 2 виртуальные машины, на которые поставим кластер PostgreSQL 16, а потом демон Patroni
```bash
ubuntu@patroni1:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
ubuntu@patroni1:~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
OK
ubuntu@patroni1:~$ sudo apt update
ubuntu@patroni1:~$ sudo apt install postgresql-16 -y

после установки останавливаем сервис Postgres и очищаем каталог с базой. далее данные отдаеются под управление демона Patroni
ubuntu@etcd1-restored:~$ sudo systemctl stop postgresql
ubuntu@etcd1-restored:~$ sudo rm -rf /var/lib/postgresql/16/main/*
```
## Установка Python


