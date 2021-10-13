# Физический уровень PostgreSQL

## Установка и настройка PostgreSQL

В Google Cloud Platform создана ВМ pg14:
```console
gcloud compute instances create pg14 ...
```
После установки PostgreSQL 14 проверяем сосотояние сервера:
```console
-bash-4.2$ /usr/pgsql-14/bin/pg_ctl status
pg_ctl: server is running (PID: 1625)
/usr/pgsql-14/bin/postgres "-D" "/var/lib/pgsql/14/data/"
```
Создаем БД otus, подключаемся к ней и создаем таблицу test, вставляем запись:
```console
CREATE DATABASE otus;
\c otus 
create table test(c1 text);
insert into test values('1');
```
Останавливаем сервер:
```console
-bash-4.2$ /usr/pgsql-14/bin/pg_ctl stop
```
Создаём новый standard persistent диск в Google Cloud Platform:
```console
gcloud beta compute disks create disk-pgtbl1 --project=core-catalyst-328410 --type=pd-ssd --size=10GB --zone=us-central1-a
```
Добавляем диск к виртуальной машине:
>VM instances -> VM instance details -> EDIT ->  Additional disks -> Attach existing disk (disk-pgtbl1)

Инициализируем диск в ОС:
```console
[root@pg14 ~]# lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   10G  0 disk

[root@pg14 ~]# parted /dev/sdb mklabel gpt
[root@pg14 ~]# parted -a optimal /dev/sdb mkpart primary ext4 0% 100%
[root@pg14 ~]# mkfs.ext4 /dev/sdb1
[root@pg14 ~]# mkdir -p /mnt/data
[root@pg14 ~]# echo "`blkid | grep '/dev/sdb1' | awk '{print $2}'` /mnt/data ext4 defaults 0 2" >> /etc/fstab
[root@pg14 ~]# mount -av
```
Делаем пользователя postgres владельцем /mnt/data:
```console
[root@pg14 ~]# chown -R postgres.postgres /mnt/data/
```
Изменяем переменную окружения PGDATA в файле профиля пользователя postgres:
```console
-bash-4.2$ cat .bash_profile 
...
#PGDATA=/var/lib/pgsql/14/data
PGDATA=/mnt/data/14/data
...
```
Запускаем сервер, подключаемся через psql и проверяем содержимое ранее созданной таблицы:
```console
-bash-4.2$ /usr/pgsql-14/bin/pg_ctl start
waiting for server to start....2021-10-13 07:56:36.622 UTC [16161] LOG:  redirecting log output to logging collector process
2021-10-13 07:56:36.622 UTC [16161] HINT:  Future log output will appear in directory "log".
 done
server started

-bash-4.2$ psql 
psql (14.0)
Type "help" for help.

postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# select * from test;
 c1 
----
 1
(1 row)
```
Для корректного запуска сервиса postgresql-14.service необходимо изменить переменную PGDATA в конфигурационном файле сервиса:
```console
[root@pg14 ~]# systemctl edit postgresql-14.service
[Service]
Environment=PGDATA=/mnt/data/14/data/

[root@pg14 ~]# systemctl cat postgresql-14.service
...

# /etc/systemd/system/postgresql-14.service.d/override.conf
[Service]
Environment=PGDATA=/mnt/data/14/data/
```

### Задание *

Выключаем ВМ pg14, создаём новую pg14-1, устанавливаем PostgreSQL 14
```console
gcloud compute instances create pg14-1 ...
```
Перемонтируйте внешний диск disk-pgtbl1 от pg14 к pg14-1.

для pg14:
>VM instances -> VM instance details -> EDIT ->  Additional disks -> delete existing disk (disk-pgtbl1)

для pg14-1:
>VM instances -> VM instance details -> EDIT ->  Additional disks -> Attach existing disk (disk-pgtbl1)

Сохраняем файл профиля пользователя postgres и удаляем данные из /var/lib/pgsql
```console
[root@pg14-1 ~]# cp /var/lib/pgsql/.bash_profile /tmp
[root@pg14-1 ~]# rm -rf /var/lib/pgsql/*
```
Инициализируем подключенный диск (disk-pgtbl1):
```console
[root@pg14-1 ~]# echo "`blkid | grep '/dev/sdb1' | awk '{print $2}'` /var/lib/pgsql ext4 defaults 0 2" >> /etc/fstab 
[root@pg14-1 ~]# mount -av
```
Возвращаем файл профиля пользователя postgres:
```console
[root@pg14-1 ~]# mv /tmp/.bash_profile /var/lib/pgsql/
```
Проверяем переменную PGDATA в окружении пользователя postgres и состояние каталога $PGDATA:
```console
-bash-4.2$ env | grep PGDATA
PGDATA=/var/lib/pgsql/14/data

-bash-4.2$ ls -l $PGDATA
total 136
drwx------. 6 postgres postgres  4096 Oct 13 05:36 base
-rw-------. 1 postgres postgres    30 Oct 13 10:26 current_logfiles
drwx------. 2 postgres postgres  4096 Oct 13 10:27 global
drwx------. 2 postgres postgres  4096 Oct 13 05:33 log
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_commit_ts
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_dynshmem
-rw-------. 1 postgres postgres  4577 Oct 13 05:33 pg_hba.conf
-rw-------. 1 postgres postgres  1636 Oct 13 05:33 pg_ident.conf
drwx------. 4 postgres postgres  4096 Oct 13 10:31 pg_logical
drwx------. 4 postgres postgres  4096 Oct 13 05:33 pg_multixact
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_notify
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_replslot
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_serial
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_snapshots
drwx------. 2 postgres postgres  4096 Oct 13 10:26 pg_stat
drwx------. 2 postgres postgres  4096 Oct 13 10:49 pg_stat_tmp
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_subtrans
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_tblspc
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_twophase
-rw-------. 1 postgres postgres     3 Oct 13 05:33 PG_VERSION
drwx------. 3 postgres postgres  4096 Oct 13 05:33 pg_wal
drwx------. 2 postgres postgres  4096 Oct 13 05:33 pg_xact
-rw-------. 1 postgres postgres    88 Oct 13 05:33 postgresql.auto.conf
-rw-------. 1 postgres postgres 28750 Oct 13 05:33 postgresql.conf
-rw-------. 1 postgres postgres    27 Oct 13 10:26 postmaster.opts
-rw-------. 1 postgres postgres   103 Oct 13 10:26 postmaster.pid
```
Запускаем сервер и проверям ранее созданную таблицу в базе otus:
```console
-bash-4.2$ /usr/pgsql-14/bin/pg_ctl start

-bash-4.2$ psql 
psql (14.0)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# select * from test ;
 c1 
----
 1
(1 row)
```
