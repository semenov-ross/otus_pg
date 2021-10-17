# Установка PostgreSQL 

### Установка и настройка PostgteSQL в контейнере Docker

Создаём в в GCE инстанс:

```console
gcloud compute instances create otus-pg-docker ...
```
Устанавливаем Docker Engine:
```console
yum install docker.x86_64 -y
systemctl enable docker.service
systemctl start docker.service

[root@otus-pg-docker ~]# systemctl status docker.service 
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-10-17 13:07:25 UTC; 16min ago
     Docs: http://docs.docker.com
...
```
Создаём каталог /var/lib/postgres:
```console
[root@otus-pg-docker ~]# mkdir /var/lib/postgres
```
Разворачиваем контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres:
```console
[root@otus-pg-docker ~]# docker network create pg-net
10ce3d6ea023c995da884b437f48bb657bf6d70f5a0c18826c0925e5c3179fc7
[root@otus-pg-docker ~]# docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data:z postgres:14
403e1f6163960469b2cfd91f3304e4374cdfd0a57b288774e6994743dc8ed00e
[root@otus-pg-docker ~]# docker ps 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
403e1f616396        postgres:14         "docker-entrypoint..."   5 seconds ago       Up 4 seconds        0.0.0.0:5432->5432/tcp   pg-docker
```
При монтировании каталога /var/lib/postgres в контейнер pg-docker указана опция z, необходимая для изменения метки selinux, указывающая на использования его несколькими контейнерами

Развернем контейнер с клиентом postgres и подключимся клиентом к контейнеру с сервером, создадим тестовую базу, в которой создадим таблицу и добавим в неё данные:
```console
docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres: 
psql (14.0 (Debian 14.0-1.pgdg110+1))
Type "help" for help.

postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# CREATE TABLE test_docker (col1 int,col2 int);
CREATE TABLE
otus=# INSERT INTO test_docker SELECT i, i+2 FROM generate_series (1,10) s(i);
INSERT 0 10
otus=# select * from test_docker ;
 col1 | col2 
------+------
    1 |    3
    2 |    4
    3 |    5
    4 |    6
    5 |    7
    6 |    8
    7 |    9
    8 |   10
    9 |   11
   10 |   12
(10 rows)
```
