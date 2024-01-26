```bash
MINICUBE

1. Создадим виртуалку
gcloud beta compute --project=quixotic-moment-397713 instances create postgres1 --zone=us-central1-a --machine-type=e2-small --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=812144456828-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-ssd --boot-disk-device-name=node1 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

2. Установим докер
ubuntu@postgres1:~$ sudo apt-get update
ubuntu@postgres1:~$ sudo install -m 0755 -d /etc/apt/keyrings
ubuntu@postgres1:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
ubuntu@postgres1:~$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
ubuntu@postgres1:~$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
ubuntu@postgres1:~$ sudo apt-get update
ubuntu@postgres1:~$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose

3. Установим minicube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

4. Запускаем minicube
ubuntu@postgres1:~$ sudo usermod -aG docker $USER && newgrp docker
ubuntu@postgres1:~$ minikube start
* minikube v1.31.2 on Ubuntu 22.04 (amd64)
* Automatically selected the docker driver. Other choices: none, ssh
* Using Docker driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
* Downloading Kubernetes v1.27.4 preload ...
    > preloaded-images-k8s-v18-v1...:  393.21 MiB / 393.21 MiB  100.00% 28.21 M
    > gcr.io/k8s-minikube/kicbase...:  447.61 MiB / 447.62 MiB  100.00% 29.47 M
* Creating docker container (CPUs=2, Memory=2200MB) ...
* Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: storage-provisioner, default-storageclass
* kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default


5.  Чтобы проверить доступность кластера извне, запускаем на виртуалке с миникубом прокси чтобы ресурсы стали доступны извне, указываем слушать со всех адресов

ubuntu@postgres1:~$ minikube kubectl -- proxy --accept-hosts='*' --address='0.0.0.0' --disable-filter=true &
[1] 57788
ubuntu@postgres1:~$ W1006 13:56:36.097285   57796 proxy.go:175] Request filter disabled, your proxy is vulnerable to XSRF attacks, please be cautious
Starting to serve on [::]:8001
после этого открывается json с ресурсами апи

6. Забираем с виртуалки конфиг кластера, для использования сертификатов, копируем на локальный комп папочку ~/.minikube 

ubuntu@postgres1:~$ cat ~/.kube/config

7. запускаем OpenLens и добавляем в него наш кластер через Add Clusters from Kubeconfig - вставляем скопированный текст конфига, заменяем в нём ip на внешний, а также прописываем пути к сертификатам - туда куда мы скачали каталог ./minikube, например:


...
server: http://158.160.78.210:8001
...
certificate-authority: /mnt/dev/sdb1/Education/PgAdvanced/otus-pg-adv/lesson_9/.minikube/ca.crt
...
client-certificate: /mnt/dev/sdb1/Education/PgAdvanced/otus-pg-adv/lesson_9/.minikube/profiles/minikube/client.crt
client-key: /mnt/dev/sdb1/Education/PgAdvanced/otus-pg-adv/lesson_9/.minikube/profiles/minikube/client.key
...


кластер подключен в OpenLens:

Minikube in OpenLens

8. возьмем postgres.yaml из лекции и применим в миникубе

ubuntu@postgres1:~$ minikube kubectl -- apply -f postgres.yaml
service/postgres created
statefulset.apps/postgres-statefulset created

9. проверяем локальное подключение, для этого установим на виртуалке клиент постгреса

ubuntu@postgres1:~$ sudo apt install postgresql-client
ubuntu@postgres1:~$ minikube service list
|-------------|------------|--------------|---------------------------|
|  NAMESPACE  |    NAME    | TARGET PORT  |            URL            |
|-------------|------------|--------------|---------------------------|
| default     | kubernetes | No node port |                           |
| default     | postgres   |         5432 | http://192.168.32.2:30545 |
| kube-system | kube-dns   | No node port |                           |
|-------------|------------|--------------|---------------------------|
ubuntu@postgres1:~$ psql -h 192.168.32.2 -p 30545 -U myuser -d myapp

10. для подкюлчения снаружи сделаем порт-форвардинг, не будем закрывать консоль

ubuntu@postgres1:~$ minikube kubectl -- port-forward --address 0.0.0.0 service/postgres 8432:5432
Forwarding from 0.0.0.0:8432 -> 5432


11. пробуем подкюллчиться извне

подключение происходит успешно
```
