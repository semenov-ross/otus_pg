# Виды и устройство репликации в PostgreSQL.
### настроить репликацию;

В Google Cloud Platform создаём 4 ВМ и устанавливаем PostgreSQL 14 :
```console
[ross@otuspg ~]$ gcloud compute instances list | awk '{print $1}'
NAME
pg14-repl1
pg14-repl2
pg14-repl3
pg14-repl4

[ross@otuspg ~]$ gcloud compute ssh pg14-repl1
[ross@pg14-repl1 ~]$ sudo su -
[root@pg14-repl1 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl1 ~]# yum -y install postgresql14-server
[root@pg14-repl1 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl1 ~]# systemctl enable --now postgresql-14.service
...
[ross@otuspg ~]$ gcloud compute ssh pg14-repl4
[ross@pg14-repl4 ~]$ sudo su -
[root@pg14-repl4 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl4 ~]# yum -y install postgresql14-server
[root@pg14-repl4 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl4 ~]# systemctl enable --now postgresql-14.service
```

```console
```