# 🔧 Гайд по установке Kubernetes из России

## Проблема
Пакеты Kubernetes на pkgs.k8s.io заблокированы для РФ. Нужны альтернативные способы.

---

## ✅ РЕШЕНИЕ 1: Скачивание бинарных файлов из GitHub (РАБОТАЕТ!)

**ЭТО САМЫЙ НАДЕЖНЫЙ СПОСОБ для России**

### Шаг 1: Подготовка всех ВМ (как обычно)

```bash
# На всех 5 ВМ выполнить:
sudo yum update -y
sudo yum install -y vim curl wget git htop net-tools

# Отключение swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules
sudo tee /etc/modules-load.d/kubernetes.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl
sudo tee /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# containerd (как обычно)
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Шаг 2: Скачивание бинарных файлов Kubernetes

**Выполнить НА ВСЕХ 5 ВМ:**

```bash
# Переход в /tmp
cd /tmp

# Переменная версии
K8S_VERSION="1.28.0"

# ===== СКАЧИВАНИЕ =====
# GitHub работает из России! Используем этот способ

echo "[*] Скачиваю Kubernetes v${K8S_VERSION}..."

# Скачиваем сервер (содержит всё необходимое)
wget -q https://github.com/kubernetes/kubernetes/releases/download/v${K8S_VERSION}/kubernetes-server-linux-amd64.tar.gz \
  -O kubernetes.tar.gz

# Проверка
ls -lh kubernetes.tar.gz

# Распаковка
echo "[*] Распаковываю архив..."
tar xzf kubernetes.tar.gz

# Проверка бинарных файлов
echo "[*] Проверяю компоненты..."
ls -lh kubernetes/server/bin/ | grep -E 'kubelet|kubectl|kubeadm'

# Копирование в /usr/local/bin
echo "[*] Устанавливаю компоненты..."
sudo install -o root -g root -m 0755 kubernetes/server/bin/kubelet /usr/local/bin/
sudo install -o root -g root -m 0755 kubernetes/server/bin/kubectl /usr/local/bin/
sudo install -o root -g root -m 0755 kubernetes/server/bin/kubeadm /usr/local/bin/

# Проверка версий
echo "[*] Проверяю версии..."
kubelet --version
kubectl version --client 2>/dev/null || kubectl --version
kubeadm version

echo "[✓] Успешно установлено!"
```

### Шаг 3: Создание systemd юнита для kubelet

**Выполнить НА ВСЕХ 5 ВМ:**

```bash
# Создание конфига kubelet
sudo mkdir -p /var/lib/kubelet

sudo tee /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \\
  --kubeconfig=/etc/kubernetes/kubelet.conf \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \\
  --cgroup-driver=systemd

Restart=always
RestartSec=5
StartLimitInterval=0
StartLimitBurst=0

Type=notify
Delegate=yes
KillMode=process
KillSignal=SIGTERM
Tasks=infinity
MemoryAccounting=true
EventsLimit=0
LoggingRateLimitBurst=0
LoggingRateLimitIntervalSec=0

[Install]
WantedBy=multi-user.target
EOF

# Включение сервиса
sudo systemctl daemon-reload
sudo systemctl enable kubelet

# НЕ запускаем еще! Сначала нужна инициализация kubeadm
```

### Шаг 4: Инициализация Control Plane (ТОЛЬКО k8s-master-01)

**Выполнить ТОЛЬКО на `k8s-master-01`:**

```bash
# Инициализация
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.10 \
  --control-plane-endpoint=10.0.1.10:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0 \
  --cri-socket=unix:///run/containerd/containerd.sock

# Ожидаемый результат:
# Your Kubernetes control-plane has initialized successfully!
# kubeadm join 10.0.1.10:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx

# СОХРАНИТЕ ВСЕ ВЫВЕДЕННЫЕ КОМАНДЫ!

# Настройка kubectl для текущего пользователя
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Проверка
kubectl get nodes
# Статус NotReady - это нормально, нужен CNI
```

### Шаг 5: Добавление других Master нодов

**На k8s-master-01 - получить токены:**

```bash
# Сертификаты для других мастеров
sudo kubeadm init phase upload-certs --upload-certs
# Сохраните ключ! (длинная строка после "Storing the certificates")

# Токен для мастеров
kubeadm token create --print-join-command --certificate-key ВСТАВЬТЕ_КЛЮЧ_СЮДА

# Токен для рабочих узлов
kubeadm token create --print-join-command

# Сохраните обе команды!
```

**На k8s-master-02 и k8s-master-03:**

```bash
# Присоединение как Master
sudo kubeadm join 10.0.1.10:6443 \
  --token XXXXX \
  --discovery-token-ca-cert-hash sha256:XXXXX \
  --control-plane \
  --certificate-key XXXXXXXX...

# Копирование конфига
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Шаг 6: Добавление Worker нодов

**На k8s-worker-01 и k8s-worker-02:**

```bash
# Присоединение как Worker
sudo kubeadm join 10.0.1.10:6443 \
  --token XXXXX \
  --discovery-token-ca-cert-hash sha256:XXXXX

# Ожидаемый результат:
# This node has joined the cluster as a worker node.
```

### Шаг 7: Установка CNI (Calico) - На k8s-master-01

```bash
# Скачивание через GitHub (тоже работает!)
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl apply -f tigera-operator.yaml

# Скачивание конфига
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/installation-resources.yaml

# Если нужны другие сети, отредактируйте cidr
# Наш cidr 10.244.0.0/16 уже совпадает

kubectl apply -f installation-resources.yaml

# Ожидание (3-5 минут)
watch kubectl get pods -A

# Проверка - все узлы должны быть Ready
kubectl get nodes
```

---

## ✅ РЕШЕНИЕ 2: Использовать Minikube (БЫСТРЕЕ)

**Если хотите быстро начать работать:**

```bash
# На одной машине с Docker

# Скачивание Minikube
cd /tmp
wget https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Скачивание kubectl (если еще нет)
wget https://github.com/kubernetes/kubernetes/releases/download/v1.28.0/bin/linux/amd64/kubectl
sudo install kubectl /usr/local/bin/kubectl

# Запуск кластера
minikube start --driver=docker --cpus=4 --memory=4096

# Проверка
kubectl get nodes
kubectl get pods -A

# Готово! Сразу можно работать
```

---

## ✅ РЕШЕНИЕ 3: Docker Desktop + Kubernetes

```bash
# Если у вас есть Docker Desktop (Windows/Mac)

# Просто включите Kubernetes в настройках Docker Desktop
# Settings → Kubernetes → Enable Kubernetes

# Готово!
```

---

## 🎯 МОЯТА РЕКОМЕНДАЦИЯ ДЛЯ ВАШЕГО КУРСА

### Для быстрого старта (первые 2 недели):
1. Используйте **Minikube** - 5 минут на установку
2. Вся теория и практика работает на нем
3. Все лабы одинаковы (kubectl команды универсальны)

### Для продвинутого курса (потом):
1. Переходите на **real K8s cluster** (Решение 1)
2. На той же инфраструктуре что вы подготовили
3. Используйте скачанные бинарные файлы из GitHub

### Для production:
1. Используйте **package managers** если доступны локально
2. Или **binary installation** (Решение 1) - всегда работает
3. Or **managed K8s** (Yandex.Cloud, VK.Cloud, etc.)

---

## 🔍 ПРОВЕРКА ЧТО РАБОТАЕТ

```bash
# Все узлы Ready?
kubectl get nodes
# Все должны быть Status=Ready

# Все pods running?
kubectl get pods -A
# Все должны быть Running или Completed

# CoreDNS работает?
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Должны быть Running

# Тест связи
kubectl run test-pod --image=alpine:latest --restart=Never -- sleep 3600
kubectl exec test-pod -- ping -c 3 1.1.1.1
# Должен быть пинг
```

---

## 📝 ВАЖНЫЕ ЗАМЕЧАНИЯ

1. **GitHub работает из России** - используйте его для скачивания
2. **Бинарные файлы vs Package manager** - результат одинаковый
3. **Версия 1.28** - стабильная, хорошо документирована
4. **Minikube** - лучший вариант для обучения
5. **containerd** - стандартный container runtime (Docker больше не нужен)

---

## 🆘 ЕСЛИ ЧТО-ТО НЕ РАБОТАЕТ

```bash
# Проверка интернета к GitHub
curl -I https://github.com
# Должен быть HTTP/200

# Если блокировано - используйте VPN
# Но для GitHub обычно достаточно

# Проверка kubelet
sudo journalctl -u kubelet -n 50

# Проверка containerd
sudo systemctl status containerd

# Перезагрузка всего
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

---

## ✨ РЕЗЮМЕ

✅ **GitHub работает из России**  
✅ **Бинарные файлы скачиваются нормально**  
✅ **Calico manifests тоже на GitHub**  
✅ **Minikube - самый быстрый вариант**  
✅ **Real K8s cluster - когда будете готовы**  

**Начните с Minikube, потом переходите на real cluster!**
