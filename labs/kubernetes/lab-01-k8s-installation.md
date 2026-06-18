# 🧪 Lab 1: Установка Kubernetes 1.28 с kubeadm на Rocky Linux 10 (Offline Friendly)

## 📋 Обзор

В данной лабораторной работе разворачивается Kubernetes-кластер высокой доступности (HA) из 3 Control Plane и 2 Worker узлов.

Установка Kubernetes выполняется без использования репозиториев Kubernetes и без установки пакетов из внешних источников.

Такой подход выбран потому что:

* не зависит от yum/dnf репозиториев;
* подходит для работы в РФ;
* позволяет использовать строго определенную версию Kubernetes;
* подходит для полностью изолированных сетей;
* позволяет развернуть кластер даже при отсутствии доступа в Интернет.

---

## 🎯 Цели

После выполнения лабораторной работы необходимо:

* установить containerd;
* установить kubeadm;
* установить kubelet;
* установить kubectl;
* инициализировать Control Plane;
* добавить дополнительные Master узлы;
* добавить Worker узлы;
* проверить работоспособность кластера.

---

# 📊 Схема кластера

| Узел          | IP             | Роль          |
| ------------- | -------------- | ------------- |
| k8s-master-01 | 192.168.77.181 | Control Plane |
| k8s-master-02 | 192.168.77.182 | Control Plane |
| k8s-master-03 | 192.168.77.183 | Control Plane |
| k8s-worker-01 | 192.168.77.186 | Worker        |
| k8s-worker-02 | 192.168.77.187 | Worker        |

---

# 📦 Подготовка файлов

Перед началом работы необходимо подготовить каталог со всеми необходимыми файлами.

Структура:

```text
k8s-lab/
├── files/
│   ├── kubeadm
│   ├── kubelet
│   ├── kubectl
│   ├── kubelet.service
│   ├── 10-kubeadm.conf
│   ├── containerd.io.rpm
│   └── kubernetes-v1.28.tar
│
├── install-k8s.sh
└── README.md
```

---

## Содержимое файлов

### kubelet.service

```ini
[Unit]
Description=kubelet
Documentation=https://kubernetes.io/docs/
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 10-kubeadm.conf

```ini
[Service]
Environment="KUBELET_EXTRA_ARGS="
```

---

# 🖥 Подготовка узлов

На всех узлах:

```bash
sudo dnf update -y

sudo dnf install -y \
vim \
curl \
wget \
git \
htop \
tar \
bash-completion \
net-tools
```

---

# 🔧 Отключение Swap

На всех узлах:

```bash
sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Проверка:

```bash
free -h
```

Swap должен быть равен 0.

---

# 🔧 Загрузка модулей ядра

На всех узлах:

```bash
sudo tee /etc/modules-load.d/kubernetes.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Проверка:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

---

# 🔧 Настройка sysctl

На всех узлах:

```bash
sudo tee /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

Проверка:

```bash
sysctl net.ipv4.ip_forward
```

Должно быть:

```text
net.ipv4.ip_forward = 1
```

---

# 📦 Установка containerd

Установка производится из локального RPM пакета.

```bash
sudo dnf install -y ./files/containerd.io.rpm
```

Создание конфигурации:

```bash
sudo mkdir -p /etc/containerd

containerd config default | \
sudo tee /etc/containerd/config.toml
```

Включение systemd cgroups:

```bash
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```

Проверка:

```bash
grep SystemdCgroup \
/etc/containerd/config.toml
```

Должно быть:

```text
SystemdCgroup = true
```

Запуск:

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
```

---

# ☸️ Установка Kubernetes

Установка выполняется из заранее подготовленных бинарных файлов.

```bash
sudo install files/kubeadm /usr/local/bin/
sudo install files/kubelet /usr/local/bin/
sudo install files/kubectl /usr/local/bin/
```

Назначение прав:

```bash
sudo chmod +x /usr/local/bin/kube*
```

Проверка:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

# ⚙️ Настройка systemd для kubelet

Установка сервисных файлов:

```bash
sudo cp files/kubelet.service \
/usr/lib/systemd/system/

sudo mkdir -p \
/usr/lib/systemd/system/kubelet.service.d

sudo cp files/10-kubeadm.conf \
/usr/lib/systemd/system/kubelet.service.d/
```

Активация:

```bash
sudo systemctl daemon-reload

sudo systemctl enable kubelet

sudo systemctl start kubelet
```

Проверка:

```bash
systemctl status kubelet
```

До выполнения kubeadm init сервис может находиться в состоянии failed или activating — это нормально.

---

# 📦 Загрузка Kubernetes образов

Для полностью автономной установки необходимо заранее импортировать контейнерные образы.

Импорт:

```bash
sudo ctr -n k8s.io images import \
files/kubernetes-v1.28.tar
```

Проверка:

```bash
sudo ctr -n k8s.io images ls
```

В списке должны присутствовать:

* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* etcd
* coredns
* pause

---

# 🚀 Инициализация первого Master

Только на k8s-master-01:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.77.181 \
  --control-plane-endpoint=192.168.77.181:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Сохранить выведенные команды подключения.

---

# 🔑 Настройка kubectl

На первом мастере:

```bash
mkdir -p ~/.kube

sudo cp \
/etc/kubernetes/admin.conf \
~/.kube/config

sudo chown \
$(id -u):$(id -g) \
~/.kube/config
```

Проверка:

```bash
kubectl get nodes
```

---

# ➕ Добавление Master узлов

Получить certificate key:

```bash
sudo kubeadm init phase upload-certs \
--upload-certs
```

Получить команду подключения:

```bash
kubeadm token create \
--print-join-command \
--certificate-key <CERTIFICATE_KEY>
```

Выполнить полученную команду на:

* k8s-master-02
* k8s-master-03

---

# ➕ Добавление Worker узлов

Получить команду:

```bash
kubeadm token create \
--print-join-command
```

Выполнить её на:

* k8s-worker-01
* k8s-worker-02

---

# 🔍 Проверка кластера

На любом Master:

```bash
kubectl get nodes
```

Ожидаемый результат:

```text
k8s-master-01
k8s-master-02
k8s-master-03
k8s-worker-01
k8s-worker-02
```

---

# 🩺 Диагностика

Логи kubelet:

```bash
journalctl -u kubelet -f
```

Логи containerd:

```bash
journalctl -u containerd -f
```

Состояние кластера:

```bash
kubectl cluster-info

kubectl get nodes

kubectl get pods -A
```

---

# ✅ Результат

После завершения лабораторной работы будет развернут Kubernetes 1.28 кластер:

* 3 Control Plane
* 2 Worker
* containerd
* kubeadm
* kubelet
* kubectl

Установка не зависит от репозиториев Kubernetes и может выполняться в полностью изолированной сети без доступа в Интернет.

Следующий этап — установка сетевого плагина Calico или Cilium.
