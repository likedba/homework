

# Пулер соединений Odyssey для PostgreSQL

### Для чего нужен \

В PostgreSQL на каждого клиента выделяется отдельный бэкенд процесс, который обслуживает запросы этого клиента. Внутри бэкенда существует кэш, в котором кэшируется системный каталог, информация о таблицах, правах, скомпилированные функции, планы запросов и т.д. При каждом создании нового бэкэнда происходит блокировка proc array в эксклюзивном режиме, что также приводит к потерям в производительности.
Так как бэкенд кэширует большое количество различных данных, повторное выполнение запросов в нем же будет быстрее, чем создание нового и его наполнение.
В итоге,чем больше коннектов, тем больше памяти потребляется и меньше производительность БД.

Пулер на стороне приложения решает проблему только частично, т.к. в случае, когда серверов приложений много и каждое выделяет свой пул, соединений к БД становится много со всеми вытекающими последствиями. Также пулер на стороне приложения плохо работает с шардингом.

Альтернативы пулеру Odyssey:

Pgpool-II 
+ универсальный
- тяжеловесный инструмент
- только сессионный пулинг

PgBouncer
+ легковесный
+ быстрый пулер
- однопоточный (сложномасштабируемый)
- усложняет диагностику, из-за того, в логах видно, что все клиенты ходят с хоста, на котором стоит пулер. Призванная решать эту проблему встроенная опция application_add_host удваивает количество запросов к БД, что сказывается на производительности. 
- нет возможности ограничить количество подключений для конкретной базы, пользователя
- нет системы возврата ошибок

### Преимущества odyssey:

+ многопоточная обработка (линейная масштабируемость по ядрам)
+ детальная настройка пулов
+ трассировка конкретной клиентской сессии
+ смешанный тип пулинга
+ правильная передача ошибок клиенту
+ удобное логирование и отладка
+ продвинутый transaction pooling 
+ совместимость с решениями для мониторинга pgbouncer

### Установка odyssey

- Скачивается из репозитория https://github.com/yandex/odyssey
- для компиляции нужен cmake либо gcc/clang
- зависимость только openssl
- для запуска нужен файл конфигурации ./odyssey <config file>

Поддерживаемые платформы: x86, x86_64

#### Пример установки:
 
Odyssey на данный момент можно установить только собрав из исходных кодов с git. Собрав пулер, также необходимо создать необходимые для работы каталоги, linux пользователя, конфигурационный файл и файл лога с соответствующими разрешениями. После чего настроить запуск сервиса и внести изменения в конфигурационный файл.


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

В случае, если odyssey работает в кластере patroni, то для корректной работы необходимо отредактировать юнит файл так, чтобы odyssey всегда запускался с запуском кластера.

Настройка запуска сервиса:
Для корректной работы Odyssey совместно с Patroni необходимо настроить запуск сервиса.


systemctl edit odyssey.service
----------------------------
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
---------------------------------

nano /etc/systemd/system/patroni.service
----------------------------------
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
-------------------------------------
systemctl daemon-reload











Тест производительности БД с пулером и без

В тестах участвуют 2 хоста:

vmbps-sbp-db (5 CPU, 35 RAM)
vmbps-sbp-db (12 CPU, 63 RAM)

## Тест 1 - Влияние увеличения коннектов к БД без пулера на производительность

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

## Тест 1 - Влияние увеличения коннектов к БД c пулером на производительность.
### Пулер на том же хосте.

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

## Тест 1 - Влияние увеличения коннектов к БД c пулерjv на производительность.
### Пулер на отдельном хосте.

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


## Вывод:

#### Значительное увеличении коннектов негативно влияет на производительность. При увеличении клиентов со 100 до 1000 производительность падает до 2 раз.


client_fwd_error off      failed to connect to remote server
client_fwd_error on       database "test" doesn't exist


Настройка маршрутов:

Сторэджи бывают только двух типов. 
1. remote - бд постгреса
2. local - для доступа к консоли odyssey,  откуда также можно получить статистику по работе пулера

Storage "postgres server" {
    type "remote"
    host "127.0.0.1"
    port "5432"
    tls "disable"
}


Настройка пулов осуществляется на основе правил - определяем бд и пользователя, который к ней относится. Если приходит определенный пользователь в определенную базу, то в конфиге ищется для него соответствующее правило, если это правило присутствует, то применяются ассоциированные с ним настройки пула. Если приходит пользователь, для которого нет отдельного правила, то он подпадает под правило "default", которое тоже имеет определенные настройки пула, либо authentification "block" , который заблокирует доступ для такого пользователя.

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

Здесь можно выставлять лимиты для определенных пользователей, включать или отключать cancel, rollback, настраивать тип пулинга, тип аутентификации, сертификаты. 
Также для database, по аналогии с user, можно выставить значение default, тогда пользователи которые захотят подключиться к не описанной в конфиге базе, например, будут заблокированы.

database "default" {
    user "default" {
        authentication "block"
    }
}

