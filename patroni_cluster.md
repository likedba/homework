```bash
Создадим 3 машины node1, node2, node3
gcloud beta compute --project=quixotic-moment-397713 instances create node1 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=node1 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud beta compute --project=quixotic-moment-397713 instances create node2 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=node2 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud beta compute --project=quixotic-moment-397713 instances create node3 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=node3 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

На всех нодах установим постгрес и другие необходимые пакеты

sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15

sudo apt install git python3 python3-pip etcd python3-dev libpq-dev haproxy pacemaker corosync pcs fence-agents keepalived
sudo pip3 install --upgrade pip
pip3 install psycopg2-binary patroni[etcd] python-etcd

sudo nano /etc/default/etcd
ETCD_NAME=etcd_node1
ETCD_DATA_DIR="/var/lib/etcd/etcd_node.etcd"
ETCD_LISTEN_PEER_URLS="http://10.128.15.197:2380"
ETCD_LISTEN_CLIENT_URLS=http://localhost:2379,http://10.128.15.197:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.128.15.197:2380
ETCD_INITIAL_CLUSTER="etcd_node1=http://10.128.15.197:2380,etcd_node2=http://10.128.15.1>
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS=http://10.128.15.197:2379

ETCD_NAME=etcd_node2
ETCD_DATA_DIR="/var/lib/etcd/etcd_node.etcd"
ETCD_LISTEN_PEER_URLS="http://10.128.15.198:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.128.15.198:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.128.15.198:2380
ETCD_INITIAL_CLUSTER="etcd_node1=http://10.128.15.197:2380,etcd_node2=http://10.128.15.1>
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS=http://10.128.15.198:2379

ETCD_NAME=etcd_node3
ETCD_DATA_DIR="/var/lib/etcd/etcd_node3.etcd"
ETCD_LISTEN_PEER_URLS="http://10.128.15.199:2380"
ETCD_LISTEN_CLIENT_URLS=http://localhost:2379,http://10.128.15.199:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.128.15.199:2380
ETCD_INITIAL_CLUSTER="etcd_node1=http://10.128.15.197:2380,etcd_node2=http://10.128.15.1>
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS=http://10.128.15.199:2379

data dir - /var/lib/postgresql/15/main
exec dir - /usr/lib/postgresql/15/bin/

nano  .profile

export PGDATA="/var/lib/postgresql/15/main"
export ETCDCTL_API="3"
export PATRONI_ETCD_URL="http://127.0.0.1:2379"
export PATRONI_SCOPE="pg_cluster"
etcd_node1=10.128.15.197
etcd_node2=10.128.15.198
etcd_node3=10.128.15.199
ENDPOINTS=$etcd_node1:2379,$etcd_node2:2379,$etcd_node3:2379

source ~/.profile
systemctl enable etcd
etcdctl endpoint status --write-out=table --endpoints=$ENDPOINTS
+--------------------+------------------+---------+---------+-----------+-----------+------------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+--------------------+------------------+---------+---------+-----------+-----------+------------+
| 10.128.15.197:2379 | 8f30599bd38189c0 |  3.2.26 |   25 kB |     false |         2 |          9 |
| 10.128.15.198:2379 | c4344704d0137568 |  3.2.26 |   25 kB |     false |         2 |          9 |
| 10.128.15.199:2379 | 31d42228a18c02c1 |  3.2.26 |   25 kB |      true |         2 |          9 |
+--------------------+------------------+---------+---------+-----------+-----------+------------+


Настройка Patroni
sudo mkdir /patroni
sudo mkdir /patroni/cluster
sudo chown postgres:postgres /patroni
sudo chown postgres:postgres /patroni/cluster
sudo chmod 700 /patroni
sudo chmod 700 /patroni/cluster
sudo nano /etc/patroni.yml 

nano /etc/patroni.yml
(настройки для первой ноды, остальные аналогично)
scope: pg_cluster
namespace: /service/
name: node1
restapi:
    listen: 10.128.15.197:8008
    connect_address: 10.128.15.197:8008
etcd3:
    hosts: 10.128.15.197:2379, 10.128.15.198:2379, 10.128.15.199:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - local   all           all                            trust
  - host    all           all          127.0.0.1/32      md5
  - host    replication   replicator   ::/0              md5
  - host    replication   replicator   127.0.0.1/32      md5
  - host    replication   replicator   10.128.15.197/0     md5
  - host    replication   replicator   10.128.15.198/0     md5
  - host    replication   replicator   10.128.15.199/0     md5
  - host    all           all          10.128.15.197/0     md5
  - host    all           all          10.128.15.198/0     md5
  - host    all           all          10.128.15.199/0     md5
  - host    all           all          ::/0              md5
  - host    all           all          all               md5
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb
postgresql:
  listen: 10.128.15.197:5432
  connect_address: 10.128.15.197:5432
  data_dir: /patroni/cluster
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass
  use_unix_socket: /var/run/postgresql
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
    rewind:
      username: rewind
      password: rewind
  synchronous_mode: true
  synchronous_commit: "on"
  synchronous_standby_names: "*"
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false


sudo su postgres
nano /tmp/pgpass

localhost:*:admin:admin
localhost:*:postgres:postgres
localhost:*:replicator:replicator
localhost:*:rewind:rewind

sudo nano /etc/systemd/system/patroni.service

[Unit]
Description=HA Postgresql Cluster
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
sudo systemctl daemon-reload

Стартуем patroni на всех нодах, и в статусе можем видеть, что у первой ноды статус лидера:
Nov 02 07:48:58 node1 patroni[30352]: 2023-11-02 07:48:58,804 INFO: no action. I am (node1), the leader with the lock

А вторая и третья видят лидера и имеют статус slave
Nov 02 07:40:48 node2 patroni[21847]: 2023-11-02 07:40:48,279 INFO: no action. I am (node2), a secondary, and following a leader (node1)
Nov 02 07:51:39 node3 patroni[21938]: 2023-11-02 07:51:39,399 INFO: no action. I am (node3), a secondary, and following a leader (node1)

patronictl -c /etc/patroni.yml list
+ Cluster: pg_cluster (7296764204462331552) ---+----+-----------+
| Member | Host          | Role    | State     | TL | Lag in MB |
+--------+---------------+---------+-----------+----+-----------+
| node1  | 10.128.15.197 | Leader  | running   |  1 |           |
| node2  | 10.128.15.198 | Replica | streaming |  1 |         0 |
| node3  | 10.128.15.199 | Replica | streaming |  1 |         0 |
+--------+---------------+---------+-----------+----+-----------+

Настроим синхронную реплику
patronictl -c /etc/patroni.yml edit-config

loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  parameters: null
  use_pg_rewind: true
  use_slots: true
retry_timeout: 10
synchronous_mode: true
ttl: 30

Видим, что вторая нода стала синхронной
+ Cluster: pg_cluster (7296764204462331552) --------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| node1  | 10.128.15.197 | Leader       | running   |  1 |           |
| node2  | 10.128.15.198 | Sync Standby | streaming |  1 |         0 |
| node3  | 10.128.15.199 | Replica      | streaming |  1 |         0 |
+--------+---------------+--------------+-----------+----+-----------+


Настроим балансировщик haproxy:

sudo nano /etc/haproxy/haproxy.cfg

global
maxconn 100
defaults
log global
mode tcp
retries 2
timeout client 30m
timeout connect 4s
timeout server 30m
timeout check 5s

listen stats
mode http
bind *:7000
stats enable
stats uri /

listen postgres
bind *:5433
option httpchk
http-check expect status 200
default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
server postgresql_10.128.15.197_5432 10.128.15.197:5432 maxconn 100 check port 8008
server postgresql_10.128.15.198_5432 10.128.15.198:5432 maxconn 100 check port 8008
server postgresql_10.128.15.199_5432 10.128.15.199:5432 maxconn 100 check port 8008


sudo nano /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl status haproxy



Настроим Keepalived

На первой ноде настроим конфиг keepalived:

 sudo nano /etc/keepalived/keepalived.conf

global_defs {
    router_id haproxy1
    enable_script_security
}

vrrp_script haproxy_check {
   script "/usr/libexec/keepalived/haproxy_check.sh"
   interval 2
   weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens4
    virtual_router_id 51
    priority 80
    advert_int 1

    track_script {
       haproxy_check
    }
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        10.128.15.195
    }
}

На двух остальных нодах настроим конфиг keepalived:

global_defs {
    router_id haproxy2
    enable_script_security
}

vrrp_script haproxy_check {
   script "/usr/libexec/keepalived/haproxy_check.sh"
   interval 2
   weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens4
    virtual_router_id 51
    priority 70
    advert_int 1

    track_script {
       haproxy_check
    }
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        10.128.15.195
    }
}


sudo mkdir -p /usr/libexec/keepalived/
sudo nano /usr/libexec/keepalived/haproxy_check.sh

#!/bin/bash
count=`ps aux | grep -v grep | grep haproxy | wc -l`
if [ $count -eq 0 ]; then
    exit 1
else
    exit 0
fi

sudo useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script
sudo groupadd -r keepalived_script
sudo chmod 700 /usr/libexec/keepalived/haproxy_check.sh
sudo chown keepalived_script:keepalived_script /usr/libexec/keepalived/haproxy_check.sh
sudo chmod +x /usr/libexec/keepalived/haproxy_check.sh

sudo systemctl start keepalived
sudo systemctl enable keepalived
sudo systemctl status keepalived

```

>## Итог
>Имеем отказоустойчивый кластер, состоящий из отказоустойчивого хранилища состояния кластера ETCD, кластера postgresql под управлением patroni, балансировщик haproxy, который направляет трафик на мастер, и keepalive для переключения точки входа (VIP) на нужную ноду в случае сбоя. Также, в случае большого количества клиентских подключений (больше 300), между haproxy и postgresql устанавливается пулер соединений, к примеру, pgbouncer или odyssey.
>
>ETCD кластер:
>```bash
>+--------------------+------------------+---------+---------+-----------+-----------+------------+
>|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
>+--------------------+------------------+---------+---------+-----------+-----------+------------+
>| 10.128.15.197:2379 | 8f30599bd38189c0 |  3.2.26 |   49 kB |      true |         2 |         57 |
>| 10.128.15.198:2379 | c4344704d0137568 |  3.2.26 |   49 kB |     false |         2 |         57 |
>| 10.128.15.199:2379 | 31d42228a18c02c1 |  3.2.26 |   49 kB |     false |         2 |         57 |
>+--------------------+------------------+---------+---------+-----------+-----------+------------+
>```
>Patroni кластер:
>```bash
>+ Cluster: pg_cluster (7296764204462331552) --------+----+-----------+
>| Member | Host          | Role         | State     | TL | Lag in MB |
>+--------+---------------+--------------+-----------+----+-----------+
>| node1  | 10.128.15.197 | Leader       | running   |  1 |           |
>| node2  | 10.128.15.198 | Sync Standby | streaming |  1 |         0 |
>| node3  | 10.128.15.199 | Replica      | streaming |  1 |         0 |
>+--------+---------------+--------------+-----------+----+-----------+
>```
>Точка входа на 10.128.15.195/32:
> ```bash
>a_salugin@node2:~$ ip a
>1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
>    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
>    inet 127.0.0.1/8 scope host lo
>       valid_lft forever preferred_lft forever
>    inet6 ::1/128 scope host 
>       valid_lft forever preferred_lft forever
>2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP group default qlen 1000
>    link/ether 42:01:0a:80:0f:c6 brd ff:ff:ff:ff:ff:ff
>    altname enp0s4
>    inet 10.128.15.198/32 scope global dynamic ens4
>       valid_lft 3201sec preferred_lft 3201sec
>    inet 10.128.15.195/32 scope global ens4
>       valid_lft forever preferred_lft forever
>    inet6 fe80::4001:aff:fe80:fc6/64 scope link 
>       valid_lft forever preferred_lft forever
> ```
>В данном кластере, haproxy общается с restapi patroni через порт 8008, посылая запрос и видит у какой ноды статус мастера. Соответственно весь трафик направляется на мастер. \
>Но также, если приклад поддерживает функционал разделения трафика на пишущий и читающий по разным портам для haproxy, то можно настроить, чтобы haproxy пишущие запросы перенаправлял на мастер, а читающие на реплики (olap). \
>Кроме этого есть еще самый правильный вариант, когда у нас имеется синхронная реплика в кластере, тогда можно настроить, чтобы haproxy принимал трафик трех категорий на разные порты (пишущий, oltp, olap) и перенаправлял пишущие запросы на мастер, читающие oltp на синхронную реплику, читающие olap на асинхронные раплики. \
>Также хотелось бы добавить, что в данной конфигурации, отказоустойчивость кластера сохраняется пока 2 из 3 нод работают. В случае падения двух нод, оставшаяся нода перейдет в режим только чтения, тк не сможет явно убедиться в отсутствии split brain в кластере. \
>Для того, чтобы работа кластера сохранялась даже в случае падения двух нод, необходимо добавить еще 2 ноды для хранилища etcd.

