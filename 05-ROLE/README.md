# Логический уровень PostgreSQL

### Работа с базами данных, пользователями и правами

Создаём новый кластер PostgresSQL 14
```console
ross@otuspg ~]$ gcloud compute instances create pg14 ...
[ross@otuspg ~]$ gcloud compute ssh pg14
[ross@pg14 ~]$ sudo su -
[root@pg14 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14 ~]# yum -y install postgresql14 postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service
```
