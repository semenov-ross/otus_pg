# Работа с большим объемом реальных данных
### Разворачиваем и настраиваем БД с большими данными

Для работы используем публичные данные из BigQuery, экпорировав таблицу bigquery-public-data:chicago_taxi_trips.taxi_trips в созданный bucket taxi_trips_20211128:
```console
[ross@otuspg ~]$ gsutil mb gs://taxi_trips_20211128
Creating gs://taxi_trips_20211128/...

[ross@otuspg ~]$ bq extract 'bigquery-public-data:chicago_taxi_trips.taxi_trips' gs://taxi_trips_20211128/taxi_trips_*.csv

Waiting on bqjob_r6a64f0ef2df8c702_0000017d656a23c6_1 ... (68s) Current status: DONE

ross@otuspg ~]$ gsutil ls gs://taxi_trips_20211128/
gs://taxi_trips_20211128/taxi_trips_000000000000.csv
gs://taxi_trips_20211128/taxi_trips_000000000001.csv
gs://taxi_trips_20211128/taxi_trips_000000000002.csv
...
gs://taxi_trips_20211128/taxi_trips_000000000295.csv
gs://taxi_trips_20211128/taxi_trips_000000000296.csv
```

В Google Cloud Platform создана ВМ g14-bigdata с увеличенным объёмом диска для загрузки большого количества данных:
```console
gcloud compute instances create pg14-bigdata ... size=50 ...
```
