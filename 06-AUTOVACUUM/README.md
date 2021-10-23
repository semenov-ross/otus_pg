# MVCC, vacuum и autovacuum
### объяснить работу механизма многоверсионности в PostgreSQL;знать и уметь использовать vacuum и autovacuum;

Cоздаём GCE инстанс типа e2-medium и standard disk, установливаем на него PostgreSQL 14 с дефолтными настройками
применяеем параметры настройки из прикрепленного к материалам занятия файла
```console
gcloud compute instances create pg14  ... diskTypes/pd-standard ...
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum -y install postgresql14 postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service 
[root@pg14 ~]# su - postgres
-bash-4.2$ cd /var/lib/pgsql/14/data
-bash-4.2$ mv postgresql.conf postgresql.conf.base
-bash-4.2$ vim postgresql.conf
-bash-4.2$ cat postgresql.conf 
include 'postgresql.base.conf'

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

[root@pg14 ~]# systemctl restart postgresql-14.service
```
![network](https://github.com/semenov-ross/otus_pg/blob/master/06-AUTOVACUUM/pgbench.png)
