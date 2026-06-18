# 🧪 Lab 1: Установка Kubernetes кластера с kubeadm

## 📋 Обзор
Эта лабораторная работа учит устанавливать и инициализировать production-like Kubernetes кластер используя kubeadm.

## 🎯 Цели
- Установить все компоненты K8s
- Инициализировать control plane
- Добавить worker nodes
- Проверить работу кластера

## 📊 Требуемое окружение

**ВМ которые нужно развернуть:**
```
5 Virtual Machines (все с Rocky Linux 10):

1. k8s-master-01
   - vCPU: 4 (минимум 2)
   - RAM: 4 GB (минимум 2)
   - Storage: 50 GB
   - IP: 10.0.1.10
   - Role: Control Plane (Master)

2. k8s-master-02
   - vCPU: 4
   - RAM: 4 GB
   - Storage: 50 GB
   - IP: 10.0.1.11
   - Role: Control Plane (Backup)

3. k8s-master-03
   - vCPU: 4
   - RAM: 4 GB
   - Storage: 50 GB
   - IP: 10.0.1.12
   - Role: Control Plane (Backup)

4. k8s-worker-01
   - vCPU: 4
   - RAM: 4 GB
   - Storage: 50 GB
   - IP: 10.0.1.20
   - Role: Worker

5. k8s-worker-02
   - vCPU: 4
   - RAM: 4 GB
   - Storage: 50 GB
   - IP: 10.0.1.21
   - Role: Worker

Итого: 3 Master + 2 Worker
```

**Требования ко всем ВМ:**
- ✅ Rocky Linux 10 (или совместимый Linux)
- ✅ Минимум 2 CPU, 2 GB RAM
- ✅ SSH доступ между всеми узлами
- ✅ Интернет доступ для загрузки образов
- ✅ Ports открыты (6443, 2379, 10250, etc.)
- ✅ Swap отключен
- ✅ Kernel modules загружены (overlay, br_netfilter)

---

## 📝 Пошаговые инструкции

### Шаг 1: Подготовка всех узлов

**Выполнить на ВСЕХ ВМ (мастерах и воркерах):**

```bash
# Обновление системы
sudo yum update -y

# Установка необходимых инструментов
sudo yum install -y vim curl wget git htop net-tools

# Отключение swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Проверка
free -h
# Swap должна быть 0
```

---

### Шаг 2: Загрузка kernel modules

**Выполнить на ВСЕХ ВМ:**

```bash
# Создание файла конфигурации
sudo tee /etc/modules-load.d/kubernetes.conf <<EOF
overlay
br_netfilter
EOF

# Загрузка модулей
sudo modprobe overlay
sudo modprobe br_netfilter

# Проверка
lsmod | grep -E 'overlay|br_netfilter'

# Ожидаемый результат:
# br_netfilter            28672  0
# overlay                86016  0
```

---

### Шаг 3: Настройка sysctl параметров

**Выполнить на ВСЕХ ВМ:**

```bash
# Создание конфигурации
sudo tee /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Применение параметров
sudo sysctl --system

# Проверка
sudo sysctl net.ipv4.ip_forward
# Ожидаемый результат: 1
```

---

### Шаг 4: Установка Container Runtime (containerd)

**Выполнить на ВСЕХ ВМ:**

```bash
# Добавление репозитория Docker
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Установка containerd
sudo yum install -y containerd.io

# Создание конфигурации
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Редактирование конфига для systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Проверка
sudo grep SystemdCgroup /etc/containerd/config.toml
# Ожидаемый результат: SystemdCgroup = true

# Запуск containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Проверка
sudo systemctl status containerd
# Ожидаемый результат: active (running)
```

---

### Шаг 5: Установка kubeadm, kubelet, kubectl

**Выполнить на ВСЕХ ВМ:**

```bash
# Добавление Kubernetes репозитория
wget https://dl.k8s.io/v1.28.0/kubernetes-server-linux-amd64.tar.gz

# Проверка версий
kubeadm version
kubectl version --client
kubelet --version

# Включение kubelet сервиса
sudo systemctl enable kubelet

# Запуск (но не стартует до инициализации)
sudo systemctl start kubelet
sudo systemctl status kubelet
# Статус может быть "activating" - это нормально
```

---

### Шаг 6: Инициализация Control Plane (Master)

**Выполнить ТОЛЬКО на `k8s-master-01`:**

```bash
# Инициализация первого master узла
sudo kubeadm init \
  --apiserver-advertise-address=192.168.77.181 \
  --control-plane-endpoint=192.168.77.180:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0 \
  --cri-socket=unix:///run/containerd/containerd.sock

# Параметры:
# --apiserver-advertise-address    - IP адрес мастера для других узлов
# --control-plane-endpoint          - VIP для load balancer (если нет LB, то IP первого мастера)
# --pod-network-cidr               - CIDR для Pods (для Calico)
# --service-cidr                   - CIDR для Services
# --cri-socket                     - containerd socket

# Ожидаемый результат:
# Your Kubernetes control-plane has initialized successfully!
# kubeadm join 10.0.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx

# СОХРАНИТЬ ВЫВЕДЕННЫЕ КОМАНДЫ! Они нужны для добавления узлов
```

**Сохраните две команды:**
1. Для добавления мастеров
2. Для добавления воркеров

---

### Шаг 7: Настройка kubectl для текущего пользователя

**Выполнить на `k8s-master-01`:**

```bash
# Для текущего пользователя
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Проверка
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS     ROLES           AGE   VERSION
# k8s-master-01   NotReady   control-plane   2m    v1.28.0
# NotReady потому что нет CNI плагина

# Bash completion (опционально)
sudo yum install -y bash-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```

---

### Шаг 8: Добавление других Master узлов

**Выполнить на `k8s-master-01` для получения токена:**

```bash
# Создание сертификатов для новых мастеров
sudo kubeadm init phase upload-certs --upload-certs

# Вывод:
# W0614 10:30:00.123456 1234 validation.go:28] Cannot validate kube-apiserver certificate
# Etcd certificates already exist. Skipping etcd certificate generation
# [upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
# [upload-certs] Using certificate key:
# xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# СОХРАНИТЕ ЭТУ СТРОКУ!

# Получение токена для мастеров
kubeadm token create --print-join-command --certificate-key xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Ожидаемый результат:
# kubeadm join 10.0.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx --control-plane --certificate-key xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# СОХРАНИТЕ ЭТУ КОМАНДУ!
```

**Выполнить на `k8s-master-02` и `k8s-master-03`:**

```bash
# Присоединение к кластеру как мастер
sudo kubeadm join 10.0.1.10:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx \
  --control-plane \
  --certificate-key xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Ожидаемый результат:
# This node has joined the cluster as a control-plane node.

# Копирование конфига
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Проверка
kubectl get nodes
```

---

### Шаг 9: Добавление Worker узлов

**Выполнить на `k8s-master-01` для получения токена (если истек):**

```bash
# Создание нового токена для воркеров
kubeadm token create --print-join-command

# Ожидаемый результат:
# kubeadm join 10.0.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx

# СОХРАНИТЕ ЭТУ КОМАНДУ!
```

**Выполнить на `k8s-worker-01` и `k8s-worker-02`:**

```bash
# Присоединение к кластеру как worker
sudo kubeadm join 10.0.1.10:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx

# Ожидаемый результат:
# This node has joined the cluster as a worker node.
```

---

### Шаг 10: Проверка состояния кластера

**Выполнить на ЛЮБОМ мастере:**

```bash
# Проверка узлов
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS     ROLES           AGE     VERSION
# k8s-master-01   NotReady   control-plane   10m     v1.28.0
# k8s-master-02   NotReady   control-plane   5m      v1.28.0
# k8s-master-03   NotReady   control-plane   4m      v1.28.0
# k8s-worker-01   NotReady   <none>          2m      v1.28.0
# k8s-worker-02   NotReady   <none>          1m      v1.28.0
# NotReady потому что нет CNI плагина

# Проверка подов
kubectl get pods -A

# Ожидаемый результат:
# NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
# kube-system   coredns-5d78c9869d-xxxxx        0/1     Pending   0          5m
# kube-system   coredns-5d78c9869d-xxxxx        0/1     Pending   0          5m
# kube-system   etcd-k8s-master-01              1/1     Running   0          5m
# kube-system   etcd-k8s-master-02              1/1     Running   0          3m
# kube-system   etcd-k8s-master-03              1/1     Running   0          2m
# kube-system   kube-apiserver-k8s-master-01    1/1     Running   0          5m
# kube-system   kube-apiserver-k8s-master-02    1/1     Running   0          3m
# kube-system   kube-apiserver-k8s-master-03    1/1     Running   0          2m
# kube-system   kube-controller-manager-xxx     1/1     Running   1          5m
# kube-system   kube-proxy-xxxxx                1/1     Running   0          5m
# kube-system   kube-scheduler-xxxxx            1/1     Running   0          5m
```

---

## ✅ Проверка успешности выполнения

**Все условия должны быть выполнены:**
```
✅ 5 узлов видны в кластере (3 Master + 2 Worker)
✅ Все мастера имеют статус "control-plane"
✅ Воркеры имеют пустые ROLES
✅ Все узлы имеют версию v1.28.0
✅ CoreDNS поды Pending (в ожидании CNI)
✅ Все system poды работают кроме CoreDNS
✅ etcd работает на всех мастерах
✅ kube-apiserver доступен на всех мастерах
```

---

## 🔍 Диагностика проблем

### Проблема: Узлы не присоединяются
```bash
# Проверка ошибок на воркере
sudo journalctl -u kubelet -n 50

# Проверка сетевого доступа
ping 10.0.1.10
telnet 10.0.1.10 6443

# Проверка firewall
sudo firewall-cmd --list-ports

# Открытие необходимых портов
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=2379-2380/tcp --permanent
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --add-port=10251/tcp --permanent
sudo firewall-cmd --add-port=10252/tcp --permanent
sudo firewall-cmd --reload
```

### Проблема: Control plane не стартует
```bash
# Проверка логов apiserver
sudo journalctl -u kubelet | grep -i error

# Проверка дискового пространства
df -h /var

# Проверка etcd
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=127.0.0.1:2379 \
  endpoint health
```

### Проблема: Статус NotReady
```bash
# Это нормально без CNI плагина
# Следующий шаг: Lab 2 (установка Calico)

# Проверка события на узле
kubectl describe node k8s-master-01

# Проверка логов kubelet
sudo journalctl -u kubelet -f
```

---

## 📚 Дополнительные команды

```bash
# Получение информации о кластере
kubectl cluster-info
kubectl cluster-info dump

# Проверка конфигурации
kubectl config view

# Информация об узле
kubectl describe node k8s-master-01

# Логи компонентов
kubectl logs -n kube-system pod/coredns-xxxxx

# Проверка API ресурсов
kubectl api-resources

# Версия сервера
kubectl version

# Проверка здоровья
kubectl get componentstatus
```

---

## 🎓 Что мы выучили

✅ Подготовка узлов для Kubernetes  
✅ Установка kubeadm, kubelet, kubectl  
✅ Инициализация control plane  
✅ Добавление мастеров для High Availability  
✅ Добавление worker узлов  
✅ Проверка состояния кластера  
✅ Диагностика проблем  

---

## 🚀 Следующие шаги

Переход к **Lab 2**: Развертывание CNI плагина (Calico/Cilium)
