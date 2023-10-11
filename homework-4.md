
# Настройка и нагрузочное тестирование БД

**Тестируемые машины:** \
host1 \
host2

**Конфигурация host1:** \
5 ядер \
35 Гб ОЗУ \
SSD массив

**Конфигурация host2:** \
12 ядер \
63 Гб ОЗУ \
SSD массив

**Нагрузка:** \
pgbench -U pgbench -c 80 -T 600 -P 60 pgbench

## Тест №1
Измерим производительность БД со стандартными настройками ОС и БД

![вывод pgbench](/images/pgbench.png)

| host1: | host2: |
| :------: | :------: |
| Стандартные настройки postgresql | Стандартные настройки postgresql |
| Профиль ОС balanced | Профиль ОС balanced |
| | |
| TPS: | TPS: |
| 5263 | 13076 |
| 5286 | 12913 |
| 5302 | 12971 |
| **Avarage tps=  5284** | **Avarage tps= 12987** |

## Тест №2
> **Измерим производительность БД с измененными настройками ОС и стандартными настройками БД**


Создадим профиль "postgres" утилиты tuned:

```bash
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
```

Применим профиль postgres на обоих машинах

`tuned-adm profile postgres`

Проверим производительность БД после внесенных изменений

| host1: | host2: |
| :------: | :------: |
| Стандартные настройки postgresql | Стандартные настройки postgresql |
| Профиль ОС postgres | Профиль ОС postgres |
| | |
| TPS: | TPS: |
| 5348 | 12727 |
| 5300 | 12847 |
| 5324 | 12549 |
| **Avarage tps= 5324** | **Avarage tps= 12708** |

## Тест №3
> **Измерим производительность БД с измененными настройками ОС и БД**

**Отредактируем настройки postgresql на обоих хостах:**
| host1: | host2: | info |
| :------ | :------ | :------ |
| data_directory = '/pgdata/' | data_directory = '/pgdata/' |  |
| hba_file = '/pgdata/pg_hba.conf' | hba_file = '/pgdata/pg_hba.conf' |  |
| ident_file = '/pgdata/pg_ident.conf' | ident_file = '/pgdata/pg_ident.conf' |  |
| listen_addresses = '*' | listen_addresses = '*' |  |
| max_connections = 1400 | max_connections = 1400 | В этом проекте пока использование пулера не представляется возможным |
| *shared_buffers = 8192MB* | *shared_buffers = 16384MB* |  |
| work_mem = 32MB | work_mem = 32MB |  |
| *maintenance_work_mem = 420MB* | *maintenance_work_mem = 520MB* |  |
| effective_io_concurrency = 200 | effective_io_concurrency = 200 |  |
| maintenance_io_concurrency = 200 | maintenance_io_concurrency = 200 |  |
| *max_worker_processes = 5* | *max_worker_processes = 12* |  |
| *max_parallel_workers_per_gather = 3* | *max_parallel_workers_per_gather = 6* |  |
| *max_parallel_maintenance_workers = 3* | *max_parallel_maintenance_workers = 6* |  |
| *max_parallel_workers = 5* | *max_parallel_workers = 12* |  |
| wal_compression = on | wal_compression = on | поскольку узким местом большинства серверов является io а не cpu |
| wal_init_zero = off | wal_init_zero = off | более эффективные операции с вал в файловых системах COW |
| wal_recycle = off | wal_recycle = off | на виртуальных машинах быстрее создание новых вал |
| wal_buffers = 64MB | wal_buffers = 64MB |  |
| max_wal_size = 5GB | max_wal_size = 5GB |  |
| min_wal_size = 2500MB | min_wal_size = 2500MB |  |
| seq_page_cost = 1.0 | seq_page_cost = 1.0 |  |
| random_page_cost = 1.25 | random_page_cost = 1.25 |  |
| cpu_tuple_cost = 0.03 | cpu_tuple_cost = 0.03 |  |
| *effective_cache_size = 23GB* | *effective_cache_size = 40GB* |  |
| shared_preload_libraries = 'pg_stat_statements,pg_prewarm' | shared_preload_libraries = 'pg_stat_statements,pg_prewarm' |  |

Измерим производительность после изменения и применения настроек:

| host1: | host2: |
| :------: | :------: |
| Отредактированные настройки postgresql | Отредактированные настройки postgresql |
| Профиль ОС postgres | Профиль ОС postgres |
| | |
| TPS: | TPS: |
| 5223 | 13996 |
| 5314 | 14103 |
| 5253 | 14007 |
| **Avarage tps= 5263** | **Avarage tps= 14035** |

## Тест №4

> **Измерим производительность БД с измененными настройками ОС и БД + настроенные huge pages, scheduler и параметром rota для диска с pgdata**

**Настроим huge pages:**

host1:
```bash
psql
postgres=# show shared_memory_size_in_huge_pages ;
shared_memory_size_in_huge_pages
----------------------------------
 4225

postgres=# exit
nano /etc/tuned/postgres/tuned.conf

#Добавим в раздел sysctl значение hp с запасом в 1%

vm.nr_hugepages=4265
```

host2:
```bash
psql
postgres=# show shared_memory_size_in_huge_pages
shared_memory_size_in_huge_pages 
----------------------------------
 8442
postgres=# exit
nano /etc/tuned/postgres/tuned.conf

#Добавим в раздел sysctl значение hp с запасом в 1%

vm.nr_hugepages=8522
```

**Настроим scheduler на обоих машинах:**

Вывод текущего планировщика:

`cat /sys/block/sda/queue/scheduler` \
`none mq-deadline kyber [bfq]`

Включим новый планировщик: \
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
Применение изменений в загрузчике: \
`update-grub2`

проверим  планировщик:
```bash
cat /sys/block/sda/queue/scheduler
[none] mq-deadline kyber bfq
```

**Настроим значение rota на обоих машинах:** 

1 - есть вращающаяся часть (HDD) \
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

`echo 0 > /sys/block/sda/queue/rotational` \
`echo 0 > /sys/block/sdb/queue/rotational`

Проверим изменения
```bash
lsblk -d -o name,rota
NAME ROTA
sda     0
sdb     0
sr0     1
```
| host1: | host2: |
| :------: | :------: |
| Отредактированные настройки postgresql | Отредактированные настройки postgresql |
| Профиль ОС postgres + доп | Профиль ОС postgres + доп |
| | |
| TPS: | TPS: |
| 5350 | 14000 |
| 5320 | 14101 |
| 5290 | 14174 |
| **Avarage tps= 5320** | **Avarage tps= 14092** |

## Тест №5

Выполним дополнительный тест для представления, на сколько может повыситься производительность если отключить синхронный коммит в ущерб возможной потери данных при сбое.

| host1: | host2: |
| :------: | :------: |
| Отредактированные настройки postgresql | Отредактированные настройки postgresql |
| Профиль ОС postgres + доп | Профиль ОС postgres + доп |
| | |
| TPS: | TPS: |
| 6343 | 17249 |
| 6524 | 17266 |
| 6489 | 17473 |
| **Avarage tps= 6452** | **Avarage tps= 17329** |

Прирост производительности для host1 составил 21%, для host2 - 23%

## Тест №6

Проведем тест для понимания насколько влияет увеничение количества коннектов к БД на производительность.

| clients | host1 | host2 |
| :------: | :------: | :------: |
| 100 | 5279 | 14471 |
| 200 | 5000 | 13175 |
| 300 | 4539 | 11689 |
| 400 | 4549 | 10670 |
| 500 | 4555 | 10058 |
| 600 | 4618 | 9369 |
| 700 | 4477 | 8928 |
| 800 | 4360 | 8529 |
| 900 | 4249 | 8032 |
| 1000 | 4011 | 7672 |
| 1100 | 3203 | 6450 |
| 1200 | 3090 | 6135 |
| 1300 | 2986 | 5932 |
| 1400 | 2868 | 5706 |
| 1500 | 2759 | 5478 |
| 1600 | 2668 | 5259 |
| 1700 | 2597 | 5121 |
| 1800 | 2502 | 4884 |

![результат теста](/images/postgres_clients_test.png)

## Тест №7

Влияние увеличения количества коннектов к БД, запущенных через пулер на производительность.

| clients | host1 | host2 |
| :------: | :------: | :------: |
| 100 | 4459 | 9223 |
| 200 | 5436 | 9345 |
| 300 | 5443 | 9519 |
| 400 | 5414 | 9643 |
| 500 | 5414 | 9576 |
| 600 | 5400 | 9804 |
| 700 | 5319 | 9760 |
| 800 | 5322 | 9682 |
| 900 | 5292 | 9777 |
| 1000 | 5239 | 9671 |
| 1100 | 4878 | 9730 |
| 1200 | 4982 | 9680 |
| 1300 | 4982 | 9694 |
| 1400 | 4968 | 9601 |
| 1500 | 4966 | 9596 |
| 1600 | 4901 | 9634 |
| 1700 | 4830 | 9589 |
| 1800 | 4788 | 9462 |


![результат теста](/images/postgres_clients_test.png)

## Тест №7

Влияние увеличения количества коннектов к БД, запущенных через пулер установленный на другом хосте на производительность.

| clients | host1 | host2 |
| :------: | :------: | :------: |
| 100 | 8933 | 11416 |
| 200 | 9030 | 11989 |
| 300 | 9074 | 11873 |
| 400 | 9030 | 12027 |
| 500 | 8938 | 11933 |
| 600 | 9169 | 11927 |
| 700 | 9188 | 11692 |
| 800 | 8970 | 11499 |
| 900 | 8881 | 11442 |
| 1000 | 8835 | 11033 |
| 1100 | 8772 | 11172 |
| 1200 | 9054 | 10888 |
| 1300 | 8938 | 10788 |
| 1400 | 8903 | 10316 |
| 1500 | 8952 | 10497 |
| 1600 | 8614 | 10031 |
| 1700 | 8601 | 10145 |
| 1800 | 8642 | 10076 |

![результат теста](/images/)


## Вывод:
#### Проанализировав результаты тестирования можно утверждать, что ощутимый прирост производительности был получен только от настройки БД (до 10%) и только на хосте с большей производительностью. Остальные настройки влияют не столь значительно (либо раскрываются в другом профиле нагрузки). В данном случае настройки ядра ОС дали прирост производительности, сравнимый с погрешностью измерения (учитывая, что pgbench запускался с того же хоста, на котором располагалась БД).
#### В случаях, когда потеря незначительной части данных в результате сбоя не несет рисков, возможно отключить синхронный коммит, что даст значительный прирост в производительности.
#### Значительное увеличении коннектов негативно влияет на производительность. При увеличении клиентов со 100 до 1000 производительность падает до 2 раз.