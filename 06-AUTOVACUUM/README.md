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
Подключаемся под пользователем postgres, выполняем pgbench -i postgres и запускаем pgbench -c8 -P 10 -T 600 -U postgres postgres
```console
[root@pg14 ~]# su - postgres 
Last login: Sat Oct 23 10:26:57 UTC 2021 on pts/0
-bash-4.2$ /usr/pgsql-14/bin/pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.43 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.26 s, vacuum 0.08 s, primary keys 0.08 s).

-bash-4.2$ /usr/pgsql-14/bin/pgbench -c8 -P 10 -T 600 -U postgres postgres                                                                                                                   
pgbench (14.0)
starting vacuum...end.
progress: 10.0 s, 712.3 tps, lat 11.184 ms stddev 6.477
progress: 20.0 s, 735.5 tps, lat 10.870 ms stddev 5.805
progress: 30.0 s, 737.0 tps, lat 10.843 ms stddev 5.984
progress: 40.0 s, 731.1 tps, lat 10.934 ms stddev 6.125
progress: 50.0 s, 735.0 tps, lat 10.877 ms stddev 6.084
progress: 60.0 s, 736.7 tps, lat 10.850 ms stddev 5.893
progress: 70.0 s, 765.8 tps, lat 10.437 ms stddev 5.387
progress: 80.0 s, 743.8 tps, lat 10.746 ms stddev 5.749
progress: 90.0 s, 734.3 tps, lat 10.887 ms stddev 5.938
progress: 100.0 s, 733.9 tps, lat 10.890 ms stddev 6.026
progress: 110.0 s, 736.1 tps, lat 10.861 ms stddev 6.012
progress: 120.0 s, 749.1 tps, lat 10.668 ms stddev 5.697
progress: 130.0 s, 756.2 tps, lat 10.572 ms stddev 6.100
progress: 140.0 s, 747.8 tps, lat 10.686 ms stddev 5.837
progress: 150.0 s, 753.1 tps, lat 10.619 ms stddev 5.500
progress: 160.0 s, 747.0 tps, lat 10.699 ms stddev 5.719
progress: 170.0 s, 734.1 tps, lat 10.888 ms stddev 5.990
progress: 180.0 s, 737.4 tps, lat 10.841 ms stddev 5.932
progress: 190.0 s, 735.0 tps, lat 10.874 ms stddev 5.963
progress: 200.0 s, 726.0 tps, lat 11.010 ms stddev 5.882
progress: 210.0 s, 747.5 tps, lat 10.694 ms stddev 5.972
progress: 220.0 s, 682.2 tps, lat 11.718 ms stddev 7.286
progress: 230.0 s, 643.0 tps, lat 12.430 ms stddev 7.440
progress: 240.0 s, 719.6 tps, lat 11.110 ms stddev 6.366
progress: 250.0 s, 723.2 tps, lat 11.049 ms stddev 6.173
progress: 260.0 s, 739.0 tps, lat 10.818 ms stddev 5.843
progress: 270.0 s, 763.2 tps, lat 10.476 ms stddev 5.802
progress: 280.0 s, 731.7 tps, lat 10.923 ms stddev 5.790
progress: 290.0 s, 743.5 tps, lat 10.752 ms stddev 5.828
progress: 300.0 s, 748.5 tps, lat 10.679 ms stddev 5.708
progress: 310.0 s, 718.4 tps, lat 11.126 ms stddev 6.065
progress: 320.0 s, 753.3 tps, lat 10.611 ms stddev 5.443
progress: 330.0 s, 761.0 tps, lat 10.502 ms stddev 5.483
progress: 340.0 s, 732.4 tps, lat 10.915 ms stddev 5.932
progress: 350.0 s, 741.0 tps, lat 10.788 ms stddev 5.622
progress: 360.0 s, 726.4 tps, lat 11.004 ms stddev 6.014
progress: 370.0 s, 747.1 tps, lat 10.701 ms stddev 5.998
progress: 380.0 s, 752.6 tps, lat 10.619 ms stddev 5.704
progress: 390.0 s, 756.1 tps, lat 10.572 ms stddev 5.860
progress: 400.0 s, 737.9 tps, lat 10.833 ms stddev 5.575
progress: 410.0 s, 742.1 tps, lat 10.770 ms stddev 5.938
progress: 420.0 s, 749.5 tps, lat 10.667 ms stddev 5.845
progress: 430.0 s, 747.6 tps, lat 10.691 ms stddev 5.804
progress: 440.0 s, 740.2 tps, lat 10.801 ms stddev 5.947
progress: 450.0 s, 522.9 tps, lat 15.274 ms stddev 21.163
progress: 460.0 s, 491.9 tps, lat 16.247 ms stddev 22.328
progress: 470.0 s, 495.4 tps, lat 16.147 ms stddev 22.014
progress: 480.0 s, 509.3 tps, lat 15.696 ms stddev 21.202
progress: 490.0 s, 496.5 tps, lat 16.094 ms stddev 23.232
progress: 500.0 s, 487.6 tps, lat 16.409 ms stddev 23.097
progress: 510.0 s, 510.3 tps, lat 15.668 ms stddev 21.722
progress: 520.0 s, 496.8 tps, lat 16.064 ms stddev 21.417
progress: 530.0 s, 498.1 tps, lat 16.043 ms stddev 21.788
progress: 540.0 s, 479.8 tps, lat 16.663 ms stddev 21.660
progress: 550.0 s, 470.7 tps, lat 16.978 ms stddev 22.617
progress: 560.0 s, 489.0 tps, lat 16.334 ms stddev 21.282
progress: 570.0 s, 513.2 tps, lat 15.578 ms stddev 21.703
progress: 580.0 s, 493.8 tps, lat 16.194 ms stddev 21.691
progress: 590.0 s, 505.1 tps, lat 15.831 ms stddev 21.629
progress: 600.0 s, 499.8 tps, lat 15.981 ms stddev 21.496
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 403952
latency average = 11.872 ms
latency stddev = 11.276 ms
initial connection time = 26.093 ms
tps = 673.257918 (without initial connection time)
```
![network](https://github.com/semenov-ross/otus_pg/blob/master/06-AUTOVACUUM/pgbench_n.png)
