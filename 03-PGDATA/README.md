# Физический уровень PostgreSQL

## Установка и настройка PostgreSQL

В Google Cloud Platform создана ВМ pg14:
```console
gcloud compute instances create pg14 ...
```
После установки PostgreSQL 14 проверяем сосотояние кластера:
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
Останавливаем кластер:
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
Запускаем кластер, подключаемся через psql и проверяем содержимое ранее созданной таблицы:
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
