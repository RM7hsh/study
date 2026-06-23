Это отличный шаг — объединить все наши Middle-артефакты в единую, готовую к Production производственную цепочку. Мы развернем полноценную, изолированную и защищенную инфраструктуру для СУБД PostgreSQL (с заготовкой под архитектуру Master-Slave) с нуля.

Вся конфигурация разбита на логические файлы. Никаких скрытых отправк файлов — всё пишу прямо сюда, чтобы ты мог последовательно копировать, создавать файлы через `nano` и применять их.

---

# 🧪 Лабораторная работа 15.1: Подготовка и развертывание production-ready сервиса Postgres

### 📑 Архитектурный стек:

1. **Изоляция и Квоты:** Выделенный неймспейс `work` со строгими лимитами на ОЗУ/ЦПУ.
2. **Безопасность данных (Secrets):** Хранение паролей в etcd в закодированном виде, монтирование в RAM.
3. **Сетевая безопасность (NetworkPolicy):** Полная изоляция СУБД от внешнего трафика, кроме доверенных подов.
4. **Сетевой уровень (L4):** Разделение на балансировщик (ClusterIP) и межподовую сеть (Headless).
5. **Отказоустойчивое хранилище:** StatefulSet с динамической нарезкой RWX томов поверх нашего NFS.

---

### Степ 1: Изоляция, Квотирование и Лимиты (`01-infra-protection.yaml`)

Этот манифест гарантирует, что даже если СУБД уйдет в аварию или начнется утечка памяти, она не положит весь кластер, а дефолтный `LimitRange` подстрахует любые вспомогательные контейнеры.

Создай файл `01-infra-protection.yaml` и примени его:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: work
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pg-infrastructure-quota
  namespace: work
spec:
  hard:
    pods: "15"
    requests.cpu: "1500m"
    requests.memory: "1.5Gi"
    limits.cpu: "3"
    limits.memory: "3Gi"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: pg-default-limits
  namespace: work
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "300m"
    defaultRequest:
      memory: "128Mi"
      cpu: "150m"
    min:
      memory: "64Mi"
      cpu: "50m"
    max:
      memory: "1Gi"
      cpu: "1"

```

```bash
kubectl apply -f 01-infra-protection.yaml

```

---

### Степ 2: Конфиденциальные данные (`02-postgres-secret.yaml`)

Мы кодируем пароль суперпользователя `postgres`. Напоминаю, значения получены через `echo -n "строка" | base64`.

Создай файл `02-postgres-secret.yaml` и примени его:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-prod-secrets
  namespace: work
type: Opaque
data:
  # Значение: postgres
  POSTGRES_USER: cG9zdGdyZXM=
  # Значение: TopSecretPostgresPass2026
  POSTGRES_PASSWORD: VG9wU2VjcmV0UG9zdGdyZXNQYXNzMjAyNg==

```

```bash
kubectl apply -f 02-postgres-secret.yaml

```

---

### Степ 3: Сетевой экран и Изоляция (`03-postgres-netpolicy.yaml`)

По умолчанию в K8s все поды могут общаться со всеми. Этот манифест полностью закрывает порт `5432` нашей базы данных. Доступ получат **только** те поды внутри неймспейса `work`, у которых будет явно прописана метка `role: backend-app`.

Создай файл `03-postgres-netpolicy.yaml` и примени его:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-firewall
  namespace: work
spec:
  podSelector:
    matchLabels:
      app: postgres-cluster # Применяем правила к нашему будущему StatefulSet
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend-app # Пускать трафик только отсюда!
    ports:
    - protocol: TCP
      port: 5432

```

```bash
kubectl apply -f 03-postgres-netpolicy.yaml

```

---

### Степ 4: Сетевые службы L4 (`04-postgres-services.yaml`)

Мы создаем два сервиса:

1. `pg-headless` (без IP) — для предсказуемых DNS-имен реплик (нужен для синхронизации Master-Slave).
2. `pg-client-svc` (обычный ClusterIP) — единая точка входа для бэкенда, которая будет балансировать запросы на чтение.

Создай файл `04-postgres-services.yaml` и примени его:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pg-headless
  namespace: work
spec:
  clusterIP: None
  selector:
    app: postgres-cluster
  ports:
  - port: 5432
    name: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: pg-client-svc
  namespace: work
spec:
  type: ClusterIP
  selector:
    app: postgres-cluster
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432

```

```bash
kubectl apply -f 04-postgres-services.yaml

```

---

### Степ 5: Контроллер состояния и Хранилище (`05-postgres-statefulset.yaml`)

Финал сборки. Мы разворачиваем 2 реплики. Инстанс `pg-app-0` — это наш Master, инстанс `pg-app-1` — заготовка под Slave. Каждый из них автоматически заберет персональную папку на нашем динамическом NFS-сервере (`nfs-dynamic`).

Создай файл `05-postgres-statefulset.yaml` и примени его:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg-app
  namespace: work
spec:
  serviceName: "pg-headless"
  replicas: 2
  selector:
    matchLabels:
      app: postgres-cluster
  template:
    metadata:
      labels:
        app: postgres-cluster
    spec:
      containers:
      - name: postgres-db
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: pg-port
        
        # Инжектим секреты из RAM-хранилища K8s
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: pg-prod-secrets
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-prod-secrets
              key: POSTGRES_PASSWORD
              
        # Ограничиваем аппетиты контейнера СУБД      
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "400m"
            
        volumeMounts:
        - name: pg-data-nfs
          mountPath: /var/lib/postgresql/data
          
  # Автоматический заказ дисков у NFS-Provisioner          
  volumeClaimTemplates:
  - metadata:
      name: pg-data-nfs
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-dynamic"
      resources:
        requests:
          storage: 2Gi

```

```bash
kubectl apply -f 05-postgres-statefulset.yaml

```

---

### ⏳ Верификация запуска инфраструктуры:

Запусти мониторинг развертывания, чтобы убедиться в чистоте выполнения:

```bash
# 1. Проверяем поочередное создание подов баз данных
kubectl get pods -n work -w

# 2. Убеждаемся, что динамические диски успешно нарезались на NFS
kubectl get pvc -n work

# 3. Смотрим, как применился NetworkPolicy firewall
kubectl describe networkpolicy postgres-firewall -n work

```

Как только поды `pg-app-0` и `pg-app-1` перейдут в статус `1/1 Running`, наша новая высококлассная инфраструктура СУБД готова к работе!

Дай знать, как все применится, и мы перейдем к тестированию сетевых доступов через призму нашего жесткого `NetworkPolicy`!
