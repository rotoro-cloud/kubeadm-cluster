Here infrastructure template for kubeadm task.
Uses Vagrant and ubuntu.

# Deploy multi-masters HA cluster with `kubeadm`, `vagrant` and `ubuntu`.

Скрипты из этого репозитория облегчают процесс установления кластера производственного уровня.
#
Итак, в курсе мы выяснили, что самый простой сетап с высокой доступностью состоит из 6 элементов:

- балансировщика
- 3 мастер-нод
- 2 рабочих узлов

### Склонируй этот репо

В корне лежит `Vagrantfile`.

По умолчанию он настроен на создание 2 мастеров, 1 воркера и 1 лоадбаленсера.

```
git clone https://github.com/rotoro-cloud/kubeadm-cluster.git
cd kubeadm-cluster
vagrant up
```

Поднимутся VMs. Их адреса:
- 192.168.66.1X - для мастеров
- 192.168.66.2X - для рабочих
- 192.168.66.30 - для балансировщика

Т.е. `controlplane01` будет 192.168.66.11, `controlplane02` будет 192.168.66.12 и т.д.

Но сачала давай познакомимся с окружением

### Окружение
Ты можешь подключиться прямо к виртуальной машине с помощью
```
vagrant ssh node01
```
Или сделай команду:
```
vagrant ssh-config
```
Это покажет тебе адреса VMs, их порты ssh и пути к ключам, чтобы подключиться к ним из твоего любимого ssh-клиента.

### Балансировщик
Для сетапа высокой доступности необходимый элемент, без него не будет полноценной `HA`.
В этом сетапе мы используем `HAproxy`, давай ее настроим.

##### Выполняется в VM `lb`
#
```
sudo apt update
sudo apt install -y haproxy
```
Далее внесем правки в `/etc/haproxy/haproxy.cfg`, добавив в конц следующее:
```
frontend kubernetes-frontend
    bind 192.168.66.30:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server controlplane01 192.168.66.11:6443 check fall 3 rise 2
    server controlplane02 192.168.66.12:6443 check fall 3 rise 2
    server controlplane03 192.168.66.13:6443 check fall 3 rise 2
```
Теперь балансировщик будет пересылать трафик на один из трех мастеров. И он достаточно умный не послылать трафик, если мастер в данный момент недоступен. 
```
sudo systemctl restart haproxy
```
Мы закончили с балансировщиком, он нам больше не понадобится.

##### Выполняется во всех VMs-нодах, и мастерах и воркерах
#
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic)
```
lsmod | grep br_netfilter
```
Если пусто, то:
```
sudo modprobe br_netfilter
```
После этого
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
Далее подготовка среды выполнения контейнера.
[https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

У меня `docker`, поэтому:
```
wget get.docker.com
bash index.html
sudo systemctl enable docker
sudo usermod -aG docker vagrant
```
Теперь поставим `kubeadm`, `kubelet` и `kubectl`
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

Добавим нужные утилиты, хотя `docker` уже их должен был добавить:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Добавим ключи репозитория `google` и сам репо:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Теперь поставим наши утилиты:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
##### Выполняется на самом первом мастере
#
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node)
Проинициализируем мастер
```
sudo kubeadm init --control-plane-endpoint="192.168.66.30" --upload-certs --apiserver-advertise-address=192.168.66.11 --pod-network-cidr=10.244.0.0/16
```
Сразу сделаем себе доступ для `kubectl`:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
##### Выполняется на других мастерах
#
Присоединим остальные матера командой из вывода `kubeadm`
```
sudo kubeadm join 192.168.66.30:6443 --apiserver-advertise-address=192.168.66.12...
```
##### Выполняется на воркерах
#
Присоединим остальные матера командой из вывода `kubeadm`
```
sudo kubeadm join 192.168.66.30:6443...
```

##### Последний шаг
#
Развернем сетевой плагин:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
