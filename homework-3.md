# Homework-3

**Создадим ВМ**
```bash
gcloud beta compute --project=quixotic-moment-397713 instances create postgres --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
**Подключимся к машине postgres**

``gcloud compute ssh postgres``

**Установим docker**
a_salugin@postgres:~$ sudo apt update
a_salugin@postgres:~$ sudo apt install docker docker.io
a_salugin@postgres:~$ sudo systemctl enable docker
a_salugin@postgres:~$ sudo systemctl start docker

создадим контейнер с сервером в /var/lib/postgres

sudo docker run -d \
        --name my-postgres \
        -p 5432:5432 \
        -v ~/apps/postgres:/var/lib/postgres \
        -e POSTGRES_PASSWORD=S3cret \
        -e POSTGRES_USER=citizix_user \
        -e POSTGRES_DB=citizix_db \
         postgres:14-alpine