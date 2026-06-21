# 🧪 Lab 2: Развертывание CNI-плагина (Calico через Tigera Operator)

## 📋 Обзор

Эта лабораторная работа посвящена развертыванию Container Network Interface (CNI) плагина Calico с использованием современного операторного подхода (Tigera Operator) для обеспечения сетевой связи и безопасности между Pods в кластере Kubernetes.

## 🎯 Цели

* Понять роль CNI в обеспечении сетевого взаимодействия Kubernetes.
* Развернуть Tigera Operator и запустить компоненты Calico.
* Настроить пул IP-адресов (IPAM) для подов.
* Проверить сетевую связь между узлами и подами.

## 📊 Требуемое окружение

**Используется готовый базовый кластер:**

```
✅ 3 Control-Plane узла (мастера)
✅ 2 Worker узла (воркеры)
✅ Все узлы находятся в статусе NotReady (это норма до установки CNI)
✅ Служебные поды (кроме CoreDNS) успешно запущены

```

---

## 📝 Пошаговые инструкции

### Шаг 1: Проверка текущего состояния кластера

Перед установкой сети убедитесь, что кластер ожидает CNI, а поды CoreDNS находятся в состоянии ожидания сети (`Pending`).

**Выполнить на любом мастере (например, `k8s-master-01`):**

```bash
# Проверка статуса узлов
kubectl get nodes

# Ожидаемый результат:
# NAME             STATUS     ROLES           AGE   VERSION
# k8s-master-01    NotReady   control-plane   15m   v1.28.0
# k8s-master-02    NotReady   control-plane   10m   v1.28.0
# k8s-master-03    NotReady   control-plane   9m    v1.28.0
# k8s-worker-01    NotReady   <none>          5m    v1.28.0
# k8s-worker-02    NotReady   <none>          4m    v1.28.0

# Проверка состояния CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ожидаемый результат (поды ждут выделения IP-адресов):
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xxxxx   0/1     Pending   0          15m
# coredns-5d78c9869d-xxxxx   0/1     Pending   0          15m

```

---

### Шаг 2: Загрузка манифеста Tigera Operator

Оператор будет управлять жизненным циклом Calico, его обновлением и конфигурацией.

**Выполнить на `k8s-master-01`:**

```bash
# Скачивание официального манифеста оператора версии v3.26.1
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Проверка загруженного файла
ls -lh tigera-operator.yaml
head -n 20 tigera-operator.yaml

# Ожидаемый размер: ~500-600 KB

```

---

### Шаг 3: Развертывание Tigera Operator

**Выполнить на `k8s-master-01`:**

```bash
# Применение манифеста оператора
kubectl create -f tigera-operator.yaml

# Ожидаемый результат:
# namespace/tigera-operator created
# customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
# ... (список остальных CRD)
# deployment.apps/tigera-operator created

# Отслеживание запуска пода оператора
kubectl get pods -n tigera-operator -w

```

> Дождитесь, пока под оператора перейдет в статус `1/1 Running`, после чего нажмите `Ctrl+C`.

---

### Шаг 4: Настройка и применение кастомных ресурсов (CRD)

Теперь необходимо передать оператору конфигурацию нашей будущей сети.

**Выполнить на `k8s-master-01`:**

```bash
# Скачивание шаблона кастомных ресурсов Calico
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

```

> ⚠️ **ВАЖНО:** Откройте файл `custom-resources.yaml` через текстовый редактор (например, `nano`) и убедитесь, что параметр **`cidr`** совпадает с подсетью подов (`--pod-network-cidr`), заданной при инициализации вашего кластера (например, через `kubeadm init`). По умолчанию там указано `192.168.0.0/16`. Если вы использовали `10.244.0.0/16`, измените это значение в файле.

```yaml
# Пример содержимого custom-resources.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 # Укажите CIDR вашего кластера
      encapsulation: VXLAN # По умолчанию для оператора
      natOutgoing: Enabled
      nodeSelector: all()

```

```bash
# Применение конфигурации сети
kubectl create -f custom-resources.yaml

# Ожидаемый результат:
# installation.operator.tigera.io/default created
# apiserver.operator.tigera.io/default created

```

---

### Шаг 5: Мониторинг процесса развертывания сети

Оператор перехватит конфигурацию, создаст новые пространства имен (`calico-system` и `calico-apiserver`) и развернет туда сетевые агенты.

**Выполнить на `k8s-master-01`:**

```bash
# Удобный мониторинг в реальном времени (выход через Ctrl+C)
watch kubectl get pods -A

```

**Альтернативный скрипт для отслеживания прогресса на экранах отладки:**

```bash
while true; do
  echo "=== Статус узлов ==="; kubectl get nodes
  echo "=== Поды сети Calico ==="; kubectl get pods -n calico-system
  echo "=== API-сервер Calico ==="; kubectl get pods -n calico-apiserver
  sleep 5
  clear
done

```

*Компоненты считаются готовыми, когда все поды в `calico-system` перейдут в статус `Running`, а узлы кластера изменят статус на `Ready`.*

---

### Шаг 6: Проверка статуса установки

**Выполнить на `k8s-master-01`:**

```bash
# Проверка созданных пространств имен сети
kubectl get ns | grep -E 'calico|tigera'

# Ожидаемый результат:
# calico-apiserver   Active   5m
# calico-system      Active   5m
# tigera-operator    Active   10m

# Проверка агентов на узлах (calico-node должен запуститься на каждой ноде)
kubectl get pods -n calico-system -o wide

```

```bash
# Проверка статуса узлов (Все должны стать Ready!)
kubectl get nodes

# Ожидаемый результат:
# NAME             STATUS   ROLES           AGE   VERSION
# k8s-master-01    Ready    control-plane   25m   v1.28.0
# k8s-master-02    Ready    control-plane   20m   v1.28.0
# k8s-master-03    Ready    control-plane   19m   v1.28.0
# k8s-worker-01    Ready    <none>          15m   v1.28.0
# k8s-worker-02    Ready    <none>          14m   v1.28.0

```

---

### Шаг 7: Проверка ожившего CoreDNS

После инициализации сети поды CoreDNS должны автоматически получить IP-адреса и запуститься.

**Выполнить на `k8s-master-01`:**

```bash
# Проверка подов CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ожидаемый результат:
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xxxxx   1/1     Running   0          25m
# coredns-5d78c9869d-xxxxx   1/1     Running   0          25m

# Проверка логов CoreDNS на отсутствие ошибок маршрутизации
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20

```

---

### Шаг 8: Проверка конфигурации IP-пулов (IPAM)

Убедимся, что ресурсы Calico успешно созданы внутри Kubernetes API.

**Выполнить на `k8s-master-01`:**

```bash
# Просмотр параметров установленной фабрики сети
kubectl get installations default -o yaml

# Просмотр созданных пулов IP-адресов
kubectl get ippools

# Ожидаемый результат:
# NAME                  AGE
# default-ipv4-ippool   5m

```

---

### Шаг 9: Тестирование сетевой связи между подами

Проверим, могут ли поды на разных физических (или виртуальных) воркерах общаться между собой напрямую.

**Выполнить на `k8s-master-01`:**

```bash
# Запуск двух тестовых изолированных подов
kubectl run test-pod-1 --image=alpine:latest --restart=Never -- sleep 3600
kubectl run test-pod-2 --image=alpine:latest --restart=Never -- sleep 3600

# Ожидание инициализации (5-10 секунд)
kubectl get pods -o wide

```

*Убедитесь, что `test-pod-1` и `test-pod-2` распределились на **разные** ноды кластера и получили IP-адреса из настроенного пула.*

```bash
# Запуск утилиты ping из первого пода во второй (подставьте IP-адрес test-pod-2)
kubectl exec test-pod-1 -- ping -c 3 <IP_АДРЕС_TEST_POD_2>

# Ожидаемый результат:
# PING x.x.x.x (x.x.x.x): 56 data bytes
# 64 bytes from x.x.x.x: seq=0 ttl=62 time=0.954 ms
# ...
# 3 packets transmitted, 3 packets received, 0% packet loss

# Очистка тестового окружения
kubectl delete pod test-pod-1 test-pod-2

```

---

### Шаг 10: Проверка низкоуровневых интерфейсов на узлах

**Выполнить непосредственно внутри ОС любого воркер-узла (через SSH):**

```bash
# Проверка появления виртуальных интерфейсов Calico и туннелей
ip link show

# Ожидаемые интерфейсы в выводе:
# - vxlan.calico: интерфейс для VXLAN оверлея (если включен VXLAN)
# - caliXXXXXXXXX: виртуальные сетевые пары, подключенные к контейнерам

# Просмотр таблицы маршрутизации ядра Linux
ip route

```

*Вы должны увидеть новые маршруты к подсетям других нод через интерфейсы Calico.*

---

## ✅ Критерии успешного выполнения лабораторной работы

1. Все 5 узлов кластера находятся в статусе `Ready`.
2. В пространстве имен `calico-system` все поды находятся в состоянии `Running` (1/1).
3. Поды `coredns` успешно запущены и имеют статус `Running`.
4. Тест `ping` между подами на разных узлах проходит со 100% успехом без потери пакетов.

---

## 🔍 Краткое руководство по диагностике (Troubleshooting)

* **Поды `calico-node` падают или не проходят Readiness Probe:**
Проверьте, не заблокирован ли сетевой трафик локальными фаерволами хостов. Для VXLAN должен быть открыт порт `UDP 4789`, для BGP — `TCP 179`.
```bash
sudo systemctl stop firewalld # Для проверки на системах RHEL/CentOS
sudo ufw disable              # Для проверки на Ubuntu

```


* **Просмотр логов конкретного упавшего компонента сети:**
```bash
kubectl logs -n calico-system deployment/calico-kube-controllers
kubectl logs -n calico-system -l k8s-app=calico-node -c calico-node --tail=50

```


* **Поды застряли в состоянии `ContainerCreating` или `Pending` после переустановки:**
Проверьте, не остались ли на узлах старые конфигурационные CNI-файлы в каталоге `/etc/cni/net.d/`. Очистите папку на узлах и перезапустите поды.
