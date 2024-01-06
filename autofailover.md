```bash
Postgres Autofailover

1. подключим репозиттории postgresql и устанавливаем postgresql-15
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-15

2. На первой ноде установим монитор
- предварительно остановим кластер
  sudo systemctl stop postgresql@15-main.service
sudo apt-get install postgresql-15-auto-failover

Запустим создание монитора
pg_autoctl create monitor --auth trust --ssl-mode require --ssl-self-signed --pgport 5432 --hostname postgres-1 --pgctl /usr/lib/postgresql/15/bin/pg_ctl

Изменим ph_hba.conf
nano /var/lib/postgresql/15/main/pg_hba.conf
local all autoctl_node trust

Запустим монитор от пользователя postgres
pg_autoctl run

в новой сессии к монитору
создадим пароль для пользователя pg_autoctl run
psql
alter user autoctl_node password 'alehan';

выполним от пользователя root для создания systemd юнита для autofailover
sudo pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/15/main" | sudo tee /etc/systemd/system/pgautofailover.service
sudo systemctl daemon-reload
sudo systemctl enable pgautofailover
sudo systemctl start pgautofailover

посмотрим состояние кластера (должна быть задана переменная pgdata)
pg_autoctl show state
Name |  Node |  Host:Port |  TLI: LSN |   Connection |      Reported State |      Assigned State
-----+-------+------------+-----------+--------------+---------------------+--------------------

Отредактируем pg_hba.conf т.к. автоматом в нем прописывается строка, которая подразумевает подключение технического пользователя в одной подсети, однако если ноды находятся в разных сетях, то пропишем строку:
sudo su postgres
nano /var/lib/postgresql/15/main/pg_hba.conf
hostssl all             autoctl_node    0.0.0.0/0                trust
psql
SELECT pg_reload_conf();

На нодах 2 и 3
- установим autofailover
sudo apt-get install postgresql-15-auto-failover
- остановим кластер 
sudo systemctl stop postgresql@15-main.service
sudo su postgres
- удалим автоматом созданный при установке кластер 
pg_dropcluster 15 main
- создадим новый кластер
pg_createcluster -u postgres -p 5432 -d /var/lib/postgresql/15/alehan --start-conf=manual -l /var/log/postgresql/alehan.log 15 alehan
- обновим переменную PGDATA ( от пользователя postgres)
export PGDATA="/var/lib/postgresql/15/alehan"
nano ~/.profile
export PGDATA="/var/lib/postgresql/15/alehan"

Создадим мастер на ноде 2
pg_autoctl create postgres --hostname postgres-2 --name postgres-2 --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node:alehan@185.241.192.28:5432/pg_auto_failover?sslmode=require'

pg_autoctl run

В новой сессии запускаем от рута
sudo pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/15/alehan" | sudo tee /etc/systemd/system/pgautofailover.service
sudo systemctl daemon-reload
sudo systemctl enable pgautofailover
sudo systemctl start pgautofailover

Посмотрим статус кластера на ноде мониторе
postgres@postgres-1:/home/ubuntu$ pg_autoctl show state
  Name |  Node |       Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | postgres-2:5432 |   1: 0/15D5028 | read-write ! |              single |              single

Зададим пароль для репликационного пользователя pgautofailover_replicator (от postgres)
pg_autoctl config set replication.password 'alehan'
psql
alter user pgautofailover_replicator password 'alehan';

На 3 ноде повторяем все опереации как на второй

pg_autoctl create postgres --hostname postgres-3 --name postgres-3 --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node:alehan@185.241.192.28:5432/pg_auto_failover?sslmode=require'

Если не проходит создание мастера, то удалим файл статуса кластера (т.к. изза имеющегося предыдущего состояния он не будет запускаться)
rm -rf /var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/15/alehan/*
rm -rf /var/lib/postgresql/.config/pg_autoctl/*

pg_autoctl run

В новой сессии запускаем от рута
sudo pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/15/alehan" | sudo tee /etc/systemd/system/pgautofailover.service
sudo systemctl daemon-reload
sudo systemctl enable pgautofailover
sudo systemctl start pgautofailover

Посмотрим статус кластера на ноде мониторе
postgres@postgres-2:/home/ubuntu$ pg_autoctl show state
  Name |  Node |       Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | postgres-2:5432 |   1: 0/15D5028 |      primary |             primary |             primary
node_2 |     2 | postgres-3:5432 |   1: 0/15D5028 |    secondary |           secondary |           secondary
```
