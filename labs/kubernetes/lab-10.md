# 🧪 Lab 10: StatefulSets и развертывание баз данных (Stateful)

## 📋 Обзор

В отличие от веб-серверов (Stateless), поды баз данных не являются взаимозаменяемыми. Каждый под в **StatefulSet** имеет:

1. **Уникальный и предсказуемый индекс** (`pod-0`, `pod-1`, `pod-2`).
2. **Собственное изолированное хранилище**, которое жестко привязывается к этому индексу. При перезапуске `pod-1` всегда заберет именно диск `pod-1`.
3. **Стабильное DNS-имя**, обеспечиваемое через специальный **Headless Service**.

В этой работе мы развернем масштабируемую базу данных PostgreSQL (в режиме Master/Slave заготовок), используя наш `StorageClass: nfs-dynamic`.

```
                    [ Headless Service: pg-headless ]
                               /        \
                              v          v
                       [ pg-app-0 ]   [ pg-app-1 ]
                            |              |
                     ( PVC-pg-app-0 ) ( PVC-pg-app-1 )
                            \              /
                         [ Наш Динамический NFS ]

```

## 🎯 Цели

* Понять разницу между `Deployment` и `StatefulSet` для Middle-уровня.
* Развернуть **Headless Service** для создания стабильной сети.
* Создать манифест `StatefulSet` с использованием механизма `volumeClaimTemplates`.
* Изучить упорядоченный (Ordered) запуск подов и проверить автоматическую нарезку дисков на NFS.

---

## 📝 Пошаговые инструкции

### Шаг 1: Создание Headless-сервиса для сети

Обычный сервис ClusterIP балансирует трафик случайным образом. Базе данных это противопоказано — нам нужно уметь обращаться к конкретному мастеру или реплике. `Headless Service` не создает виртуальный IP, а просто возвращает через CoreDNS список IP-адресов всех подов напрямую.

Выполнить на `k8s-master-01`:

```bash
mkdir -p ~/k8s-lab/lab10 && cd ~/k8s-lab/lab10

# Создаем файл headless-svc.yaml
cat > headless-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: pg-headless
  namespace: work
spec:
  clusterIP: None # 🌟 ВОТ ОНА — магия Headless (IP отсутствует)
  selector:
    app: postgres-cluster
  ports:
  - port: 5432
    name: postgres
EOF

kubectl apply -f headless-svc.yaml

```

---

### Шаг 2: Создание манифеста StatefulSet с шаблоном дисков

Вместо привязки к одному конкретному PVC, мы используем `volumeClaimTemplates`. Каждый под, который будет рождаться в рамках этого сета, сам попросит у нашего NFS-провижонера персональный диск.

Создай файл `statefulset-pg.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg-app
  namespace: work
spec:
  serviceName: "pg-headless" # Связываем с нашим Headless-сервисом
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
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "SuperSecretMidPass"
        ports:
        - containerPort: 5432
          name: pg-port
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: pg-data-volume
          mountPath: /var/lib/postgresql/data # Куда складывать таблицы
  
  # 🌟 АВТОМАТИЧЕСКАЯ НАРЕЗКА ДИСКОВ ДЛЯ КАЖДОГО ПОДА
  volumeClaimTemplates:
  - metadata:
      name: pg-data-volume
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-dynamic" # Наш рабочий динамический класс!
      resources:
        requests:
          storage: 1Gi

```

Примени манифест:

```bash
kubectl apply -f statefulset-pg.yaml

```

---

### Шаг 3: Наблюдение за упорядоченным запуском (Middle-анализ)

В отличие от деплойментов, где все поды стартуют одновременно кашей, StatefulSet делает это строго по очереди. Запусти мониторинг:

```bash
kubectl get pods -n work -l app=postgres-cluster -w

```

**Что ты увидишь:**

1. Сначала создается под `pg-app-0`. Кластер ждет, пока он пройдет все проверки и перейдет в статус `Running`.
2. Только после полной готовности нулевого пода, планировщик начинает создавать `pg-app-1`.

Это гарантирует, что если у тебя кластер базы данных, реплики не начнут подниматься раньше, чем стартанет мастер.

---

### Шаг 4: Проверка созданных ресурсов и NFS-дисков

Давай проверим, что произошло с дисками в пространстве имен `work`:

```bash
# Проверяем созданные PVC
kubectl get pvc -n work

```

Ты увидишь два абсолютно разных PVC: `pg-data-volume-pg-app-0` и `pg-data-volume-pg-app-1`. Они завязаны на индивидуальные PV.

Если ты зайдешь на свой **NFS-сервер** в каталог `/srv/nfs/k8s-storage/`, то увидишь, что наш провижонер автоматически нарезал там **две разные папки** под каждый под.

---

### Шаг 5: Проверка предсказуемого DNS-имени

Благодаря Headless-сервису, CoreDNS теперь знает точные имена подов. Давай проверим это. Запустим наш отладочный под:

```bash
kubectl run dns-test --image=curlimages/curl --restart=Never -it -- /bin/sh

```

Внутри пода выполни проверку сетевых имен через `nslookup` (утилита доступна или проверим обычным пингом/curl):
*Так как в `curlimages/curl` нет nslookup, мы можем выполнить запрос к API или запустить под с `busybox` для чистоты эксперимента.* Давай для DNS-тестов запустим классический сетевой тулбокс:

```bash
# Выходим из curl-пода, если зашел в него (exit)
kubectl run net-toolbox --image=busybox:1.28 --restart=Never -it -- /bin/sh

```

**Выполняй внутри пода `net-toolbox`:**

```bash
# Проверяем имя Headless-сервиса (он вернет IP ВСЕХ подов базы данных)
nslookup pg-headless

# Проверяем имя конкретного ПЕРВОГО пода
nslookup pg-app-0.pg-headless

# Проверяем имя ВТОРОГО пода
nslookup pg-app-1.pg-headless

```

Ты увидишь, что DNS-сервер четко возвращает конкретный IP-адрес конкретного пода. Теперь реплики могут общаться с мастером по стабильному имени `pg-app-0.pg-headless` даже после перезапусков кластера!

---

## 🔍 Инженерный траблшутинг StatefulSet (Middle-level)

1. **Что будет, если уменьшить количество реплик (Scale Down)?**
Если ты изменишь `replicas` с 2 до 1, Kubernetes удалит под `pg-app-1`. Но **его PVC и папка на NFS останутся нетронутыми!** Это сделано специально для безопасности данных. Если это была случайность, при обратном масштабировании под подцепит свой старый диск без потери информации. Зачищать диски баз данных нужно только вручную.
2. **Под завис в `Terminating` при удалении StatefulSet:**
StatefulSet очень ревностно относится к идентичности. Если нода потеряла связь, K8s не удалит под сам, чтобы избежать ситуации, когда одна база данных пишет в один диск из двух мест (Split-Brain).
* *Решение:* Если ты уверен, что нода мертва, можно применить force-удаление:
`kubectl delete pod pg-app-1 -n work --force --grace-period=0`.



---

## ✅ Критерии успешного выполнения лабораторной работы

1. Развернут `Headless Service` с флагом `clusterIP: None`.
2. Поды PostgreSQL поднялись строго по очереди (`pg-app-0`, затем `pg-app-1`).
3. Механизм `volumeClaimTemplates` автоматически создал уникальные PVC для каждого инстанса БД.
4. Проверена сетевая доступность подов по их персональным доменным именам.
