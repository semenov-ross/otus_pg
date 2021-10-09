# SQL и реляционные СУБД. Введение в PostgreSQL 

В Google Cloud Platform создана ВМ pg14:
```console
gcloud compute instances create pg14 ...
```
Подключаемся к созданной pg14:
```console
gcloud compute ssh pg14
```
Добавляем репозитроий PostgreSQL:
```console
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Установливаем PostgreSQL 14:
```console
yum -y install postgresql14 postgresql14-server
```
Инициализируем базу данных:
```console
/usr/pgsql-14/bin/postgresql-14-setup initdb
```
Запускаем сервис postgresql-14.service:
```console
systemctl start postgresql-14.service
systemctl enable postgresql-14.service
```
