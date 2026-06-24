Наш HA-кластер теперь полностью защищен изнутри: и сеть, и доступы (RBAC), и ресурсы (Quotas) находятся под жестким контролем. Но продакшн-инженер не может спать спокойно, если он не видит, что происходит внутри системы в реальном времени. Нам нужна **Обсервабилити (Observability) и мониторинг**.

В мире Kubernetes стандартом де-факто является связка **Prometheus** (сбор и хранение метрик методом пуллинга/scraping) и **Grafana** (визуализация и красивые дашборды). Но перед тем как разворачивать тяжелые комбайны, мидл-инженер должен активировать базовый движок кластера — **Metrics Server**, который включает легендарные команды `kubectl top`.

Переходим к **Lab 17**. Все наши приложения в пространстве имен `work` сейчас станут полностью прозрачными.

---

# 🧪 Lab 17: Мониторинг, обсервабилити и сбор метрик (Metrics Server, Prometheus и Grafana)

## 📋 Обзор

Мониторинг в Kubernetes работает по схеме сбора метрик с системных компонентов (`kubelet`, `containerd`, `coredns`) и самих приложений. Архитектура выглядит так:

В этой лабораторной работе мы сначала развернем bare-metal версию `Metrics Server` и настроим его для обхода проверок самоподписанных сертификатов наших нод Rocky Linux. Затем развернем легковесный стек Prometheus внутри пространства имен `work`, настроим сбор метрик с наших PostgreSQL и Nginx, а в конце подключим Grafana для вывода графиков утилизации памяти и CPU.

## 🎯 Цели

* Развернуть и кастомизировать `Metrics Server` под bare-metal HA-кластер.
* Освоить команды оперативного дебага ресурсов (`kubectl top nodes / pods`).
* Развернуть Prometheus и настроить сбор метрик приложений в namespace `work`.
* Развернуть Grafana, связать её с Prometheus (Data Source) и проанализировать графики.

---

## 📝 Пошаговые инструкции

### Шаг 1: Развертывание Metrics Server (Включаем команды `kubectl top`)

Metrics Server собирает метрики утилизации ресурсов с каждой ноды (через встроенный в kubelet `cAdvisor`) и предоставляет их для K8s API.

Выполнить на `k8s-master-01`:

```bash
mkdir -p ~/k8s-lab/lab17 && cd ~/k8s-lab/lab17

# 1. Скачиваем официальный манифест Metrics Server
#curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.2/components.yaml
```

🚨 **Внимание! Критический bare-metal хак (Мидл-траблшутинг):**
По умолчанию Metrics Server требует, чтобы сертификаты на нодах кластера были подписаны доверенным публичным CA. На нашем локальном стенде сертификаты самоподписанные (`kubeadm`), поэтому сервер упадет с ошибкой `x509: certificate signed by unknown authority`.

Исправим это. Открой скачанный файл `components.yaml` через `nano`, найди блок `Deployment` с именем `metrics-server` и в секцию `args` контейнера добавь флаг **`--kubelet-insecure-tls`**:

```yaml
# Фрагмент файла components.yaml
spec:
  template:
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # ➕ ДОБАВЛЯЕМ ЭТУ СТРОКУ!
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.1
        name: metrics-server

```

Примени манифест:

```bash
kubectl apply -f components.yaml

# Ждем запуска (он разворачивается в kube-system)
kubectl get pods -n kube-system -l k8s-app=metrics-server

```

---

### Шаг 2: Тестирование оперативного мониторинга

Подожди 30–60 секунд, пока Metrics Server соберет первую порцию данных из рантайма `containerd`. Теперь мы можем на лету смотреть утилизацию «железа»:

```bash
# 1. Проверяем нагрузку на физические ноды кластера
kubectl top nodes

# 2. Проверяем, сколько конкретно процессора и ОЗУ жрут наши поды в неймсейсе work!
kubectl top pods -n work

```

*Команда выдаст точные данные в миллиядрах CPU (`m`) и Мегабайтах (`Mi`). Теперь ты сразу увидишь, если какая-то база данных Postgres начинает упираться в свои limits.*

---

### Шаг 3: Развертывание Prometheus (Слой хранения метрик)

Metrics Server хранит данные только в памяти и за пару минут. Для долгосрочных графиков развернем Prometheus. Мы упакуем его в наш рабочий namespace `work`.

Создай файл `prometheus-core.yaml`. Чтобы не плодить лишние тома, Prometheus будет использовать наш крутой динамический `StorageClass: nfs-dynamic` из Lab 9!

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  namespace: work
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: work
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
          - "--config.file=/etc/prometheus/prometheus.yml"
          - "--storage.tsdb.path=/prometheus"
        ports:
        - containerPort: 9090
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: storage-volume
          mountPath: /prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-server-conf
      - name: storage-volume
        persistentVolumeClaim:
          claimName: prometheus-pvc # Закажем диск у нашего NFS автоматизатора
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: work
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-dynamic # Наш рабочий NFS-класс
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: work
spec:
  type: ClusterIP
  ports:
  - port: 9090
    targetPort: 9090
  selector:
    app: prometheus-server

```

Примени манифест:

```bash
kubectl apply -f prometheus-core.yaml

# Убеждаемся, что PVC перешел в Bound, а под запустился
kubectl get pvc,pods -n work -l app=prometheus-server

```

---

### Шаг 4: Развертывание Grafana (Визуализация)

Grafana подключится к нашему `prometheus-service` по внутреннему DNS-имени и начнет рисовать графики. Выведем её наружу через `NodePort`, чтобы ты мог зайти в неё со своего MacBook.

Создай файл `grafana.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
  namespace: work
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:10.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "150m"
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: work
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 32300 # Будет слушать на всех нодах Rocky Linux
  selector:
    app: grafana

```

Примени:

```bash
kubectl apply -f grafana.yaml

# Проверяем готовность
kubectl get pods -n work -l app=grafana

```

---

### Шаг 5: Вход в систему и подключение данных

1. Открой браузер на своем MacBook/ПК и перейди по адресу: `http://192.168.77.181:32300` (IP любой мастер или воркер ноды).
2. Перед тобой откроется приветственное окно Grafana. Дефолтный логин/пароль: **`admin` / `admin**`. Система попросит сменить пароль при первом входе.
3. Перейди в меню: **Connections -> Data Sources -> Add data source**.
4. Выбери **Prometheus**.
5. В поле **URL** укажи внутренний домен нашего сервиса (благодаря CoreDNS это работает идеально):
```text
http://prometheus-service.work.svc.cluster.local:9090

```


6. Прокрути вниз и нажми **Save & Test**. Если все сделано верно, загорится зеленая надпись *"Data source is working"*.

Теперь ты можешь нажать на кнопку *Explore* и писать кастомные метрики (например, `container_memory_usage_bytes`), собирая полную статистику по приложениям в реальном времени.

---

## 🔍 Инженерный траблшутинг мониторинга (Middle-level)

1. **Команда `kubectl top` выдает ошибку `Metrics not available`:**
* *Причина:* Metrics Server только запущен и еще не успел провести первый раунд Scraping-опроса подов через TLS. Подожди 1 минуту. Если ошибка не исчезает — выполни `kubectl logs -n kube-system -l k8s-app=metrics-server` и убедись, что ты добавил флаг `--kubelet-insecure-tls`.


2. **Grafana теряет данные после перезапуска пода:**
* *Решение:* Для Grafana мы не настраивали PVC в этой лабораторной работе (она хранит свои настройки во внутренней sqlite-базе прямо внутри контейнера). Чтобы сделать Grafana персистентной, к ней добавляется блок `volumeMounts` на путь `/var/lib/grafana` и подвязывается PVC от нашего `StorageClass: nfs-dynamic` точно так же, как мы сделали для Prometheus.



---

## ✅ Критерии успешного выполнения лабораторной работы

1. Команда `kubectl top nodes` и `kubectl top pods -n work` возвращает актуальные метрики нагрузки.
2. Prometheus успешно развернут в неймсейсе `work` и примонтировал динамический RWX-диск с NFS.
3. Grafana доступна с внешней машины на порту `32300`.
4. Внутри веб-интерфейса Grafana успешно зарегистрирован Data Source, смотрящий на Prometheus.

---

Обсервабилити-слой готов! Наш стек теперь полностью укомплектован. Мы выходим на финишную прямую Блока 6.

Последний фундаментальный шаг — это автоматизация пакетирования приложений. Переходим к **Lab 18: Helm — пакетный менеджер Kubernetes**. Мы научимся упаковывать все наши разрозненные YAML-файлы (Deployments, Services, ConfigMaps, Настройки ресурсов) в один переиспользуемый шаблон (Chart), устанавливать его одной командой и управлять версиями релизов.

Дай знать, как Grafana подружится с Prometheus, и мы добьем этот курс!
