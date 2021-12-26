# Секционирование
### Научиться секционировать таблицы. Секционировать большую таблицу из демо базы flights

В Google Cloud Platform создана ВМ pg14-partition, установлен PostgreSQL 14 и импортирована демонстрационная БД от компании Postgres Professional:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14-partition ....
[ross@otuspg ~]$ gcloud compute ssh pg14-partition
[root@pg14-partition ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-partition ~]# yum -y install postgresql14 postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service
[root@pg14-partition ~]# su - postgres
-bash-4.2$ curl https://edu.postgrespro.ru/demo-big.zip --output demo-big.zip
-bash-4.2$ unzip demo-big.zip
-bash-4.2$ psql < demo-big-20170815.sql
```

```console

```

```console
```