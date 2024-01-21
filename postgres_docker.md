```bash

1. Создадим ВМ postgres
gcloud beta compute --project=quixotic-moment-397713 instances create postgres --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=node1 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

2. Подключимся к виртуалке
ssh ubuntu@174.138.80.42
ubuntu@postgres:~$ sudo apt-get update
ubuntu@postgres:~$ sudo install -m 0755 -d /etc/apt/keyrings
ubuntu@postgres:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
ubuntu@postgres:~$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
ubuntu@postgres:~$ echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
ubuntu@postgres:~$ sudo apt-get update
ubuntu@postgres:~$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

3. Установим docker compose
ubuntu@postgres:~$ sudo apt install docker-compose

4. Создадим каталог для данных
ubuntu@postgres:~$ sudo mkdir /var/lib/postgres

5. Создадим docker-compose.yaml для постгреса
----------------------------
version: '3.1'

services:
  pg_db:
    image: postgres:15.4
    name: mypg
    restart: always
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=test
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - ${POSTGRES_DIR:-/var/lib/postgres}:/var/lib/postgresql/data
    ports:
      - ${POSTGRES_PORT:-5432}:5432
--------------------------------

6. Запустим контейнер
ubuntu@postgres:~$ sudo docker-compose up -d
ubuntu@postgres:~$ sudo docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS                                       NAMES
7d064c79da5a   postgres:15.4   "docker-entrypoint.s…"   35 seconds ago   Up 33 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   mypg

7. Запустим psql в контейнере
Создадим докер файл
FROM alpine:3.18
RUN apk --no-cache add postgresql15-client
ENTRYPOINT [ "psql" ]

8. Собираем образ
ubuntu@postgres:~$ sudo docker build -t pg-client .

9. создадим файл с переменными окружения .env_pg
--------------------------------
PGDATABASE=test
PGHOST=127.0.0.1
PGPORT=5432
PGUSER=postgres
PGPASSWORD=postgres
--------------------------------

10. Запуск контейнера с клиентом
ubuntu@postgres:~$ sudo docker run --env-file=.env_pg --name=mypgclient --network="host" -it pg-client
psql (15.4)
Type "help" for help.

test=#

11. Создание таблицы и наполнение данными
test=# create table test(s text);
CREATE TABLE
test=# insert into test(s) select gen_random_uuid () from generate_series(1,10);
INSERT 0 10
test=# select * from test;
                  s                   
--------------------------------------
 7d643618-472f-44ba-b645-3f0d0c750df6
 78160b04-4487-4e88-8e05-130bf4ea1f83
 8efbe2f6-e8bc-4aef-9103-2dfcf295ade2
 03322eff-0ecc-46c6-be73-c7ef5a20089b
 efa1684e-37a9-4730-b689-9d1dead577d4
 9e02049e-45f5-4206-b19c-295403b50839
 2e3cf6ae-9bdc-4099-8056-4d0f990421f9
 34032890-accb-4821-969e-9e52f52cbf26
 0deb0e91-05fd-463f-b7c1-805abfef88d8
 f00f0b7f-8ec4-4093-b12a-bb5be5c088ca
(10 rows)

12. Проверка таблицы на диске
test=# select pg_relation_filepath('test');
 pg_relation_filepath 
----------------------
 base/16384/16389
(1 row)

ubuntu@postgres:~$ sudo ls -la /var/lib/postgres/pgdata/base/16384 | grep 16389
-rw------- 1 lxd docker   8192 Sep 13 09:11 16389

13. Проверка внешнего подключения
psql -h 158.160.80.124 -U postgres -d test
Пароль пользователя postgres: 
psql (15.4)
Type "help" for help.

test=# select count(*) from test;
 count 
-------
    10
(1 строка)

14. Пересоздание контейнера
ubuntu@postgres:~$ sudo docker container ls -a
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS                      PORTS                                       NAMES
fe2b7a231b02   pg-client       "psql"                   24 minutes ago   Exited (0) 22 minutes ago                                               mypgclient
2f078d79ea3b   postgres:15.4   "docker-entrypoint.s…"   2 hours ago      Up 2 hours                  0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   mypg
ubuntu@postgres:~$ sudo docker stop mypg
mypg
ubuntu@postgres:~$ sudo docker rm mypg
mypg
ubuntu@postgres:~$ sudo docker container ls -a
CONTAINER ID   IMAGE       COMMAND   CREATED          STATUS                      PORTS     NAMES
fe2b7a231b02   pg-client   "psql"    25 minutes ago   Exited (0) 23 minutes ago             mypgclient
ubuntu@postgres:~$ sudo docker container ls -a
CONTAINER ID   IMAGE       COMMAND   CREATED          STATUS                      PORTS     NAMES
fe2b7a231b02   pg-client   "psql"    25 minutes ago   Exited (0) 23 minutes ago             mypgclient
ubuntu@postgres:~$ sudo docker-compose up -d
Creating mypg ... done

15. Пересоздаем клиентский контейнер (т.к. он имеет фиксированное имя), подключаемся, и проверяем данные
ubuntu@postgres:~$ sudo docker rm mypgclient
mypgclient
ubuntu@postgres:~$ sudo docker run --env-file=.env_pg --name=mypgclient --network="host" -it pg-client
psql (15.4)
Type "help" for help.

test=# select * from test;
                  s                   
--------------------------------------
 7d643618-472f-44ba-b645-3f0d0c750df6
 78160b04-4487-4e88-8e05-130bf4ea1f83
 8efbe2f6-e8bc-4aef-9103-2dfcf295ade2
 03322eff-0ecc-46c6-be73-c7ef5a20089b
 efa1684e-37a9-4730-b689-9d1dead577d4
 9e02049e-45f5-4206-b19c-295403b50839
 2e3cf6ae-9bdc-4099-8056-4d0f990421f9
 34032890-accb-4821-969e-9e52f52cbf26
 0deb0e91-05fd-463f-b7c1-805abfef88d8
 f00f0b7f-8ec4-4093-b12a-bb5be5c088ca
(10 rows)
```
