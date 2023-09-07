# Homework-2

## Задание 1

**Создадим ВМ**
```bash
gcloud beta compute --project=quixotic-moment-397713 instances create postgres --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

**Создадим два диска один для pgdata, другой для предполагаемых бэкапов**
```bash
gcloud compute disks create pgdata --size=10GB
gcloud compute disks create backup --size=10GB
```
**Приаттачим диски к машине postgres**
```bash
gcloud compute instances attach-disk postgres --disk=pgdata
gcloud compute instances attach-disk postgres --disk=backup
```
**Подключимся к машине postgres**

``gcloud compute ssh postgres``

**Установим постгрес 15**
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15
```
**Посмотрим состояние инстансов постгреса**
```bash
a_salugin@postgres:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
**Создадим таблицу**
```bash
a_salugin@postgres:~$ sudo su postgres
postgres@postgres:/home/a_salugin$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
Type "help" for help.
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
\q
```
**Установка parted**

`sudo apt install parted`

**Посмотрим добавленные диски**
```bash
a_salugin@postgres:~$  sudo parted -l | grep Error
Error: /dev/sdb: unrecognised disk label
Error: /dev/sdc: unrecognised disk label

a_salugin@postgres:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0     7:0    0 63.5M  1 loop /snap/core20/2015
loop1     7:1    0  349M  1 loop /snap/google-cloud-cli/165
loop2     7:2    0 91.9M  1 loop /snap/lxd/24061
loop3     7:3    0 40.9M  1 loop /snap/snapd/19993
sda       8:0    0   10G  0 disk
├─sda1    8:1    0  9.9G  0 part /
├─sda14   8:14   0    4M  0 part
└─sda15   8:15   0  106M  0 part /boot/efi
sdb       8:16   0   10G  0 disk
sdc       8:32   0   10G  0 disk
```
**Выберем способ разметки GPT для новых дисков sdb и sdc**
```bash
a_salugin@postgres:~$ sudo parted /dev/sdb mklabel gpt
Information: You may need to update /etc/fstab.

a_salugin@postgres:~$ sudo parted /dev/sdc mklabel gpt
Information: You may need to update /etc/fstab.
```
**Создадим разделы на дисках**
```bash
a_salugin@postgres:~$ sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

a_salugin@postgres:~$ sudo parted -a opt /dev/sdc mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.
```
**проверим что разделы создались**
```bash
a_salugin@postgres:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0     7:0    0 63.5M  1 loop /snap/core20/2015
loop1     7:1    0  349M  1 loop /snap/google-cloud-cli/165
loop2     7:2    0 91.9M  1 loop /snap/lxd/24061
loop3     7:3    0 40.9M  1 loop /snap/snapd/19993
sda       8:0    0   10G  0 disk
├─sda1    8:1    0  9.9G  0 part /
├─sda14   8:14   0    4M  0 part
└─sda15   8:15   0  106M  0 part /boot/efi
sdb       8:16   0   10G  0 disk
└─sdb1    8:17   0   10G  0 part
sdc       8:32   0   10G  0 disk
└─sdc1    8:33   0   10G  0 part
```
**Создадим файловую систему**
```bash
sudo mkfs.ext4 -L pg_data_part /dev/sdb1
sudo mkfs.ext4 -L backup /dev/sdc1
```
**Смонтируем файловую систему в соответствующий каталог**
```bash
a_salugin@postgres:~$ sudo mkdir -p /pgdata
a_salugin@postgres:~$ sudo mkdir -p /backup

a_salugin@postgres:~$ sudo mount -o defaults /dev/sdb1 /pgdata
a_salugin@postgres:~$ sudo mount -o defaults /dev/sdc1 /backup
```
**Отредактируем fstab для автоматического монтрования при запуске ос**
```bash
sudo nano /etc/fstab

LABEL=pg_data_part /pgdata ext4 defaults 0 2
LABEL=backup /backup ext4 defaults 0 2
```
**Перезапустим систему и проверим что диски смонтированы**
```bash
a_salugin@postgres:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  2.3G  7.3G  24% /
devtmpfs        977M     0  977M   0% /dev
/dev/loop0       64M   64M     0 100% /snap/core20/2015
/dev/loop1      349M  349M     0 100% /snap/google-cloud-cli/165
/dev/loop2       92M   92M     0 100% /snap/lxd/24061
/dev/loop3       41M   41M     0 100% /snap/snapd/19993
/dev/sda15      105M  6.1M   99M   6% /boot/efi
/dev/sdb1       9.8G   24K  9.3G   1% /pgdata
/dev/sdc1       9.8G   24K  9.3G   1% /backup
```
**Сделаем владельцем новых каталогов пользователя postgres**
```bash
a_salugin@postgres:~$ sudo chown -R postgres:postgres /pgdata
a_salugin@postgres:~$ sudo chown -R postgres:postgres /backup
```
**перенесем данные из стандартного каталога в новый /pgdata**

`a_salugin@postgres:~$ sudo -u postgsre mv /var/lib/postgresql/15 /pgdata`

**Запустим постгрес**
```bash
a_salugin@postgres:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
**Постгрес не видит кластера на привычном месте и ругается. Чтобы исправить эту ситуацию, необходимо указать пг в его конфиге где теперь его кластер**
```bash
nano /etc/postgresql/15/main/postgresql.conf
data_directory = '/pgdata/15/main' 
```
**Запустим кластер и проверим  его статус**
```bash
postgres@postgres:/home/a_salugin$ pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
postgres@postgres:/home/a_salugin$ pg_ctlcluster 15 main status
pg_ctl: server is running (PID: 8332)
/usr/lib/postgresql/15/bin/postgres "-D" "/pgdata/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
```
**После внесения изменений в конфиг, постгрес видит каталог с данными и успешно запускается**

**Проверим созданную ранее таблицу**

```bash
postgres=# \dp
                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 public | test | table |                   |                   |
(1 row)

postgres=# select * from test
postgres-# ;
 c1
----
 1
(1 row)
```

**Таблица на месте, и данные тоже**


## Задание со звездочкой

**Создадим еще одну виртуалку**

```bash
gcloud beta compute --project=quixotic-moment-397713 instances create postgres2 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres2 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
gcloud compute instances attach-disk postgres2 --disk=pgdata
```
**Приаттачить второй раз диск не дает, сначала надо отвязать от первой машины**

```bash
gcloud compute instances detach-disk postgres --disk=pgdata
Updated [https://www.googleapis.com/compute/v1/projects/quixotic-moment-397713/zones/us-central1-a/instances/postgres].
gcloud compute instances attach-disk postgres2 --disk=pgdata
Updated [https://www.googleapis.com/compute/v1/projects/quixotic-moment-397713/zones/us-central1-a/instances/postgres2].
```
**Установим на вторую машину постгрес**
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15
```
**Создадим каталог для монтирования диска с данными первой машины**

`a_salugin@postgres2:~$ sudo mkdir -p /pgdata`

**Cмонтируем диск в каталог /pgdata**

`a_salugin@postgres2:~$ sudo mount -o defaults /dev/sdb1 /pgdata`

**Отредактируем конфиг, чтобы постгрес смотрел в новый каталог pgdata**
```bash
a_salugin@postgres2:~$ sudo -u postres nano /etc/postgresql/15/main/postgresql.conf
data_directory = '/pgdata/15/main'
```
**Перезапустим постгрес чтобы изменения в конфиге вступили в силу**

`a_salugin@postgres2:~$ sudo pg_ctlcluster 15 main restart`

**Проверим на месте ли данные**

```bash
a_salugin@postgres2:~$ sudo -u postgres psql
postgres=# \dp
                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 public | test | table |                   |                   |
(1 row)

postgres=# select * from test
postgres-# ;
 c1
----
 1
(1 row)
```

**Постгрес успешно работает с данными на внешнем диске**