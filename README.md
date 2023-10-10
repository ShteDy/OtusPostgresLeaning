

--создали ВМ в VKCloud. ОС Ubuntu 22. подключение по связке приватного и публичного ключа.

1.подключимся по SSH
ssh -a ubuntu@146.185.210.143


2. Установим 15ю версию Postgres из репозитория
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

3. проверим статус установки
ubuntu@otuscoursepostgre:/var/lib$  sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
ubuntu@otuscoursepostgre:/var/lib$ 

4. Создадим табличку с данными 
ubuntu@otuscoursepostgre:/var/lib$ sudo su postgres
postgres@otuscoursepostgre:/var/lib$ cd ~
postgres@otuscoursepostgre:~$ pwd
/var/lib/postgresql
postgres@otuscoursepostgre:~$ psql
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
postgres@otuscoursepostgre:~$ 

5. остановим кластер
postgres@otuscoursepostgre:~$ pg_ctlcluster 15 main status
pg_ctl: server is running (PID: 127974)
/usr/lib/postgresql/15/bin/postgres "-D" "/var/lib/postgresql/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
postgres@otuscoursepostgre:~$ pg_ctlcluster 15 main stop
postgres@otuscoursepostgre:~$ pg_ctlcluster 15 main status
pg_ctl: no server running
postgres@otuscoursepostgre:~$ 

6. Подключим дополнительный диск
ubuntu@otuscoursepostgre:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        29G  5.1G   22G  19% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
ubuntu@otuscoursepostgre:~$ 

7. перезагружаем виртуалку
делаем юзера postgres владельцем /mnt/data/
ubuntu@otuscoursepostgre:~$ sudo chown -R postgres:postgres /mnt/data/
ubuntu@otuscoursepostgre:~$ 

смувим данные из коневой папки постгре на новый диск:
ubuntu@otuscoursepostgre:~$ sudo mv /var/lib/postgresql/15 /mnt/data
ubuntu@otuscoursepostgre:~$ 

запускаем кластер и получаем ошибку, что постгре не может найти свои базы
ubuntu@otuscoursepostgre:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
ubuntu@otuscoursepostgre:~$ 

в файлике postgresql.conf пропишем новый путь к файлам:
ubuntu@otuscoursepostgre:~$ sudo nano /etc/postgresql/15/main/postgresql.conf 

#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------

# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

data_directory = '/mnt/data/15/main'            # use data in another directory
                                        # (change requires restart)

8. Запустим сервер Postgres, подключимся к базе, проверем где лежит каталог с данными, и посмотрим наличие созданной таблицы:

ubuntu@otuscoursepostgre:~$ sudo systemctl status start postgresql 
Unit start.service could not be found.
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Mon 2023-10-09 14:18:08 UTC; 16min ago
    Process: 785 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 785 (code=exited, status=0/SUCCESS)
        CPU: 3ms

Oct 09 14:18:08 otuscoursepostgre systemd[1]: Starting PostgreSQL RDBMS...
Oct 09 14:18:08 otuscoursepostgre systemd[1]: Finished PostgreSQL RDBMS.
ubuntu@otuscoursepostgre:~$ psql
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "ubuntu" does not exist
ubuntu@otuscoursepostgre:~$ sudo -u postgres psql
could not change directory to "/home/ubuntu": Permission denied
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

postgres=# SHOW data_directory;
  data_directory   
-------------------
 /mnt/data/15/main
(1 row)

postgres=# \l
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)

postgres=# select * from table test;
ERROR:  syntax error at or near "table"
LINE 1: select * from table test;
                      ^
postgres=# select * from test;
 c1 
----
 1
(1 row)

postgres=# 


9. создадим новую виртуальную машину и подключимся к ней ssh -a ubuntu@89.208.86.220

поставим Postgre:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

подключим дополнительный диск отключенный от изначально машины
ubuntu@otuscoursepostgre2:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0    62M  1 loop /snap/core20/1587
loop1    7:1    0  79.9M  1 loop /snap/lxd/22923
loop2    7:2    0    47M  1 loop /snap/snapd/16292
loop3    7:3    0 111.9M  1 loop /snap/lxd/24322
vda    252:0    0    30G  0 disk 
└─vda1 252:1    0    30G  0 part /
vdb    252:16   0    10G  0 disk 
└─vdb1 252:17   0    10G  0 part 

ubuntu@otuscoursepostgre2:~$  sudo mount /dev/vdb1 /mnt/
ubuntu@otuscoursepostgre2:~$ sudo ls /mnt/
15  lost+found

удаляем каталог /var/lib/postgres
ubuntu@otuscoursepostgre2:~$ sudo systemctl stop postgresql@15-main
ubuntu@otuscoursepostgre2:~$ sudo rm -Rfv /var/lib/postgresql
removed directory '/var/lib/postgresql/15/main/pg_replslot'
removed '/var/lib/postgresql/15/main/postgresql.auto.conf'
removed '/var/lib/postgresql/15/main/pg_stat/pgstat.stat'
removed directory '/var/lib/postgresql/15/main/pg_stat'
removed '/var/lib/postgresql/15/main/pg_multixact/offsets/0000'
removed directory '/var/lib/postgresql/15/main/pg_multixact/offsets'
removed '/var/lib/postgresql/15/main/pg_multixact/members/0000'
removed directory '/var/lib/postgresql/15/main/pg_multixact/members'
removed directory '/var/lib/postgresql/15/main/pg_multixact'
removed directory '/var/lib/postgresql/15/main/pg_tblspc'
removed '/var/lib/postgresql/15/main/postmaster.opts'
removed directory '/var/lib/postgresql/15/main/pg_notify'

ubuntu@otuscoursepostgre2:~$ ls /var/lib/postgresql
ls: cannot access '/var/lib/postgresql': No such file or directory
ubuntu@otuscoursepostgre2:~$ ls /var/lib/
PackageKit  boltd              dbus  dpkg   grub       man-db  plymouth  python        snmp     tpm                      ucf                  update-manager   usbutils
apport      cloud              dhcp  fwupd  landscape  misc    polkit-1  shells.state  sudo     ubuntu-advantage         udisks2              update-notifier  vim
apt         command-not-found  dkms  git    logrotate  pam     private   snapd         systemd  ubuntu-release-upgrader  unattended-upgrades  usb_modeswitch
ubuntu@otuscoursepostgre2:~$ 


стартуем инстанс Postgres, он ожидаемо не стартует:
ubuntu@otuscoursepostgre2:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist

дадим права юзеру postgres на каталог на новом диске
ubuntu@otuscoursepostgre2:~$ sudo chown -R postgres:postgres /mnt/15

исправим файл postgresql.conf 
ubuntu@otuscoursepostgre2:~$ sudo nano /etc/postgresql/15/main/postgresql.conf 
# Memory units:  B  = bytes            Time units:  us  = microseconds
#                kB = kilobytes                     ms  = milliseconds
#                MB = megabytes                     s   = seconds
#                GB = gigabytes                     min = minutes
#                TB = terabytes                     h   = hours
#                                                   d   = days


#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------

# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

data_directory = '/mnt/15/main'		# use data in another directory
					# (change requires restart)


Запустим инстанс postgres
ubuntu@otuscoursepostgre2:~$ sudo systemctl start postgresql@15-main
ubuntu@otuscoursepostgre2:~$ 

подключимся к базе и посмотрим запись в табличке test:
ubuntu@otuscoursepostgre2:~$ sudo -u postgres psql
could not change directory to "/home/ubuntu": Permission denied
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

postgres=# select * from table test;
ERROR:  syntax error at or near "table"
LINE 1: select * from table test;
                      ^
postgres=# select * from  test;
 c1 
----
 1
(1 row)

postgres=


##инстанс постгреса смог подключится к базе на подключенном диске с другого сервера.