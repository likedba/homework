```bash
VK Cloud
1. Зарегистрировался в VK Cloud, получил тестовые 3к на счет
2. Создал 2 ВМ Postgres-1 и Postgres-2
3. Подключаюсь к ВМ при помощи ssh клиента MobaXterm. Создаю новую сессию SSH, используя внешний IP указанный в граф. интерфейсе VK cloud, linux пользователя ubuntu, а также указав в вкладке Advanced SSH settings, в параметре use private key, ssh ключ котторый автоматом сохранился в загрузки в браузере при создании ВМ
4. Установил PostgreSQL 15

sudo apt update && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt update && sudo apt -y install postgresql-15 && sudo apt -y install unzip mc nano

5. Проверим установленную версию postgresql

sudo -u postgres psql -c "SELECT version();"

PostgreSQL 15.5 (Ubuntu 15.5-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit

6. Установим пароль для постгрес-пользователя postgres

sudo -u postgres psql
\password postgres
alehan

7. Добавим сетевые правила для подключения к postgres

cd /etc/postgresql/15/main/
sudo nano /etc/postgresql/15/main/postgresql.conf
listen_addresses = '*' 

sudo nano /etc/postgresql/15/main/pg_hba.conf
host    all             all             0.0.0.0/0               scram-sha-256 

8. Перезапустим кластер чтобы изменения вступили в силу

sudo pg_ctlcluster 15 main restart
sudo systemctl status postgresql@15-main.service

pg_lsclusters
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

9. Подключимся к БД на соседних хостах

PGPASSWORD=alehan psql -h 5.188.141.108 -p 5432 -U postgres -d postgres
PGPASSWORD=alehan psql -h 185.241.192.28 -p 5432 -U postgres -d postgres

10. Проведем тест при помощи утилиты pgbench
export PGPASSWORD=alehan
ubuntu@postgres-1:~$ psql -h 5.188.141.108 -p 5432 -U postgres -d postgres -c "CREATE DATABASE benchmark;"
ubuntu@postgres-2:~$ psql -h 185.241.192.28 -p 5432 -U postgres -d postgres -c "CREATE DATABASE benchmark;"

11. Инициализируем БД benchmark для нагрузочного теста
ubuntu@postgres-1:~$ pgbench -h 5.188.141.108 -p 5432 -U postgres -i -s 15 benchmark
ubuntu@postgres-2:~$ pgbench -h 185.241.192.28 -p 5432 -U postgres -i -s 15 benchmark

12. Запускаем тесты с одного хоста на БД на другом хосте
ubuntu@postgres-1:~$ pgbench -h 5.188.141.108 -p 5432 -U postgres -c 50 -j 2 -P 60 -T 180 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 360.4 tps, lat 134.225 ms stddev 231.759, 0 failed
progress: 120.0 s, 191.1 tps, lat 263.433 ms stddev 404.289, 0 failed
progress: 180.0 s, 214.0 tps, lat 234.497 ms stddev 355.987, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 15
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 45984
number of failed transactions: 0 (0.000%)
latency average = 194.426 ms
latency stddev = 323.661 ms
initial connection time = 1261.298 ms
tps = 257.010874 (without initial connection time)

ubuntu@postgres-2:~$ pgbench -h 185.241.192.28 -p 5432 -U postgres -c 50 -j 2 -P 60 -T 180 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 364.3 tps, lat 130.747 ms stddev 264.246, 0 failed
progress: 120.0 s, 199.0 tps, lat 256.818 ms stddev 458.134, 0 failed
progress: 180.0 s, 214.1 tps, lat 229.762 ms stddev 326.738, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 15
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 46696
number of failed transactions: 0 (0.000%)
latency average = 191.772 ms
latency stddev = 347.874 ms
initial connection time = 1457.922 ms
tps = 260.554809 (without initial connection time)
```
