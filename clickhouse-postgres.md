Создадим два хоста
```bash
gcloud beta compute --project=quixotic-moment-397713 instances create postgres --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=100GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud beta compute --project=quixotic-moment-397713 instances create clickhouse --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=100GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

Хост postgres:
Установим постгрес
`sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15`

настроим pg_hba:

```bash
nano /etc/postgresql/15/main/pg_hba.conf \
host all all  0.0.0.0/0 scram-sha
```

Настроим конфиг для работы с движком клика:
```bash
nano /etc/postgresql/15/main/postgresql.conf
listen_addresses = ‘*’
wal_level = logical
```

`psql``
зададим пароль для роли postgres
`alter user postgres with password 'aleksei';`
создадим бд test
```bash
create database test;
\c test
```
сгенерим таблицу 10гб:

`postgres=# create table key as select s, md5(random()::text) from generate_Series(1,160000000) s;`

![результат теста](/images/table.png)

Проверим размер:
```bash
postgres=# SELECT pg_size_pretty(pg_total_relation_size('"public"."test"'));
pg_size_pretty
----------------
 10 GB
```
Хост clickhouse:
Установим клик как рекомендует дока:
```bash
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754

echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update

sudo apt-get install -y clickhouse-server clickhouse-client

sudo service clickhouse-server start

clickhouse-client
```
Создадим связанную с таблицей key в постгрес таблицу в clickhouse. На данном этапе она физически не находится на хосте клика, а только по запросу получает данные по протоколу репликации.
`clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) CREATE table default.pg_test (s Int32, md5 String) ENGINE = PostgreSQL('10.128.0.19:5432', 'test', 'key', 'postgres', 'aleksei', 'public');`

Теперь перенесем таблицу test непосредственно на хост клика, для этого создадим таблицу с движком mergetree на основе исходной в постгресе
```bash
CREATE TABLE default.test Engine = MergeTree ORDER BY s AS (SELECT * FROM default.pg_test);

CREATE TABLE default.test
ENGINE = MergeTree
ORDER BY s AS
SELECT *
FROM default.pg_test

Query id: 369d5ded-cda0-4520-bf02-f02065d55c57

Ok.

0 rows in set. Elapsed: 333.781 sec. Processed 160.00 million rows, 7.20 GB (479.36 thousand rows/s., 21.57 MB/s.)
Peak memory usage: 88.46 MiB.


## Проверим быстродействие баз
### Postgresql

Psql 
\c test
test=# \timing on
Timing is on.
1.
test=# select * from key where md5 like '1234%';
Time: 43332.726 ms (00:43.333)

2.
test=# select * from key where md5 = '123457ac4ca571e651d51bf7eda83a5c';
Time: 43451.686 ms (00:43.452)

3.
select * from key where s between 57548718 and 57548735;
Time: 43394.269 ms (00:43.394)

4.
test=# select * from key where s in (57548718,159423532,134);
Time: 43372.473 ms (00:43.372)

5. 
Создадим индекс 
test=# CREATE UNIQUE INDEX s_idx ON key (s);
test=# select * from key where s in (57548718,159423532,134);
Time: 17.460 ms

6.
test=# select * from key where s in (57548718,159423532,134) and md5 like '1234%';
(2 rows)
Time: 2.834 ms

7.
test=# select * from key where s between 54000000 and 55000000 and md5 like '1234%';
(15 rows)
Time: 778.170 ms

8.
test=# select * from key order by 1 limit 10;
(1 row)
Time: 43145.207 ms (00:43.145)

### Clickhouse

1.

```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where md5 like '1234%';
2446 rows in set. Elapsed: 22.171 sec. Processed 160.00 million rows, 6.60 GB (7.22 million rows/s., 297.66 MB/s.)
Peak memory usage: 8.95 MiB.
```

2.

```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test wher   e md5 = '123457ac4ca571e651d51bf7eda83a5c';
1 row in set. Elapsed: 21.006 sec. Processed 160.00 million rows, 6.56 GB (7.62 million rows/   s., 312.29 MB/s.)
Peak memory usage: 7.96 MiB.
```

3.
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where s between 57548718 and 57548735;
18 rows in set. Elapsed: 0.024 sec. Processed 8.19 thousand rows, 191.52 KB (337.51 thousand rows/s., 7.89 MB/s.)
Peak memory usage: 12.27 KiB.
```

4.
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where s in (57548718,159423532,134);
3 rows in set. Elapsed: 0.015 sec. Processed 24.58 thousand rows, 341.39 KB (1.65 million rows/s., 22.91 MB/s.)
Peak memory usage: 27.65 KiB.
```

5.

### Создадим индекс

`CREATE UNIQUE INDEX s_idx ON default.test (s);`
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where s in (57548718,159423532,134);
3 rows in set. Elapsed: 0.005 sec. Processed 24.58 thousand rows, 341.39 KB (5.14 million rows/s., 71.34 MB/s.)
Peak memory usage: 27.52 KiB.
```

6.
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where s in (57548718,159423532,134) and md5 like '1234%';
2 rows in set. Elapsed: 0.009 sec. Processed 24.58 thousand rows, 1.04 MB (2.85 million rows/s., 120.45 MB/s.)
Peak memory usage: 26.78 KiB.
```

7.
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select * from default.test where s between 54000000 and 55000000 and md5 like '1234%';
15 rows in set. Elapsed: 0.140 sec. Processed 1.01 million rows, 41.61 MB (7.18 million rows/s., 296.30 MB/s.)
Peak memory usage: 103.27 KiB.
```

8.
```bash
clickhouse.us-central1-a.c.quixotic-moment-397713.internal :) select count(s) from default.test;
1 row in set. Elapsed: 0.013 sec.
```

## Сравнение производительности PostgreSQL и Odyssey

| № Теста | Запрос | Индекс Postgres | Индекс Clickhouse | Postgres, сек | Clickhouse, сек |
| :------: | :------: | :------: | :------: | :------: | :------: |
| 1 | select * from default.test where md5 like '1234%'; | - | - | 43,333 | 22,171 |
| 2 | select * from default.test where md5 = '123457ac4ca571e651d51bf7eda83a5c'; | 43,452 | 21,006 |
| 3 | select * from default.test where s between 57548718 and 57548735; | - | - | 43,394 | 0,024 |
| 4 | select * from default.test where s in (57548718,159423532,134); | - | - | 43,372 | 0,015 |
| 5 | select * from default.test where s in (57548718,159423532,134); | + | + | 0,017 | 0,005 |
| 6 | select * from default.test where s in (57548718,159423532,134) and md5 like '1234%'; | + | + | 0,003 | 0,009 |
| 7 | select * from default.test where s between 54000000 and 55000000 and md5 like '1234%'; | + | + | 0,778 | 0,140 |
| 8 | select * from key order by 1 limit 10; | + | + | 43,145 | 0,013 |