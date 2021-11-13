# Настройка PostgreSQL
### поработать с параметрами конфигурации PostgreSQL;объяснить в чем разница между различными группами параметров;объяснить выбор оптимального значения для параметров.

Создаём новый кластер PostgresSQL 14:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14lock ...
[ross@otuspg ~]$ gcloud compute ssh pg14lock
[ross@pg14lock ~]$ sudo su -
[root@pg14lock ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14lock ~]# yum -y install postgresql14 postgresql14-server
[root@pg14lock ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
Initializing database ... OK
[root@pg14lock ~]# systemctl enable --now postgresql-14.service
```
