# 🧪 Lab 2: Развертывание CNI плагина (Calico)

## 📋 Обзор
Эта лабораторная работа учит развертывать Container Network Interface (CNI) плагин для обеспечения сетевой связи между Pods.

## 🎯 Цели
- Понять роль CNI в Kubernetes
- Установить Calico как CNI плагин
- Проверить работу сети
- Тестировать связь между Pods

## 📊 Требуемое окружение

**Использовать кластер из Lab 1:**
```
✅ 3 мастера (все Ready)
✅ 2 воркера (все Ready)
✅ Все узлы NotReady без CNI (нормально)
✅ Все поды в кубе системе работают
```

---

## 📝 Пошаговые инструкции

### Шаг 1: Проверка текущего состояния кластера

**Выполнить на ЛЮБОМ мастере:**

```bash
# Проверка узлов
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS     ROLES           AGE   VERSION
# k8s-master-01   NotReady   control-plane   15m   v1.28.0
# k8s-master-02   NotReady   control-plane   10m   v1.28.0
# k8s-master-03   NotReady   control-plane   9m    v1.28.0
# k8s-worker-01   NotReady   <none>          5m    v1.28.0
# k8s-worker-02   NotReady   <none>          4m    v1.28.0

# Проверка CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ожидаемый результат:
# NAME                     READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xxxxx 0/1     Pending   0          15m
# coredns-5d78c9869d-xxxxx 0/1     Pending   0          15m
```

---

### Шаг 2: Загрузка манифеста Calico

**Выполнить на `k8s-master-01`:**

```bash
# Скачивание официального манифеста Calico
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Альтернатива если wget не работает
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Проверка файла
ls -lh tigera-operator.yaml
head -20 tigera-operator.yaml

# Ожидаемый размер: ~500 KB
```

---

### Шаг 3: Применение Calico оператора

**Выполнить на `k8s-master-01`:**

```bash
# Применение манифеста
kubectl apply -f tigera-operator.yaml

# Ожидаемый результат:
# namespace/tigera-operator created
# customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
# customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
# ... (множество других CRD)
# deployment.apps/tigera-operator created

# Проверка что оператор запускается
kubectl get pods -n tigera-operator

# Ожидаемый результат:
# NAME                              READY   STATUS    RESTARTS   AGE
# tigera-operator-xxxxxxxxxxxxx     1/1     Running   0          10s
```

---

### Шаг 4: Скачивание и применение Calico конфигурации

**Выполнить на `k8s-master-01`:**

```bash
# Скачивание конфигурации Installation
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/installation-resources.yaml

# Проверка файла
cat installation-resources.yaml | head -50

# Ожидаемо увидеть:
# apiVersion: operator.tigera.io/v1
# kind: Installation
# metadata:
#   name: default
# spec:
#   calicoNetwork:
#     ipPools:
#     - blockSize: 26
#       cidr: 10.244.0.0/16
#       encapsulation: VXLANCrossSubnet
#       natOutgoing: Enabled
#       nodeSelector: all()
```

**Редактирование конфигурации (если нужно):**

```bash
# Если у вас другой pod-network-cidr, отредактируйте cidr
# В нашем случае это 10.244.0.0/16 - совпадает!

# Применение конфигурации
kubectl apply -f installation-resources.yaml

# Ожидаемый результат:
# installation.operator.tigera.io/default created
```

---

### Шаг 5: Мониторинг процесса развертывания Calico

**Выполнить на `k8s-master-01`:**

```bash
# Проверка процесса (может занять 2-5 минут)
watch kubectl get pods -A

# Или в отдельной сессии каждые 5 сек проверять
while true; do
  echo "=== $(date) ==="
  kubectl get nodes
  echo
  kubectl get pods -n calico-system
  echo
  kubectl get pods -n calico-apiserver
  sleep 5
  clear
done

# Ожидаемый прогресс:
# 1. Появляются поды в calico-system namespace
# 2. Появляются поды в calico-apiserver namespace
# 3. Поды переходят в Running
# 4. Узлы переходят в Ready
```

---

### Шаг 6: Проверка статуса установки

**Выполнить на `k8s-master-01`:**

```bash
# Проверка пространства имен
kubectl get namespace | grep calico

# Ожидаемый результат:
# calico-system            Active   5m
# calico-apiserver         Active   5m
# tigera-operator          Active   10m

# Проверка подов в calico-system
kubectl get pods -n calico-system

# Ожидаемый результат (все должны быть Running):
# NAME                                      READY   STATUS    RESTARTS   AGE
# calico-kube-controllers-xxxxxxxxxxxxx     1/1     Running   0          3m
# calico-node-xxxxx                         1/1     Running   0          3m
# calico-node-xxxxx                         1/1     Running   0          3m
# calico-node-xxxxx                         1/1     Running   0          3m
# calico-node-xxxxx                         1/1     Running   0          3m
# calico-node-xxxxx                         1/1     Running   0          3m

# Проверка узлов - ДОЛЖНЫ БЫТЬ READY!
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS   ROLES           AGE   VERSION
# k8s-master-01   Ready    control-plane   20m   v1.28.0
# k8s-master-02   Ready    control-plane   15m   v1.28.0
# k8s-master-03   Ready    control-plane   14m   v1.28.0
# k8s-worker-01   Ready    <none>          10m   v1.28.0
# k8s-worker-02   Ready    <none>          9m    v1.28.0
```

---

### Шаг 7: Проверка CoreDNS

**Выполнить на `k8s-master-01`:**

```bash
# Проверка CoreDNS подов
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ожидаемый результат (ДОЛЖНЫ быть Running):
# NAME                     READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xxxxx 1/1     Running   0          20m
# coredns-5d78c9869d-xxxxx 1/1     Running   0          20m

# Проверка logs CoreDNS
kubectl logs -n kube-system deployment/coredns

# Ожидаемый результат:
# .:53
# [SUCCESS] plugin/reload: Running with pid 1
# [INFO] plugin/ready: Making DNS ready UDP:53
# [INFO] plugin/ready: Making DNS ready TCP:53
```

---

### Шаг 8: Проверка сетевых политик

**Выполнить на `k8s-master-01`:**

```bash
# Проверка Calico конфигурации
kubectl get installation default -o yaml | head -50

# Ожидаемо увидеть:
# spec:
#   calicoNetwork:
#     ipPools:
#     - blockSize: 26
#       cidr: 10.244.0.0/16
#       encapsulation: VXLANCrossSubnet
#       natOutgoing: Enabled
#       nodeSelector: all()

# Проверка BGP конфигурации (опционально)
kubectl get bgpconfiguration

# Проверка IP Pool
kubectl get ippools

# Ожидаемый результат:
# NAME                  CREATED AT
# default               2024-06-14T10:45:00Z
# default-ipv6          2024-06-14T10:45:00Z
```

---

### Шаг 9: Тестирование сетевой связи

**Выполнить на `k8s-master-01`:**

```bash
# Создание тестовых подов
kubectl run test-pod-1 --image=alpine:latest --restart=Never -- sleep 3600
kubectl run test-pod-2 --image=alpine:latest --restart=Never -- sleep 3600

# Ожидаемый результат:
# pod/test-pod-1 created
# pod/test-pod-2 created

# Ожидание запуска подов
sleep 5

# Получение IP адресов подов
kubectl get pods -o wide

# Ожидаемый результат (поды должны иметь IP):
# NAME         READY   STATUS    RESTARTS   AGE   IP            NODE
# test-pod-1   1/1     Running   0          10s   10.244.x.x    k8s-worker-01
# test-pod-2   1/1     Running   0          10s   10.244.y.y    k8s-worker-02

# Тестирование связи между подами
kubectl exec test-pod-1 -- ping -c 3 10.244.y.y

# Ожидаемый результат:
# PING 10.244.y.y (10.244.y.y): 56 data bytes
# 64 bytes from 10.244.y.y: seq=0 ttl=62 time=1.234 ms
# 64 bytes from 10.244.y.y: seq=1 ttl=62 time=1.456 ms
# 64 bytes from 10.244.y.y: seq=2 ttl=62 time=1.678 ms
# --- 10.244.y.y statistics ---
# 3 packets transmitted, 3 packets received, 0% packet loss

# ✅ ЕСЛИ ПИНГ РАБОТАЕТ - CNI НАСТРОЕНА ПРАВИЛЬНО!

# Удаление тестовых подов
kubectl delete pod test-pod-1 test-pod-2
```

---

### Шаг 10: Проверка сетевых интерфейсов на узлах

**Выполнить на ЛЮБОМ worker узле:**

```bash
# SSH на воркер
ssh user@10.0.1.20

# Проверка сетевых интерфейсов
ip link show

# Ожидаемо увидеть интерфейсы:
# - eth0: основной интерфейс
# - docker0: docker bridge (может быть)
# - veth-xxxxx: виртуальные интерфейсы для контейнеров
# - tunl0: Calico туннель для encapsulation

# Проверка маршрутов
ip route

# Ожидаемо увидеть маршруты для 10.244.x.x (Pod CIDR)

# Проверка Calico демона
sudo ps aux | grep calico-node

# Ожидаемо увидеть процесс calico-node
```

---

## ✅ Проверка успешности выполнения

**Все условия должны быть выполнены:**
```
✅ Все 5 узлов имеют статус Ready
✅ 3 мастера имеют role control-plane
✅ Все поды в calico-system работают
✅ Все поды в kube-system работают
✅ CoreDNS поды в Running
✅ Связь между подами работает (ping тест)
✅ Каждый узел имеет IP из 10.244.0.0/16
✅ Нет ошибок в логах Calico компонентов
```

---

## 🔍 Диагностика проблем

### Проблема: Узлы остаются NotReady
```bash
# Проверка статуса Calico компонентов
kubectl get pods -n calico-system

# Если есть CrashLoopBackOff, проверить логи
kubectl logs -n calico-system pod/calico-node-xxxxx

# Проверка дискового пространства
df -h

# Перезагрузка Calico узла
kubectl delete pod -n calico-system calico-node-xxxxx
```

### Проблема: Поды не получают IP адреса
```bash
# Проверка IP Pool
kubectl get ippools

# Если пусто, переприменить installation
kubectl apply -f installation-resources.yaml

# Проверка calico-kube-controllers
kubectl logs -n calico-system deployment/calico-kube-controllers
```

### Проблема: Нет сетевой связи между подами
```bash
# Проверка сетевых политик
kubectl get networkpolicies --all-namespaces

# Если есть restrictive policies, они могут блокировать трафик

# Проверка iptables на узле
sudo iptables -L -n | grep CALICO

# Проверка логов calico-node
kubectl logs -n calico-system pod/calico-node-xxxxx -f
```

---

## 📚 Дополнительные команды

```bash
# Проверка версии Calico
kubectl get daemonset -n calico-system calico-node -o yaml | grep image

# Просмотр Calico конфигурации
kubectl describe installation default

# Проверка BGP статуса (если используется BGP)
kubectl get bgpconfiguration

# Удаление Calico (если нужно переустановить)
kubectl delete -f installation-resources.yaml
kubectl delete -f tigera-operator.yaml

# Просмотр IP Pool использования
kubectl get ipamblocks

# Проверка Calico версии в образе
kubectl describe pod -n calico-system calico-node-xxxxx | grep Image
```

---

## 🎓 Что мы выучили

✅ Роль CNI плагина в Kubernetes  
✅ Установка Calico как CNI  
✅ Проверка статуса сети  
✅ Тестирование связи между Pods  
✅ Диагностика сетевых проблем  
✅ IP Pool конфигурация  

---

## 🚀 Следующие шаги

Переход к **Lab 3**: Создание и управление Pods
