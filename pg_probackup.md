```bash
pg_probackup

1. Создал 2 ВМ Postgres-1 и Postgres-2
2. Подключаюсь к обоим ВМ 
3. Установил PostgreSQL 15

sudo apt update && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt update && sudo apt -y install postgresql-15 && sudo apt -y install unzip mc nano

4. Проверим установленную версию postgresql

sudo -u postgres psql -c "SELECT version();"

PostgreSQL 15.5 (Ubuntu 15.5-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit

5. Установим пароль для постгрес-пользователя postgres

sudo -u postgres psql
\password postgres
alehan

6. Добавим сетевые правила для подключения к postgres

cd /etc/postgresql/15/main/
sudo nano /etc/postgresql/15/main/postgresql.conf
listen_addresses = '*' 

sudo nano /etc/postgresql/15/main/pg_hba.conf
host    all             all             0.0.0.0/0               scram-sha-256 

7. Перезапустим кластер чтобы изменения вступили в силу

sudo pg_ctlcluster 15 main restart
sudo systemctl status postgresql@15-main.service

pg_lsclusters
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

8. Подключимся к БД на соседних хостах

PGPASSWORD=alehan psql -h 5.188.141.108 -p 5432 -U postgres -d postgres
PGPASSWORD=alehan psql -h 185.241.192.28 -p 5432 -U postgres -d postgres

9. Создадим БД alehan, добавим таблицу test с данными
sudo su postgres
psql
CREATE DATABASE alehan;
CREATE TABLE test(i int);"
INSERT INTO test VALUES (10), (20), (30);

10. Установим pg_probackup на postgres-1

sudo apt install gpg wget
wget -qO - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG-PROBACKUP | sudo tee /etc/apt/trusted.gpg.d/pg_probackup.asc
. /etc/os-release
echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb $VERSION_CODENAME main-$VERSION_CODENAME" | sudo tee /etc/apt/sources.list.d/pg_probackup.list
echo "deb-src [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb $VERSION_CODENAME main-$VERSION_CODENAME" | sudo tee -a /etc/apt/sources.list.d/pg_probackup.list
sudo apt update
apt search pg_probackup
sudo apt install pg-probackup-15

11. Поставим доп пакеты
sudo DEBIAN_FRONTEND=noninteractive apt install pg-probackup-15 pg-probackup-15-dbg postgresql-contrib postgresql-15-pg-checksums -y

12. Создаем каталог и устанавливаем переменную окружения BACKUP_PATH
sudo rm -rf /home/backups && sudo mkdir /home/backups && sudo chmod 777 /home/backups
sudo su postgres
echo "BACKUP_PATH=/home/backups/">>~/.bashrc
echo "export BACKUP_PATH">>~/.bashrc
cd $HOME
. .bashrc
postgres@postgres-1:~$ echo $BACKUP_PATH
/home/backups/

13. Создадим роль backup в PostgreSQL для выполнения бекапов и дадим ей соответствующие права
Для каждой БД:
sudo su postgres 
psql
BEGIN;
CREATE ROLE backup WITH LOGIN;
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;
ALTER ROLE backup WITH REPLICATION;
ALTER USER backup PASSWORD 'alehan';
\q
exit
echo "localhost:5432:alehan:backup:alehan">>~/.pgpass
chmod 600 ~/.pgpass

nano /etc/postgresql/15/main/pg_hba.conf
host    all             backup          localhost               scram-sha-256
psql -c "select pg_reload_conf()"

14. Включим контрольные суммы
sudo systemctl stop postgresql@15-main
postgres@postgres-1:/home/ubuntu$ /usr/lib/postgresql/15/bin/pg_checksums -D /var/lib/postgresql/15/main --enable
Checksum operation completed
Files scanned:   1554
Blocks scanned:  34753
Files written:  1285
Blocks written: 34698
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

sudo systemctl start postgresql@15-main

15. Инициализируем бэкап
pg_probackup-15 init
INFO: Backup catalog '/home/backups' successfully initialized

после инициализации появились каталоги backups и wal
cd $BACKUP_PATH
ls -l 
total 8
drwx------ 2 postgres postgres 4096 Jan  2 18:12 backups
drwx------ 2 postgres postgres 4096 Jan  2 18:12 wal

14. Инициализируем инстанс main
pg_probackup-15 add-instance --instance 'main' -D /var/lib/postgresql/15/main
INFO: Instance 'main' successfully initialized

16. Создадим новую БД
sudo su postgres
psql
CREATE DATABASE alehan;"
CREATE DATABASE

17. Создадим таблицу test в БД alehan
postgres=# \c alehan
psql (16.1 (Ubuntu 16.1-1.pgdg22.04+1), server 15.5 (Ubuntu 15.5-1.pgdg22.04+1))
You are now connected to database "alehan" as user "postgres".
CREATE TABLE test(i int);
INSERT INTO test VALUES (10), (20), (30);

alehan=# SELECT * FROM test;
 i
----
 10
 20
 30
(3 rows)

18. Создадим резервную копию кластера. Параллельно с созданием РК пустим нагрузку с другого хоста для имитации реальных условий.
ubuntu@postgres-2:~$ pgbench -h 185.241.192.28 -p 5432 -U postgres -c 50 -j 2 -P 60 -T 600 benchmark

postgres@postgres-1:/home/ubuntu$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot -h localhost -U backup -p 5432
INFO: Backup start, pg_probackup version: 2.5.13, instance: main, backup ID: S6OZHQ, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
Password for user backup:
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/backups/backups/main/S6OZHQ/database/pg_wal/000000010000000000000045 to be streamed
INFO: PGDATA size: 266MB
INFO: Current Start LSN: 0/450458A8, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1m:6s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/4C09F4F8
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup S6OZHQ
INFO: Backup S6OZHQ data files are valid
INFO: Backup S6OZHQ resident size: 394MB
INFO: Backup S6OZHQ completed

postgres@postgres-1:/home/ubuntu$ pg_probackup-15 show

BACKUP INSTANCE 'main'
======================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI    Time   Data    WAL  Zratio  Start LSN   Stop LSN    Status
======================================================================================================================================
 main      15       S6OZHQ  2024-01-03 15:41:02+00  FULL  STREAM    1/0  1m:40s  266MB  128MB    1.00  0/450458A8  0/4C09F4F8  OK
 main      15       S6OZEE  2024-01-03 15:37:51+00  FULL  STREAM    1/0     31s  272MB   16MB    1.00  0/42000028  0/4200DE80  OK



19. Настроим ssh (т.к. через него производится restore на удаленный хост)

- создать ключи для пользователя postgres на обоих хостах
  ssh-keygen -t rsa
- отредактировать конфиг ssh для того, чтобы скопировать ключи (из под суперпользователя)
  sudo nano /etc/ssh/sshd_config
  PasswordAuthentication yes
  PubkeyAuthentication yes
  После копирования ключа на другой хост можно отключить PasswordAuthentication
- перезапустить службу sshd (ssh)
  sudo systemctl resatrt sshd.service
- Создать пароль пользователя postgres
  sudo passwd postgres
- Копировать публичный ключ на другой хост
  postgres@postgres-1:/home/ubuntu$ ssh-copy-id -i ~/.ssh/id_rsa.pub postgres@5.188.141.108
  postgres@postgres-2:/home/ubuntu$ ssh-copy-id -i ~/.ssh/id_rsa.pub postgres@185.241.192.28
- Проверить соединение по ssh
  ssh postgres@5.188.141.108
  ssh postgres@185.241.192.28


20. На postgres-2 остановим кластер
sudo systemctl stop postgresql@15-main.service
Удалим данные кластера
sudo su postgres
rm -rf /var/lib/postgresql/15/main/*

21. На postgres-1 восстановим РК на postgres-2
pg_probackup-15 restore --instance 'main' -D /var/lib/postgresql/15/main --remote-host=5.188.141.108 --remote-proto=ssh

INFO: Validating backup S6OZHQ
INFO: Backup S6OZHQ data files are valid
INFO: Backup S6OZHQ WAL segments are valid
INFO: Backup S6OZHQ is valid.
INFO: Restoring the database from backup at 2024-01-03 15:39:26+00
INFO: Start restoring backup files. PGDATA size: 394MB
INFO: Backup files are restored. Transfered bytes: 394MB, time elapsed: 20s
INFO: Restore incremental ratio (less is better): 100% (394MB/394MB)
INFO: Syncing restored files to disk
INFO: Restored backup files are synced, time elapsed: 1s
INFO: Restore of backup S6OZHQ completed.

22. Запустим кластер на postgres-2
sudo systemctl start postgresql@15-main.service
проверим состояние кластера после запуска
sudo systemctl status postgresql@15-main.service

● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Wed 2024-01-03 20:47:17 UTC; 25min ago
    Process: 118724 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=1/FAILURE)
   Main PID: 118729 (postgres)
      Tasks: 6 (limit: 1030)
     Memory: 234.7M
        CPU: 1.311s
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─118729 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─118730 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─118731 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─118741 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─118742 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             └─118743 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">

Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:18.281 UTC [118732] LOG:  database system was interrupted; last known up at 2024-01-03 15:39:53>
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:18.281 UTC [118732] LOG:  creating missing WAL directory "pg_wal/archive_status"
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:18.388 UTC [118732] LOG:  redo starts at 0/450458A8
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:30.081 UTC [118732] LOG:  redo in progress, elapsed time: 11.67 s, current LSN: 0/4987AB70
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:38.408 UTC [118732] LOG:  redo in progress, elapsed time: 20.01 s, current LSN: 0/4BAC3688
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:41.047 UTC [118732] LOG:  consistent recovery state reached at 0/4C09F4F8
Jan 03 20:47:17 postgres-2 postgresql@15-main[118724]: 2024-01-03 20:46:41.074 UTC [118732] LOG:  redo done at 0/4C09F4F8 system usage: CPU: user: 0.29 s, system: 0.38

Кластер успешно запустился.

23. Теперь проверим БД
sudo su postgres
psql
\du
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------------------
 backup    | Replication
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS

\c alehan
You are now connected to database "alehan" as user "postgres"

                                                   List of databases
   Name    |  Owner   | Encoding | Locale Provider | Collate |  Ctype  | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+---------+---------+------------+-----------+-----------------------
 alehan    | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
 benchmark | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
 postgres  | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |         |         |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |         |         |            |           | postgres=CTc/postgres
(5 rows)

alehan=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

alehan=# select * from test;
 i
----
 10
 20
 30
  4
(4 rows)

Все данные успешно перенеслись.
```
