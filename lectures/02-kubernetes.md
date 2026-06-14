# ☸️ Лекция 2: Kubernetes - Архитектура и Компоненты

## Оглавление
1. [Что такое Kubernetes?](#что-такое-kubernetes)
2. [Архитектура Kubernetes](#архитектура-kubernetes)
3. [Компоненты Control Plane](#компоненты-control-plane)
4. [Компоненты Worker Node](#компоненты-worker-node)
5. [Основные объекты Kubernetes](#основные-объекты-kubernetes)
6. [Pod - основная единица](#pod---основная-единица)
7. [Сравнение Swarm vs K8s](#сравнение-swarm-vs-k8s)

---

## Что такое Kubernetes?

**Kubernetes (K8s)** — это **production-grade система оркестрации контейнеров** с открытым исходным кодом.

### История:
- 2014: Google открыл Kubernetes (основан на Borg/Omega)
- 2015: Передан CNCF (Cloud Native Computing Foundation)
- 2019+: De-facto стандарт для оркестрации контейнеров

### Ключевые характеристики:
- ✅ **Автоматизация развертывания** контейнеров
- ✅ **Масштабирование** приложений
- ✅ **Self-healing** и восстановление
- ✅ **Load balancing** и service discovery
- ✅ **Rolling updates** без downtime
- ✅ **Resource management** и scheduling
- ✅ **Declarative configuration** (Infrastructure as Code)
- ✅ **Multi-cloud** поддержка

---

## Архитектура Kubernetes

```
┌──────────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Control Plane (Master)                       │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │                                                            │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐   │   │
│  │  │  API Server │  │  Scheduler   │  │ Controller Mgr │   │   │
│  │  └─────────────┘  └──────────────┘  └────────────────┘   │   │
│  │                                                            │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │         etcd (Distributed Database)              │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          │                                       │
│         ┌────────────────┼────────────────┐                      │
│         │                │                │                      │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐              │
│  │ Worker Node1│  │ Worker Node2│  │ Worker Node3│              │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤             │
│  │ kubelet      │  │ kubelet      │  │ kubelet      │             │
│  │ kube-proxy   │  │ kube-proxy   │  │ kube-proxy   │             │
│  │ Container RT │  │ Container RT │  │ Container RT │             │
│  │              │  │              │  │              │             │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │             │
│  │ │  Pod 1   │ │  │ │  Pod 2   │ │  │ │  Pod 3   │ │             │
│  │ │Container │ │  │ │Container │ │  │ │Container │ │             │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │             │
│  │              │  │              │  │              │             │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │             │
│  │ │  Pod 4   │ │  │ │  Pod 5   │ │  │ │  Pod 6   │ │             │
│  │ │Container │ │  │ │Container │ │  │ │Container │ │             │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Два главных компонента:

1. **Control Plane** (Master)
   - Управляет состоянием кластера
   - Принимает решения о расписании (scheduling)
   - Обеспечивает высокую доступность

2. **Worker Nodes**
   - Запускают контейнеры в Pods
   - Сообщают о своем состоянии
   - Подчиняются указаниям Control Plane

---

## Компоненты Control Plane

### 1. **kube-apiserver** (API Server)
```
Роль: Главный компонент K8s

Функции:
- REST API для всех операций
- Аутентификация и авторизация
- Валидация запросов
- Персистентное хранилище (etcd)
- Вся коммуникация идет через API Server

Команда: kubectl → API Server → etcd
```

### 2. **etcd** (Distributed Database)
```
Роль: Хранилище состояния всего кластера

Хранит:
- Конфигурацию всех объектов
- Состояние Pods, Services, etc.
- Secrets и ConfigMaps
- Информацию о узлах

Характеристики:
- Key-value хранилище
- Консистентное распределенное хранилище
- Бэкапы критичны!
- Обычно 3-5 инстансов для HA
```

### 3. **kube-scheduler** (Scheduler)
```
Роль: Распределение Pods на Worker Nodes

Процесс:
1. Получает новый Pod из API Server
2. Анализирует требования (resources, affinity, etc.)
3. Оценивает доступные узлы
4. Выбирает лучший узел
5. Присваивает Pod узлу

Алгоритм:
1. Filtering - исключает несовместимые узлы
2. Scoring - оценивает оставшиеся узлы
3. Selection - выбирает узел с лучшей оценкой
```

### 4. **kube-controller-manager** (Controllers)
```
Роль: Управление контроллерами кластера

Включает несколько контроллеров:
- Node Controller       - мониторит узлы
- Job Controller        - управляет Job объектами
- Service Controller    - создает LoadBalancers
- Deployment Controller - управляет Deployments
- StatefulSet Controller- управляет StatefulSets
- DaemonSet Controller  - запускает на всех узлах
- ReplicaSet Controller - поддерживает кол-во реплик

Основной цикл контроллера:
1. Observe   - смотрим текущее состояние
2. Analyze   - сравниваем с desired state
3. Act       - выполняем действия для выравнивания
```

### 5. **cloud-controller-manager** (Cloud Integration)
```
Роль: Интеграция с cloud providers

Управляет:
- Node Controller - удаление missing nodes из cloud
- Route Controller - network routes
- Service Controller - cloud LoadBalancers

Используется: AWS, Azure, GCP, etc.
```

---

## Компоненты Worker Node

### 1. **kubelet** (Node Agent)
```
Роль: Agent на каждом узле, обеспечивает запуск контейнеров

Функции:
- Регистрация узла в API Server
- Получение Pod specifications
- Запуск и управление контейнерами
- Мониторинг здоровья Pod
- Проверка лiveness/readiness probes
- Логирование и метрики узла
- Очистка остановленных контейнеров

Процесс:
1. Слушает API Server для новых Pods
2. Загружает нужные образы
3. Инструктирует container runtime запустить контейнер
4. Мониторит состояние
5. Сообщает статус обратно в API Server
```

### 2. **kube-proxy** (Network Proxy)
```
Роль: Сетевой прокси для Services

Функции:
- Внедрение Service abstractions
- Load balancing трафика к backend Pods
- Управление iptables rules (обычно)
- NetworkPolicy enforcement

Режимы:
- userspace (медленный, старый)
- iptables (быстрый, стандартный)
- ipvs (самый быстрый)
- ebpf (будущее)

Пример:
Client → Service IP:Port → kube-proxy → iptables rules → Pod IP:Port
```

### 3. **Container Runtime**
```
Роль: Запуск контейнеров

Опции:
- containerd (стандарт)
- Docker (уже не стандарт)
- CRI-O
- Podman
- gVisor
- kata

Интерфейс: CRI (Container Runtime Interface)
```

---

## Основные объекты Kubernetes

### 1. **Pod** - Основная единица
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

### 2. **Deployment** - Управление репликами
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
```

### 3. **Service** - Сетевой доступ
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### 4. **ConfigMap** - Конфигурация
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://db:5432"
  LOG_LEVEL: "info"
```

### 5. **Secret** - Секретные данные
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: "super-secret"
  username: "admin"
```

### 6. **StatefulSet** - Stateful приложения
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: "db"
  replicas: 3
  selector:
    matchLabels:
      app: db
```

### 7. **DaemonSet** - На каждом узле
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitor
```

### 8. **Job** - One-time задачи
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-image
```

### 9. **Ingress** - Внешний доступ
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

---

## Pod - основная единица

### Что такое Pod?
```
Pod = Wrapper вокруг одного или больше контейнеров

┌─────────────────────────────┐
│          Pod                 │
├─────────────────────────────┤
│  ┌───────────────────────┐  │
│  │   Container 1         │  │
│  │   (обычно один)       │  │
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │   Container 2 (rare)  │  │
│  └───────────────────────┘  │
│                              │
│  Общее в Pod:                │
│  - Network namespace (IP)    │
│  - Storage volumes           │
│  - Lifecycle                 │
└─────────────────────────────┘
```

### Почему не просто контейнеры?
```
1. Kubernetes управляет Pods, а не контейнерами
2. Абстракция выше контейнеров
3. Pod может содержать sidecar контейнеры
4. Подготовка к микросервисам
```

### Typicql Pod структура:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  labels:
    app: web
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db-url
    volumeMounts:
    - name: data
      mountPath: /app/data
  
  # Sidecar контейнер для логирования
  - name: logging
    image: logging-sidecar:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
  
  volumes:
  - name: data
    emptyDir: {}
  - name: log-volume
    emptyDir: {}
```

---

## Lifecycle объектов K8s

```
Pod Lifecycle:

Pending → Running → Succeeded/Failed
   ↓        ↓
 Waiting  Container
 (image  Starting/
  pull)   Running


Deployment Lifecycle:

Desired → Scheduling → Running → (Updates/Rollback)
   ↓         ↓            ↓
Create   Pods are      Pods are
object   assigned      running
         to nodes
```

---

## Сравнение Swarm vs K8s

| Параметр | Swarm | Kubernetes |
|----------|-------|------------|
| **Установка** | 1 команда | Сложное развертывание |
| **Масштаб** | До 1000 узлов | До 5000+ узлов |
| **CLI** | docker service | kubectl |
| **Конфиги** | docker service create | YAML files |
| **Networking** | Встроенный overlay | Гибкая подсистема |
| **Storage** | Ограниченный | Полная подсистема |
| **Learning Curve** | Пологая | Крутая |
| **Production Adoption** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Ecosystem** | Мал | Огромный |
| **Self-healing** | ✅ Базовое | ✅ Продвинутое |
| **Rolling Updates** | ✅ Есть | ✅ Гибкие |
| **Resource Limits** | ✅ Есть | ✅ Полная система |

---

## Резюме

### Kubernetes это:
- 🎯 **Production-grade** оркестрация контейнеров
- 📦 **Declarative** управление через YAML
- 🔄 **Self-healing** и автоматическое восстановление
- 📈 **Масштабируемо** на тысячи узлов
- 🌍 **Multi-cloud** поддержка
- 🛠️ **Огромная экосистема** инструментов
- 📚 **Кривая обучения** крутая, но оно того стоит

### Ключевые концепции:
1. **Control Plane** - мозг кластера
2. **Worker Nodes** - рабочие машины
3. **Pods** - основные единицы
4. **Services** - сетевая абстракция
5. **Deployments** - управление репликами
6. **Declarative Config** - описываем желаемое состояние
7. **Self-healing** - K8s исправляет проблемы

---

**Дальше:** 👉 Практические занятия по Kubernetes