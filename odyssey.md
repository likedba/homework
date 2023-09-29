# Пулер соединений Odyssey

## Оглавление

1. [Почему для PostgreSQL необходим пулер соединений](#Почему-для-PostgreSQL-необходим-пулер-соединений)
2. [Преимущества odyssey](#Преимущества-odyssey)
3. [Установка odyssey](#Установка-odyssey)
    1. [Требования](#Требования)
    2. [Установка](#Установка)
4. [Настройка Odyssey](#Настройка-Odyssey)
    1. [Общие настройки](#Общие-настройки)
    2. [Настройка маршрутов](#Настройка-маршрутов)
        - [Секция listen](#Секция-listen)
        - [Секция storage](#Секция-storage)
        - [Секция database](#Секция-database)
5. [Проверка работы Odyssey](#Проверка-работы-Odyssey)
6. [Тест производительности БД с пулером и без](#Тест-производительности-БД-с-пулером-и-без)
    1. [Без пулера](#Производительность-БД-без-пулера)
    2. [С пулером](#Производительность-БД-с-пулером)
    3. [С пулером на отдельном хосте](#Производительность-БД-с-пулером-на-отдельном-хосте)
7. [Вывод](#Вывод)

### Почему для PostgreSQL необходим пулер соединений

В PostgreSQL на каждого клиента выделяется отдельный бэкенд процесс, который обслуживает запросы этого клиента. Внутри бэкенда существует кэш, в котором кэшируется системный каталог, информация о таблицах, правах, скомпилированные функции, планы запросов и т.д. Так как бэкенд кэширует большое количество различных данных, повторное выполнение запросов в нем же будет быстрее, чем создание нового и его наполнение. При каждом создании нового бэкэнда происходит блокировка proc array в эксклюзивном режиме, что приводит к большой конкуренции и потерям в производительности.
В итоге,чем больше коннектов, тем больше памяти потребляется и меньше производительность БД.

Пулер на стороне приложения решает проблему только частично, т.к. в случае, когда серверов приложений много и каждое выделяет свой пул, соединений к БД становится много со всеми вытекающими последствиями. Также пулер на стороне приложения плохо работает с шардингом.

**Альтернативы пулеру Odyssey:**

*Pgpool-II*
+ универсальный
- тяжеловесный инструмент
- только сессионный пулинг

*PgBouncer*
+ легковесный
+ быстрый пулер
- однопоточный (сложномасштабируемый)
- усложняет диагностику, из-за того, в логах видно, что все клиенты ходят с хоста, на котором стоит пулер. Призванная решать эту проблему встроенная опция application_add_host удваивает количество запросов к БД, что сказывается на производительности. 
- нет возможности ограничить количество подключений для конкретной базы, пользователя
- нет системы возврата ошибок

### Преимущества odyssey

+ многопоточная обработка (линейная масштабируемость по ядрам)
+ детальная настройка пулов
+ трассировка конкретной клиентской сессии
+ смешанный тип пулинга
+ правильная передача ошибок клиенту
+ удобное логирование и отладка
+ продвинутый transaction pooling 
+ совместимость с решениями для мониторинга pgbouncer

### Установка odyssey

#### Требования

- скачивается из репозитория https://github.com/yandex/odyssey
- для компиляции нужен cmake либо gcc/clang
- зависимости: openssl
- для запуска нужен файл конфигурации ./odyssey config_file


#### Установка
 
Odyssey на данный момент можно установить только собрав из исходных кодов с git. Собрав пулер, также необходимо создать необходимые для работы каталоги, linux пользователя, конфигурационный файл и файл лога с соответствующими разрешениями. После чего настроить запуск сервиса и внести изменения в конфигурационный файл.

```bash
zypper install git cmake gcc postgresql15-server-devel
git clone https://github.com/yandex/odyssey.git
cd odyssey/
make local_build
groupadd --system odyssey
useradd --system --shell /sbin/nologin --gid odyssey --home-dir /var/lib/odyssey --no-create-home odyssey
mkdir /run/odyssey
chown -R odyssey:odyssey /run/odyssey
echo "d /run/odyssey 0755 odyssey odyssey - -" > /usr/lib/tmpfiles.d/odyssey.conf
mkdir /var/lib/odyssey
chown -R odyssey:odyssey /var/lib/odyssey
touch /var/log/odyssey.log
chown odyssey:odyssey /var/log/odyssey.log
mkdir /etc/odyssey
cd ~/odyssey/
cp build/sources/odyssey /usr/bin
chmod a+x /usr/bin/odyssey
chown root:root /usr/bin/odyssey
cp -p odyssey.conf /etc/odyssey/odyssey.conf
chmod 644 /etc/odyssey/odyssey.conf
cp scripts/systemd/odyssey.service /usr/lib/systemd/systemsystemctl daemon-reload
```

В случае, если odyssey работает совместно с кластером patroni, то для корректной работы необходимо отредактировать юнит файлы для правильного запуска сервиса.

<details>

  <summary>odyssey.service</summary>

```bash
### Editing /etc/systemd/system/odyssey.service.d/override.conf
### Anything between here and the comment below will become the new contents of the file
[Unit]
After=patroni.service
PartOf=patroni.service
### Lines below this comment will be discarded
### /usr/lib/systemd/system/odyssey.service
# [Unit]
# Description=Advanced multi-threaded PostgreSQL connection pooler and request router
# After=network.target
#
# [Service]
# User=odyssey
# Group=odyssey
# Type=simple
# ExecStart=/usr/bin/odyssey /etc/odyssey/odyssey.conf
# LimitNOFILE=100000
# LimitNPROC=100000
#
# [Install]
# WantedBy=multi-user.target
```

</details>

<details>

  <summary>patroni.service</summary>

```bash
[Unit]
Description=HA Postgresql Cluster
After=syslog.target network.target
Requires=odyssey.service
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
```

</details>

systemctl daemon-reload

### Настройка Odyssey
#### Общие настройки

<details>

  <summary>пример конфигурационного файла odyssey.conf</summary>

```yaml
daemonize no
pid_file "/var/run/odyssey/odyssey.pid"
unix_socket_dir "/var/run/postgresql"
unix_socket_mode "0644"
locks_dir "/var/run/odyssey"
graceful_die_on_errors no
enable_online_restart no
bindwith_reuseport no
log_file "/var/log/odyssey.log"
log_format "%p %t %l [%i %s] (%c) %m\n"
log_to_stdout yes
log_syslog no
log_syslog_ident "odyssey"
log_syslog_facility "daemon"
log_debug no
log_config no
log_session no
log_query no
log_stats no
stats_interval 60
log_general_stats_prom no
log_route_stats_prom no
workers 1
resolvers 1
readahead 8192
cache_coroutine 0
coroutine_stack_size 8
nodelay yes
keepalive 15
keepalive_keep_interval 75
keepalive_probes 9
keepalive_usr_timeout 0
listen {
        host "*"
        port 6432
        backlog 128
        compression no
}
storage "postgres_server" {
        type "remote"
        host "10.1.69.170"
        port 5432
}
database "postgres" {
        user default {
                authentication "none"
                storage "postgres_server"
                pool "session"
                pool_size 10
                client_max 100
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                application_name_add_host yes
                log_debug no
        }
        user "postgres" {
                authentication "none"
                storage "postgres_server"
                pool "session"
                pool_size 20
                client_max 50
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                application_name_add_host yes
                log_query no
        }
}
database "pgbench" {
        user default {
                authentication "none"
                storage "postgres_server"
                pool "transaction"
                pool_size 80
                client_max 1800
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                application_name_add_host yes
                log_debug no
        }
        user "pgbench" {
                authentication "scram-sha-256"
                password "<password>"
                storage "postgres_server"
                pool "session"
                pool_size 80
                client_max 1800
                pool_discard no
                pool_cancel yes
                pool_rollback yes
                client_fwd_error yes
                application_name_add_host yes
                log_query no
        }
}
storage "local" {
        type "local"
}
database "console" {
        user default {
                authentication "none"
                pool "session"
                storage "local"
        }
}
```

</details>

Odyssey позволяет гибко настраивать логирование, а также поддерживает мониторинг prometheus:

log_file "/var/log/odyssey.log" \
файл лога должен существовать и его владельцем должен быть пользователь odyssey

log_format - формат вывода лога \
log_debug - логирование отладочной информации \
log_config - записывает содержимое конфигурационного файла при запуске \
log_session - запись статистической информации о сессиях \
log_query - запись всех клиентских запросов \
log_stats - периодическая запись статистики \
stats_interval - интервал записи статистики

promhttp_server_port - порт для сбора метрик сервером prometheus \
log_general_stats_prom, log_route_stats_prom - отвечают за вывод лога в формате prometheus


#### Настройка маршрутов

##### Секция listen

Данная секция отвечает за подключение к Odyssey.

```yaml
listen {
        host "*"
        port 6432
        backlog 128
        compression no
}
```

##### Секция storage
Сторэджи, для которых настраиваются правила подключения бывают двух типов:

1. remote

Здесь указываются данные для подключения к инстансу PostgreSQL. В конфиге может быть указано несколько внешних хранилищ.

```yaml
Storage "postgres server" {
    type "remote"
    host "127.0.0.1"
    port "5432"
    tls "disable"
}
```

2. local

Данный тип хранилища необходим для описания правил подключения к консоли Odyssey. В нем существует только одна БД "console".

```yaml
storage "local" {
        type "local"
}
database "console" {
        user default {
                authentication "none"
                pool "session"
                storage "local"
        }
}
```
После создания правила подключения к консоли odyssey появляется возможность смотреть статистику:
`psql -h 127.0.0.1 -p 6432 -d console`

```sql
show stats;
show pools;
show clients;
show lists;
show servers;
show databases;
```
##### Секция database

Odyssey позволяет гибко настраивать пулы с привязкой к конкретной базе и пользователю, а так же настраивать для них различные методы аутентификации: по паролю или сертификату (clear_text, md5, scram-sha-256, certificate), по паролю из внешней базы PostgreSQL.
Настройка пулов осуществляется на основе правил - определяется бд и пользователь, который к ней относится. Если подключается определенный пользователь в определенную базу, то в конфиге ищется для него соответствующее правило. Если такое правило присутствует, то применяются ассоциированные с ним настройки пула. Если подключается пользователь, для которого нет отдельного правила, то он подпадает под правило "default", для которого также можно установить определенные настройки пула, либо authentification "block" , который заблокирует доступ для такого пользователя.

```yaml
database "test" {
    user "test" {
        storage "postgres_server"
        authentication "none"
        client_max 100
        pool "transaction"
        pool_size 10
        pool_cancel yes
        pool_rollback yes
    }
    user "default" {
        authentication "block"
    }
}
```

В этой секции можно выставлять лимиты для определенных пользователей, включать или отключать cancel, rollback, настраивать тип пулинга, тип аутентификации, сертификаты и т.д. 
Также по аналогии с пользователем "default", можно описать правило подключения к database "default", которое будет действовать при подключении пользователей к не описанной в конфигурационном файле БД. В данном примере, такие пользователи будут заблокированы:

```yaml
database "default" {
    user "default" {
        authentication "block"
    }
}
```

### Проверка работы Odyssey

Запустим нагрузку





## Тест производительности БД с пулером и без

В тестах участвуют 2 хоста:

host1 (5 CPU, 35 RAM) \
host2 (12 CPU, 63 RAM)

## Тест 1 - Производительность БД без пулера

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

## Тест 2 - Производительность БД с пулером

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

## Тест 3 - Производительность БД с пулером на отдельном хосте

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

![результат теста](/images/postgres_clients_test.png)


## Вывод

#### Значительное увеличении коннектов негативно влияет на производительность. При увеличении клиентов со 100 до 1000 производительность падает до 2 раз.