# 🧪 Lab 1: Установка Kubernetes кластера с kubeadm

## 📋 Обзор

Эта лабораторная работа посвящена развертыванию отказоустойчивого (production-like) кластера Kubernetes версии v1.28.0 с использованием утилиты `kubeadm` в изолированном или локальном окружении.

## 🎯 Цели

* Подготовить операционную систему и среду выполнения контейнеров (containerd).
* Установить базовые компоненты Kubernetes из бинарных файлов.
* Инициализировать отказоустойчивый Control Plane.
* Подключить Worker-ноды к кластеру.

## 📊 Требуемое окружение

**Спецификация виртуальных машин (Rocky Linux 10):**

| Имя узла | IP-адрес | vCPU | RAM | Диск | Роль |
| --- | --- | --- | --- | --- | --- |
| **k8s-master-01** | 192.168.77.181 | 4 | 4 GB | 50 GB | Control Plane (Инициатор) |
| **k8s-master-02** | 192.168.77.182 | 4 | 4 GB | 50 GB | Control Plane (Резерв) |
| **k8s-master-03** | 192.168.77.183 | 4 | 4 GB | 50 GB | Control Plane (Резерв) |
| **k8s-worker-01** | 192.168.77.186 | 4 | 4 GB | 50 GB | Worker-нода |
| **k8s-worker-02** | 192.168.77.187 | 4 | 4 GB | 50 GB | Worker-нода |

**Обязательные требования ко всем узлам:**

* ✅ Отключенный раздел подкачки (Swap).
* ✅ Настроенная сетевая связанность и открытые порты (6443, 2379-2380, 10250).
* ✅ Загруженные модули ядра ядра Linux (`overlay`, `br_netfilter`).

---

## 📝 Пошаговые инструкции

### Шаг 1: Подготовка директорий и загрузка дистрибутивов

Чтобы обеспечить независимость от внешних репозиториев (Air-Gapped подход), мы скачаем все необходимые бинарные файлы на первой мастер-ноде, а затем распространим их.

**Выполнить на `k8s-master-01`:**

```bash
# Создание рабочей структуры каталогов
mkdir -p ~/k8s-lab/files
cd ~/k8s-lab/files

# 1. Загрузка рантайма containerd
wget https://download.docker.com/linux/centos/10/x86_64/stable/Packages/containerd.io-1.7.29-1.el10.x86_64.rpm
mv containerd.io-1.7.29-1.el10.x86_64.rpm containerd.io.rpm

# 2. Загрузка серверных бинарников Kubernetes (kubeadm, kubelet, kubectl)
wget https://dl.k8s.io/v1.28.0/kubernetes-server-linux-amd64.tar.gz
tar xzf kubernetes-server-linux-amd64.tar.gz
cp kubernetes/server/bin/{kubeadm,kubelet,kubectl} ./
rm -rf kubernetes kubernetes-server-linux-amd64.tar.gz

# 3. Загрузка утилиты отладки CRI
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz
sudo tar zxvf crictl-v1.28.0-linux-amd64.tar.gz crictl
rm -f crictl-v1.28.0-linux-amd64.tar.gz

```

---

### Шаг 2: Создание конфигурационных файлов служб

Подготовим конфигурацию системных юнитов для `kubelet`, которую наш автоматический скрипт разложит по путям системы.

**Выполнить на `k8s-master-01` в директории `~/k8s-lab/files`:**

```bash
# Создание базового юнита kubelet
cat > kubelet.service <<EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Создание конфигурации интеграции с kubeadm
cat > 10-kubeadm.conf <<EOF
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
EOF

```

---

### Шаг 3: Написание и запуск скрипта автоматической настройки нод

Этот скрипт полностью берет на себя рутинную настройку ОС (модули ядра, sysctl, отключение Swap, установка рантайма containerd и компонентов K8s).

**Выполнить на `k8s-master-01` (создать файл `~/k8s-lab/install-k8s.sh`):**

```bash
cat > ~/k8s-lab/install-k8s.sh <<'EOF'
#!/bin/bash
set -e

echo "[1/6] Отключение Swap и Firewalld"
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
systemctl stop firewalld || true
systemctl disable firewalld || true

echo "[2/6] Настройка модулей ядра"
cat > /etc/modules-load.d/kubernetes.conf <<MODULES
overlay
br_netfilter
MODULES
modprobe overlay
modprobe br_netfilter

echo "[3/6] Настройка параметров sysctl (сеть)"
cat > /etc/sysctl.d/99-kubernetes.conf <<SYSCTL
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
SYSCTL
sysctl --system

echo "[4/6] Установка и настройка containerd"
dnf install -y ./files/containerd.io.rpm
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl daemon-reload
systemctl enable --now containerd

echo "[5/6] Установка утилит Kubernetes"
install files/kubeadm /usr/local/bin/
install files/kubelet /usr/local/bin/
install files/kubectl /usr/local/bin/
if [ -f files/crictl ]; then install files/crictl /usr/local/bin/; fi

ln -sf /usr/local/bin/kubeadm /usr/bin/kubeadm
ln -sf /usr/local/bin/kubectl /usr/bin/kubectl
ln -sf /usr/local/bin/kubelet /usr/bin/kubelet
ln -sf /usr/local/bin/crictl /usr/bin/crictl

echo "[6/6] Настройка службы kubelet"
cp files/kubelet.service /usr/lib/systemd/system/
mkdir -p /usr/lib/systemd/system/kubelet.service.d
cp files/10-kubeadm.conf /usr/lib/systemd/system/kubelet.service.d/

systemctl daemon-reload
systemctl enable kubelet

echo "=== Предварительная загрузка образов системных контейнеров ==="
kubeadm config images pull --kubernetes-version=v1.28.0

echo "ПОДГОТОВКА УЗЛА ЗАВЕРШЕНА УСПЕШНО!"
EOF
chmod +x ~/k8s-lab/install-k8s.sh

```

> **Важно:** Скопируйте всю директорию `~/k8s-lab` на остальные 4 ноды кластера (используя `scp` или аналогичный инструмент) и запустите скрипт на **каждом** узле:

```bash
sudo cd ~/k8s-lab/ && sudo ./install-k8s.sh

```

---

### Шаг 4: Инициализация Первой Мастер-ноды (Control Plane)

**Выполнить ТОЛЬКО на `k8s-master-01`:**

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.77.181 \
  --control-plane-endpoint=192.168.77.181:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0 \
  --cri-socket=unix:///run/containerd/containerd.sock

```

> **Внимание:** В конце успешного вывода вы получите две разные строки `kubeadm join`. **Обязательно сохраните их!**

---

### Шаг 5: Настройка прав доступа для kubectl

Чтобы начать управлять кластером, пользователю необходим конфигурационный токен администратора.

**Выполнить на `k8s-master-01`:**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Проверка связи с API
kubectl get nodes
# k8s-master-01 должен отобразиться в статусе NotReady (так как еще нет CNI)

```

---

### Шаг 6: Подключение дополнительных Мастер-нод

Для обеспечения высокой доступности (HA) подключим остальные мастера.

**1. Получение ключа сертификатов на `k8s-master-01`:**

```bash
sudo kubeadm init phase upload-certs --upload-certs
# Команда выведет длинный sha256-ключ (Certificate Key). Скопируйте его.

```

**2. Генерация команды подключения для мастеров на `k8s-master-01`:**

```bash
kubeadm token create --print-join-command --certificate-key <ВАШ_CERTIFICATE_KEY>

```

**3. Выполнение команды на `k8s-master-02` и `k8s-master-03`:**
Выполните сгенерированную команду на резервных мастерах, обязательно добавив в конец флаг `--control-plane`:

```bash
sudo kubeadm join 192.168.77.181:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>

```

*После успешного подключения скопируйте конфигурацию `.kube/config` с первого мастера на второй и третий аналогично Шагу 5.*

---

### Шаг 7: Подключение Worker-нод

**1. Получение обычной команды подключения на `k8s-master-01`:**

```bash
kubeadm token create --print-join-command

```

**2. Выполнение на `k8s-worker-01` и `k8s-worker-02`:**

```bash
sudo kubeadm join 192.168.77.181:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

```

---

### Шаг 8: Проверка финального состояния кластера

**Выполнить на `k8s-master-01`:**

```bash
# Контроль всех участников кластера
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS     ROLES           AGE   VERSION
# k8s-master-01   NotReady   control-plane   10m   v1.28.0
# k8s-master-02   NotReady   control-plane   5m    v1.28.0
# k8s-master-03   NotReady   control-plane   4m    v1.28.0
# k8s-worker-01   NotReady   <none>          2m    v1.28.0
# k8s-worker-02   NotReady   <none>          1m    v1.28.0

# Проверка системных подов во всех пространствах
kubectl get pods -A

```

*Все поды (etcd, api, proxy, scheduler) должны быть в состоянии `Running`, кроме подов `coredns` — они законно останутся в статусе `Pending` до момента установки сетевого плагина в следующей лабораторной работе.*

---

## 🔍 Диагностика типовых проблем

* **Проблема: Ошибка `kubeadm join` из-за таймаута сети**
* *Решение:* Убедитесь, что `firewalld` на целевом мастере действительно выключен (`sudo systemctl status firewalld`). Проверьте доступность порта 6443 с воркера: `nc -zvw3 192.168.77.181 6443`.


* **Проблема: Служба `kubelet` падает и не перезапускается**
* *Решение:* Проверьте логи системного репортера: `sudo journalctl -u kubelet -n 50`. Чаще всего падение вызвано невыключенным Swap или конфликтом драйверов cgroup (убедитесь, что в `/etc/containerd/config.toml` параметр `SystemdCgroup` равен `true`).



---

## 🚀 Следующие шаги

Кластер успешно инициализирован и готов к работе. Переходите к **Lab 2**, чтобы развернуть сетевой плагин Calico и перевести узлы в состояние `Ready`.
