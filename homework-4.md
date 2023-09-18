**Тестируемые машины:**
host1
host2

**Конфигурация host1:**
5 ядер
35 Гб ОЗУ
SSD массив

**Конфигурация host2:**
12 ядер
63 Гб ОЗУ
SSD массив

**Нагрузка:**
pgbench -U pgbench -c 80 -T 600 -P 60 pgbench

# Тест №1
Измерим производительность БД со стандартными настройками ОС и БД

host1:                                                                                            host2:
Стандартные настройки postgresql                                                                  Стандартные настройки postgresql
Профиль ОС balanced                                                                               Профиль ОС balanced

TPS:                                                                                              TPS:
    1. 5263                                                                                       13076
    2. 5286                                                                                       12913
    3. 5302                                                                                       12971
**Avarage tps=  5284                                                                                Avarage tps= 12987**

# Тест №2
**Измерим производительность БД с измененными настройками ОС и стандартными настройками БД**

host1:                                                                                            host2:
Стандартные настройки postgresql                                                                  Стандартные настройки postgresql
Профиль ОС postgres                                                                               Профиль ОС postgres

Создадим профиль "postgres" утилиты tuned:

[main]
summary=Tuned profile for  PostgreSQL Instances

[bootloader]
cmdline=transparent_hugepage=never

[cpu]
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[sysctl]
vm.swappiness = 10
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 250
vm.dirty_ratio = 10
vm.dirty_background_ratio = 3
vm.overcommit_memory=0
net.ipv4.tcp_timestamps=0

[vm]
transparent_hugepages=never

Применим профиль postgres на обоих машинах

tuned-adm profile postgres

Проверим производительность БД после внесенных изменений

TPS:                                                                                              TPS:
    1. 5348                                                                                        12727
    2. 5300                                                                                        12847
    3. 5324                                                                                        12549
**Avarage tps=  5324                                                                                Avarage tps= 12708**

# Тест №3
**Измерим производительность БД с измененными настройками ОС и БД**

host1:                                                                                            host2:
Отредактированные настройки postgresql                                                            Отредактированные настройки postgresql
Профиль ОС postgres                                                                               Профиль ОС postgres

postgresql.conf       (для host2 отмечены только отличающиесянастройки)

data_directory = '/pgdata/'
hba_file = '/pgdata/pg_hba.conf'
ident_file = '/pgdata/pg_ident.conf'
listen_addresses = '*'
max_connections = 1400                    # В этом проекте пока использование пулера не представляется возможным
shared_buffers = 8192MB                                                                           shared_buffers = 16384MB
work_mem = 32MB
maintenance_work_mem = 420MB                                                                      maintenance_work_mem = 520MB
effective_io_concurrency = 200
maintenance_io_concurrency = 200
max_worker_processes = 5                                                                          max_worker_processes = 12
max_parallel_workers_per_gather = 3                                                               max_parallel_workers_per_gather = 6
max_parallel_maintenance_workers = 3                                                              max_parallel_maintenance_workers = 6
max_parallel_workers = 5                                                                          max_parallel_workers = 12
wal_compression = on                      # поскольку узким местом большинства серверов является io а не cpu
wal_init_zero = off                       # более эффективные операции с вал в файловых системах COW
wal_recycle = off                         # на виртуальных машинах быстрее создание новых вал
wal_buffers = 64MB
max_wal_size = 5GB
min_wal_size = 2500MB
seq_page_cost = 1.0
random_page_cost = 1.25
cpu_tuple_cost = 0.03
effective_cache_size = 23GB                                                                        effective_cache_size = 40GB
shared_preload_libraries = 'pg_stat_statements,pg_prewarm'

TPS:                                                                                               TPS:
    1. 5223                                                                                         13996
    2. 5314                                                                                         14103
    3. 5253                                                                                         14007
**Avarage tps=  5263                                                                                 Avarage tps= 14035**

# Тест №4
**Измерим производительность БД с измененными настройками ОС и БД + настроенные huge pages, scheduler и параметром rota для диска с pgdata**

**Настроим huge pages:**

host1:                                                                                              host2:

`psql`                                                                                                psql
`postgres=# show shared_memory_size_in_huge_pages ;`                                                  postgres=# show `shared_memory_size_in_huge_pages ;`
`shared_memory_size_in_huge_pages`
`----------------------------------`                                                                  `----------------------------------`
 `4225`                                                                                                `8442`

`postgres=# exit`                                                                                     `postgres=# exit`
`nano /etc/tuned/postgres/tuned.conf`                                                                 `nano /etc/tuned/postgres/tuned.conf`

Добавим в раздел sysctl значение hp с запасом в 1%

`vm.nr_hugepages=4265`                                                                                `vm.nr_hugepages=8522`

**Настроим scheduler на обоих машинах:**

Вывод текущего планировщика:

`cat /sys/block/sda/queue/scheduler`
`none mq-deadline kyber [bfq]`

Включим новый планировщик:
`echo "none" | sudo tee /sys/block/sda/queue/scheduler`

Сохранение настроек при перезапуске ОС:
```bash
/etc/default/grub
----------------------
…
GRUB_CMDLINE_LINUX="elevator=none"
…
----------------------
```
Применение изменений в загрузчике:
update-grub2

проверим  планировщик:
```bash
cat /sys/block/sda/queue/scheduler
[none] mq-deadline kyber bfq
```

**Настроим значение rota на обоих машинах:**

1 - есть вращающаяся часть (HDD)
0 - нет вращающейся части (SSD)

Выведем текущие параметры rotate для дисков в системе :
```bash
lsblk -d -o name,rota
NAME ROTA
sda     1
sdb     1
sr0     1
```

Изменим тип дисков:

`echo 0 > /sys/block/sda/queue/rotational`
`echo 0 > /sys/block/sdb/queue/rotational`

Проверим изменения
```bash
lsblk -d -o name,rota
NAME ROTA
sda     0
sdb     0
sr0      1
```

TPS:                                                                                             TPS:
    1. 5350                                                                                       14000
    2. 5320                                                                                       14101
    3. 5290                                                                                       14174
**Avarage tps=  5320                                                                               Avarage tps= 14092**

>Вывод:
>Проанализировав результаты тестирования можно предположить, что ощутимый прирост производительности можно получить только от настройки БД (до 10%), при достаточной производительности системы, остальные настройки сказываются не значительно, в пределах погрешности измерений.