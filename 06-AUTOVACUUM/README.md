# MVCC, vacuum и autovacuum
### по возможности не допускать проблем с autovacuum - отсутствием или наличием, если допустили то знать как их решать;

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
Меняем параметра autovacuum:
```console
postgres=# ALTER SYSTEM SET autovacuum_naptime = 10;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET autovacuum_max_workers = 8;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.1;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET autovacuum_vacuum_threshold = 0;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```
Запускаем тот же тест:
```console
-bash-4.2$ /usr/pgsql-14/bin/pgbench -c8 -P 10 -T 600 -U postgres postgres
pgbench (14.0)
starting vacuum...end.
progress: 10.0 s, 669.7 tps, lat 11.864 ms stddev 6.773
progress: 20.0 s, 654.0 tps, lat 12.221 ms stddev 7.641
progress: 30.0 s, 675.9 tps, lat 11.829 ms stddev 7.449
progress: 40.0 s, 712.9 tps, lat 11.210 ms stddev 5.933
progress: 50.0 s, 710.8 tps, lat 11.249 ms stddev 5.971
progress: 60.0 s, 732.6 tps, lat 10.910 ms stddev 5.612
progress: 70.0 s, 716.6 tps, lat 11.154 ms stddev 6.187
progress: 80.0 s, 690.6 tps, lat 11.576 ms stddev 6.487
progress: 90.0 s, 715.9 tps, lat 11.162 ms stddev 5.863
progress: 100.0 s, 721.2 tps, lat 11.086 ms stddev 5.987
progress: 110.0 s, 734.4 tps, lat 10.883 ms stddev 5.890
progress: 120.0 s, 723.4 tps, lat 11.050 ms stddev 5.854
progress: 130.0 s, 696.3 tps, lat 11.480 ms stddev 6.437
progress: 140.0 s, 734.2 tps, lat 10.889 ms stddev 5.472
progress: 150.0 s, 734.7 tps, lat 10.881 ms stddev 5.744
progress: 160.0 s, 735.8 tps, lat 10.863 ms stddev 5.897
progress: 170.0 s, 731.1 tps, lat 10.933 ms stddev 6.018
progress: 180.0 s, 747.7 tps, lat 10.694 ms stddev 5.822
progress: 190.0 s, 717.7 tps, lat 11.131 ms stddev 5.784
progress: 200.0 s, 729.8 tps, lat 10.958 ms stddev 5.933
progress: 210.0 s, 737.0 tps, lat 10.847 ms stddev 6.032
progress: 220.0 s, 730.3 tps, lat 10.944 ms stddev 5.573
progress: 230.0 s, 733.9 tps, lat 10.893 ms stddev 5.539
progress: 240.0 s, 730.2 tps, lat 10.947 ms stddev 5.862
progress: 250.0 s, 732.8 tps, lat 10.908 ms stddev 6.068
progress: 260.0 s, 722.0 tps, lat 11.069 ms stddev 6.235
progress: 270.0 s, 751.3 tps, lat 10.639 ms stddev 5.719
progress: 280.0 s, 735.6 tps, lat 10.869 ms stddev 5.850
progress: 290.0 s, 733.1 tps, lat 10.904 ms stddev 5.884
progress: 300.0 s, 739.8 tps, lat 10.800 ms stddev 5.680
progress: 310.0 s, 731.5 tps, lat 10.931 ms stddev 5.725
progress: 320.0 s, 708.4 tps, lat 11.283 ms stddev 6.232
progress: 330.0 s, 729.6 tps, lat 10.955 ms stddev 6.386
progress: 340.0 s, 651.8 tps, lat 12.265 ms stddev 7.324
progress: 350.0 s, 690.4 tps, lat 11.582 ms stddev 6.900
progress: 360.0 s, 742.0 tps, lat 10.768 ms stddev 5.756
progress: 370.0 s, 727.7 tps, lat 10.987 ms stddev 5.938
progress: 380.0 s, 714.6 tps, lat 11.188 ms stddev 5.825
progress: 390.0 s, 750.1 tps, lat 10.658 ms stddev 5.514
progress: 400.0 s, 714.0 tps, lat 11.193 ms stddev 6.126
progress: 410.0 s, 742.6 tps, lat 10.763 ms stddev 5.675
progress: 420.0 s, 722.2 tps, lat 11.067 ms stddev 6.036
progress: 430.0 s, 710.2 tps, lat 11.254 ms stddev 6.108
progress: 440.0 s, 696.9 tps, lat 11.475 ms stddev 6.494
progress: 450.0 s, 730.5 tps, lat 10.941 ms stddev 5.943
progress: 460.0 s, 717.7 tps, lat 11.141 ms stddev 5.948
progress: 470.0 s, 741.4 tps, lat 10.780 ms stddev 5.248
progress: 480.0 s, 744.0 tps, lat 10.742 ms stddev 5.723
progress: 490.0 s, 719.5 tps, lat 11.113 ms stddev 5.992
progress: 500.0 s, 588.1 tps, lat 13.590 ms stddev 16.271
progress: 510.0 s, 488.0 tps, lat 16.380 ms stddev 22.400
progress: 520.0 s, 489.4 tps, lat 16.344 ms stddev 22.008
progress: 530.0 s, 498.8 tps, lat 16.029 ms stddev 22.562
progress: 540.0 s, 481.0 tps, lat 16.622 ms stddev 22.127
progress: 550.0 s, 481.3 tps, lat 16.596 ms stddev 23.087
progress: 560.0 s, 476.9 tps, lat 16.750 ms stddev 21.766
progress: 570.0 s, 491.6 tps, lat 16.267 ms stddev 22.574
progress: 580.0 s, 484.3 tps, lat 16.506 ms stddev 22.365
progress: 590.0 s, 498.3 tps, lat 16.028 ms stddev 22.063
progress: 600.0 s, 481.9 tps, lat 16.595 ms stddev 22.628
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 407768
latency average = 11.761 ms
latency stddev = 9.918 ms
initial connection time = 53.945 ms
tps = 679.646534 (without initial connection time)
```
Количество tps увеличелось с 673.257918 до 679.646534
![network](https://github.com/semenov-ross/otus_pg/blob/master/06-AUTOVACUUM/pgbench_n.png)
