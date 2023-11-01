```bash
В Google Cloud SDK Shell
gcloud beta compute --project=quixotic-moment-397713 instances create citus1 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=citus1 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud beta compute --project=quixotic-moment-397713 instances create citus2 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=citus2 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud beta compute --project=quixotic-moment-397713 instances create citus3 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=citus3 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

C:\Users\alext\AppData\Local\Google\Cloud SDK>gcloud compute ssh citus1
C:\Users\alext\AppData\Local\Google\Cloud SDK>gcloud compute ssh citus2
C:\Users\alext\AppData\Local\Google\Cloud SDK>gcloud compute ssh citus3

Устанавливаем citus на все машины:
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && curl https://install.citusdata.com/community/deb.sh | sudo bash

sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-16-citus-12.1

Добавим citus в shared_preload_libraries
sudo pg_conftool 16 main set shared_preload_libraries citus

Настроим доступ к базе
sudo pg_conftool 16 main set listen_addresses '*'
sudo su postgres
nano /etc/postgresql/16/main/pg_hba.conf
host  all all 10.128.0.0/16            trust
host    all             all             127.0.0.1/32            trust -- прям заменим scram-sha-256 только на контроллере
host    all             all             0.0.0.0/0            scram-sha-256  -- только на контроллере

sudo service postgresql restart
sudo update-rc.d postgresql enable
sudo su postgres
psql

На всех нодах:
БД на каждой ноде создается вручную
CREATE DATABASE aleksei;
\c aleksei
Экстеншн Citus создается на каждой ноде, для каждой базы
Создадим экстеншн для БД aleksei
CREATE EXTENSION citus;

Создал бакет aleksei_taxi
загрузил в него данные chicago taxi
bq extract bigquery-public-data:chicago_taxi_trips.taxi_trips gs://aleksei_taxi/taxi.csv.*

На координаторе
sudo su postgres
cd ~
mkdir taxi

Установим gcsfuse
export GCSFUSE_REPO=gcsfuse-`lsb_release -c -s`
echo "deb https://packages.cloud.google.com/apt $GCSFUSE_REPO main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install gcsfuse
gcsfuse aleksei_taxi taxi

создадим в БД aleksei таблицу taxi_trips
sudo su postgres
psql
\c aleksei
create table taxi_trips (
unique_key text primary key,
taxi_id text,
trip_start_timestamp TIMESTAMP,
trip_end_timestamp TIMESTAMP,
trip_seconds bigint,
trip_miles numeric,
pickup_census_tract bigint,
dropoff_census_tract bigint,
pickup_community_area bigint,
dropoff_community_area bigint,
fare numeric,
tips numeric,
tolls numeric,
extras numeric,
trip_total numeric,
payment_type text,
company text,
pickup_latitude numeric,
pickup_longitude numeric,
pickup_location text,
dropoff_latitude numeric,
dropoff_longitude numeric,
dropoff_location text
);

SELECT * FROM master_add_node('citus2', 5432);
SELECT * FROM master_add_node('citus3', 5432);
SELECT create_distributed_table('taxi_trips', 'unique_key');
Загрузим данные в постгрес
for f in *.csv*; do psql -d aleksei -c "\\copy taxi_trips from program 'cat $f' csv header"; done
select rebalance_table_shards('taxi_trips');


Результаты по скорости
select * from taxi_trips order by 2 desc limit 10\gx
Time: 114169.923 ms (01:54.170)

select * from taxi_trips limit 10;
Time: 135.467 ms

select * from taxi_trips order by trip_seconds limit 10\gx
Time: 68147.283 ms (01:08.147)




Создал новую машину одиночным инстансом postgresql, подключил бакет, загрузил данные, и запустил те же запросы, что и в кластере citus


select * from taxi_trips order by trip_seconds limit 10\gx
Time: 305981.399 ms (05:05.981)

Вывод - кластер citus выполнил запрос в 5 раз быстрее.

```