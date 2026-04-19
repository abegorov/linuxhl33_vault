# Веб-портал с централизованным хранилищем секретов в Kubernetes

## Задание

Развернуть веб-приложение через Kubernetes. Централизовать хранение секретов с помощью **Vault** и реализовать автоматическое обновление паролей к БД.

1. Разверните кластер веб-приложения с использованием **Kubernetes**.
2. Разверните кластер **Vault**.
3. Настройте интеграцию **Vault** с базой данных и реализацию динамической выдачи паролей, которые обновляются каждые 2 минуты.
4. Обеспечьте доставку обновлённого пароля в приложение (например, через шаблоны **Consul Template**, **Vault Agent** или переменные окружения).

## Реализация

Проект базируется на предудущем проекте [linuxhl31_kubernetes](https://github.com/abegorov/linuxhl31_kubernetes).

Задание сделано так, чтобы его можно было запустить как в **Vagrant**, так и в **Yandex Cloud**. После запуска происходит развёртывание следующих виртуальных машин отказоустойчивого кластера **kubernetes**:

- **vault-control-01** - узел kubernetes control plane;
- **vault-control-02** - узел kubernetes control plane;
- **vault-control-03** - узел kubernetes control plane;
- **vault-worker-01** - узел kubernetes worker node;
- **vault-worker-02** - узел kubernetes worker node;
- **vault-worker-03** - узел kubernetes worker node.

В независимости от того, как созданы виртуальные машины, для их настройки запускается **Ansible Playbook** [provision.yml](provision.yml) который последовательно запускает следующие роли:

- **wait_connection** - ожидает доступность виртуальных машин (при разворачивании в **yandex cloud**).
- **apt_sources** - настраивает репозитории для пакетного менеджера **apt** (используется [mirror.yandex.ru](https://mirror.yandex.ru)).
- **disable_swap** - отключает использование swap.
- **haproxy** - устанавливает и настраивает **haproxy** на **control plane** для проксирования порта 8443 на 6443 узлы **control plane**, а также 80 и 443 порты на 30080 и 30443 порты **worker** узлов (запускается только при разворачивании в **vagrant**, в **yandex cloud** используется **network load balancer**).
- **keepalived** - устанавливает и настраивает **keepalived** на общий адрес **192.168.56.11** для всех узлов **control plane** (запускается только при разворачивании в **vagrant**).
- **docker_repo** - настраивает зеркало для репозитория **docker.io** для последующей установке **containerd.io**.
- **kubernetes** - поднимает кластер **kubernetes** с помощью **kubeadm**, также устанавливает **flannel**, **gateway api**, **certmanager** и **envoy-gateway-system**.
- **disk_facts** - собирает информацию о дисках и их сигнатурах (с помощью утилит `lsblk` и `wipefs`) на узлах **worker**.
- **disk_label** - разбивает диски и устанавливает на них **GPT Partition Label** для их дальнейшей идентификации на узлах **worker**.
- **mount** - форматрует диск под данные и монтриует его в `/var/lib/longhorn` на узлах **worker**.
- **longhorn** - устанавливает **longhorn**.
- **kubernetes_apply** - применяет дополнительные манифесты в директории [manifests](manifests).

Данные роли настраиваются с помощью переменных, определённых в следующих файлах:

- [group_vars/all/ansible.yml](group_vars/all/ansible.yml) - общие переменные **ansible** для всех узлов;
- [group_vars/all/k8s.yml](group_vars/all/k8s.yml) - адрес и порт **load balancer** для подключения к разворачиваемому кластеру **kubernetes**;
- [group_vars/all/kubernetes.yml](group_vars/all/kubernetes.yml) - настройки кластера **kubernetes** (аргументы для **kubelet**, список узлов **control plane**, сеть и интерфейс для **flannel**);
- [group_vars/control/etcdctl.yml](group_vars/control/etcdctl.yml) - перечень узлов кластера **etcd** для настройки утилиты **etcdctl**;
- [group_vars/control/haproxy.yml](group_vars/control/haproxy.yml) - конфигурация **haproxy**;
- [group_vars/control/keepalived.yml](group_vars/control/keepalived.yml) - конфигурация **keepalived**;
- [group_vars/control/kubernetes.yml](group_vars/control/kubernetes.yml) - параметры **kubeadm** поднятия кластера **kubernetes**, список дополнительных манифестов, которые нужно применить через роль **kubernetes_apply**;
- [group_vars/control/wordpress.yml](group_vars/control/wordpress.yml) - настройки для **wordpress** (генерация паролей для **mariadb**, версии образов, имя домена).
- [group_vars/worker/mount.yml](group_vars/worker/mount.yml) - настройки для ролей **disk_label** и **mount** для форматирования и монтирования `/var/lib/longhorn`.

Для разворачивания **wordpress** были написаны следующие манифесты для кластера **kubernetes** (они применяются через роль **kubernetes_apply**):

- [manifests/certmanager-selfsigned-issuer.yml](manifests/certmanager-selfsigned-issuer.yml) - эмитент для сертификата шлюза **kubernetes**;
- [manifests/certmanager-gateway.yml](anifests/certmanager-gateway.yml) - сертификат для шлюза;
- [manifests/eg-nodeport.yml](manifests/eg-nodeport.yml) - дополнительные настройки шлюза (чтобы он работал через **NodePort** сервис, а не **LoadBalancer**);
- [manifests/gateway.yml](manifests/gateway.yml) - шлюз;
- [manifests/http-to-https-redirect.yml](manifests/http-to-https-redirect.yml) - настройки шлюза для перенаправления **http** на **https**;
- [manifests/wordpress-mariadb-secret.yml](manifests/wordpress-mariadb-secret.yml) - пароли для **mariadb**;
- [manifests/wordpress-mariadb-service-headless.yml](manifests/wordpress-mariadb-service-headless.yml) - headless сервис для **mariadb**;
- [manifests/wordpress-mariadb-service.yml](manifests/wordpress-mariadb-service.yml) - сервис для подключения к **mariadb**;
- [manifests/wordpress-mariadb.yml](manifests/wordpress-mariadb.yml) - разворачивание **mariadb**;
- [manifests/wordpress-angie-config.yml](manifests/wordpress-angie-config.yml) - конфигурация **angie** для **wordpress** (`fastcgi_pass unix:/run/php-fpm.sock;`);
- [manifests/wordpress-config.yml](manifests/wordpress-config.yml) - конфигурация **php-fpm** для **wordpress** (`listen = /run/php-fpm.sock`);
- [manifests/wordpress-pvc.yml](manifests/wordpress-pvc.yml) - claim для общего тома **wordpress**;
- [manifests/wordpress-service.yml](manifests/wordpress-service.yml) - сервис для доступа к **wordpress** через **angie**;
- [manifests/wordpress.yml](manifests/wordpress.yml) - разворачивания **wordpress**;
- [manifests/wordpress-httproute.yml](manifests/wordpress-httproute.yml) - маршрут для **gateway api**.

## Запуск

### Общие требования

1. Необходимо установить **Ansible**.
2. Необходимо установить **kubernetes** модуль для **python** (python3-kubernetes).
3. Для разворачивания манифеста **envoy proxy** также нужен **helm** версии 3.

### Запуск в Yandex Cloud

1. Необходимо установить и настроить утилиту **yc** по инструкции [Начало работы с интерфейсом командной строки](https://yandex.cloud/ru/docs/cli/quickstart).
2. Необходимо установить **Terraform** по инструкции [Начало работы с Terraform](https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart).
3. Необходимо перейти в папку проекта и запустить скрипт [up.sh](up.sh).

### Запуск в Vagrant (VirtualBox)

Необходимо скачать **VagrantBox** для **bento/ubuntu-24.04** версии **202510.26.0** и добавить его в **Vagrant** под именем **bento/ubuntu-24.04/202510.26.0**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/bento/boxes/ubuntu-24.04/versions/202510.26.0/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "bento/ubuntu-24.04/202510.26.0"
rm vagrant.box
```

После этого нужно сделать **vagrant up** в папке проекта.

## Проверка

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.4.9**
- **VirtualBox 7.2.6_SUSE r172322**
- **Ansible 2.20.4**
- **Python 3.13.12**
- **Python client for kubernetes 35.0.0**
- **kubectl v1.35.3**
- **helm v3.20.1**
- **helm-diff v3.15.5**
- **helm-git v1.5.2**
- **Jinja2 3.1.6**
- **Terraform 1.14.8**
