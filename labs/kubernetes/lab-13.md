Добро пожаловать в заключительный, самый мощный блок нашего курса — **Блок 6: Advanced Topics**. Здесь мы переходим от классического обслуживания пользовательского трафика к автоматизации самой инфраструктуры, регламентных задач и системных утилит.

В реальном Production администратору постоянно нужно решать две задачи:

1. Запустить какую-то утилиту (например, сборщик логов Fluentd или агент мониторинга Promtail) **строго на каждом сервере кластера**.
2. Запустить разовую задачу (миграцию базы данных) или регулярную таску (бэкап базы данных каждую ночь в 3:00) и положить результат в наше NFS-хранилище.

Для этого в Kubernetes существуют три контроллера: **DaemonSet**, **Job** и **CronJob**. Работать продолжаем строго в пространстве имен `work`.

---

# 🧪 Lab 13: Автоматизация инфраструктуры (DaemonSets, Jobs и CronJobs)

## 📋 Обзор

В этой лабораторной работе мы развернем системный агент через `DaemonSet`, который автоматически размножится на все ваши Rocky Linux воркеры. Затем мы настроим разовую утилиту `Job` для проверки связности сети, а в конце закроем критический бэкап-слой: напишем полноценный `CronJob`, который будет по расписанию подключаться к нашей базе PostgreSQL (из Lab 10) и сливать дампы в динамический NFS-том.

## 🎯 Цели

* Изучить механику работы `DaemonSet` и его отличие от стандартных Deployments.
* Написать разовый таск (`Job`) с контролем успешности выполнения (`completions`, `backoffLimit`).
* Создать регламентный бэкап-скрипт через `CronJob`.
* Связать CronJob с динамическим `StorageClass: nfs-dynamic` для сохранения персистентных дампов.

---

## 📝 Пошаговые инструкции

### Шаг 1: Развертывание DaemonSet (Агент сбора логов)

`DaemonSet` гарантирует, что на выбранных (или на вообще всех) нодах кластера будет запущено ровно по одной реплике указанного пода. Если в кластер добавляется новая физическая нода `k8s-worker-03`, DaemonSet сам мгновенно развернет там свой под.

Выполнить на `k8s-master-01`:

```bash
mkdir -p ~/k8s-lab/lab13 && cd ~/k8s-lab/lab13

# Создаем манифест daemonset-log.yaml
cat > daemonset-log.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-log-collector
  namespace: work
  labels:
    tier: infrastructure
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16-1
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: varlog
          mountPath: /var/log # Монтируем системную папку логов самой ноды Rocky Linux
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

kubectl apply -f daemonset-log.yaml

```

---

### Шаг 2: Анализ распределения DaemonSet по нодам

Давай посмотрим, сколько реплик создал кластер и где они приземлились:

```bash
kubectl get daemonset fluentd-log-collector -n work
kubectl get pods -n work -l app=log-agent -o wide

```

**Мидл-анализ планирования (Scheduling):**
Ты увидишь, что поды запустились ровно на воркер-нодах (`k8s-worker-01` и `k8s-worker-02`).

> *Вопрос на засыпку:* Почему поды не запустились на `k8s-master-01..03`?
> *Ответ:* По умолчанию на мастер-нодах Kubernetes висят запреты (**Taints**) `node-role.kubernetes.io/control-plane:NoSchedule`. DaemonSet уважает эти запреты и не трогает мастера, если явным образом не прописать блоки `tolerations` в манифесте.

---

### Шаг 3: Запуск разовой задачи (Job)

Обычный под при завершении процесса с `Exit Code 0` перейдет в статус `Completed`, но контроллер Deployment посчитает это аварией и попытается запустить его снова. Контроллер **Job**, наоборот, ожидает, что процесс выполнит работу и завершится.

Создадим задачу, которая симулирует проверку структуры таблиц или пинг нашей СУБД.

Создай файл `job-db-check.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-integrity-check
  namespace: work
spec:
  template:
    spec:
      containers:
      - name: checker
        image: alpine:latest
        command: ["/bin/sh", "-c"]
        # Симулируем работу утилиты проверки сети и БД, которая завершается успешно через 10 секунд
        args: ["echo 'Starting integrity validation...'; sleep 10; echo 'Validation SUCCESS!'; exit 0"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      restartPolicy: Never # Для Job политикой перезапуска может быть только Never или OnFailure
  backoffLimit: 3 # Количество попыток перезапуска контейнера, если скрипт вернет ошибку (Exit Code != 0)

```

Примени манифест:

```bash
kubectl apply -f job-db-check.yaml

# Наблюдаем за выполнением
kubectl get jobs -n work -w

```

Через 10 секунд колонка `COMPLETIONS` покажет `1/1`. Под перейдет в статус `Completed` и перестанет потреблять ресурсы процессора и памяти твоего кластера.

---

### Шаг 4: Настройка регламентных бэкапов через CronJob

Теперь объединим все наши знания: Стейтфул-базы (Lab 10), Динамический NFS (Lab 9) и шедулинг. Напишем **CronJob**, который будет каждую минуту (для тестов, в проде ставится `0 2 * * *` — каждую ночь в 2 часа) подключаться к нашей базе PostgreSQL `pg-app-0.pg-headless` по стабильному DNS, делать `pg_dump` и сохранять этот файл на сетевой NFS-диск.

Создай файл `cronjob-backup.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-nightly-backup
  namespace: work
spec:
  schedule: "*/1 * * * *" # Запуск КАЖДУЮ минуту (Канонический синтаксис Linux Crontab)
  concurrencyPolicy: Forbid # Запретить запуск нового бэкапа, если предыдущий еще не успел отработать
  successfulJobsHistoryLimit: 3 # Сколько успешных подов-логов хранить в истории
  failedJobsHistoryLimit: 1 # Сколько упавших хранить для дебага
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: pg-backup-agent
            image: postgres:15-alpine
            env:
            - name: PGPASSWORD
              value: "SuperSecretMidPass" # Пароль, который мы задали в Lab 10
            command: ["/bin/sh", "-c"]
            args:
            - >
              FILENAME="/backup/pg-dump-\$(date +%Y%m%d-%H%M%S).sql";
              echo "Starting database backup to NFS...";
              pg_dump -h pg-app-0.pg-headless.work.svc.cluster.local -U postgres -d postgres -f \$FILENAME;
              echo "Backup completed successfully! Saved as \$FILENAME";
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
          
          # Разрешаем монтировать наш общий динамический NFS том прямо внутрь бэкап-таски
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: dynamic-pvc-test # Используем наш готовый RWX NFS-клейм из Lab 9
          restartPolicy: OnFailure

```

Примени манифест:

```bash
kubectl apply -f cronjob-backup.yaml

# Проверяем статус шедулера
kubectl get cronjob postgres-nightly-backup -n work

```

**Проверка выполнения:**
Подожди 1-2 минуты. Введи команду `kubectl get jobs -n work`. Ты увидишь рождение подчиненных тасок вида `postgres-nightly-backup-28392180`.

А теперь иди на свой **NFS-сервер** (или в ту директорию хоста, куда примонтирован NFS) в каталог хранения:

```bash
ls -lh /srv/nfs/k8s-storage/work-dynamic-pvc-test-pvc-*/

```

Ты увидишь там свежие SQL-дампы нашей базы данных! Автоматизация сработала в полном объеме.

---

## 🔍 Инженерный траблшутинг (Middle-level)

1. **Множественные поды CronJob зависают в статусе `Error` или плодятся:**
* *Причина:* Если твоя база данных выросла, а бэкап выполняется дольше, чем интервал шедулера (например, бэкап идет 3 минуты, а CronJob настроен на запуск раз в минуту), они начнут накладываться друг на друга и забьют оперативную память нод.
* *Решение:* Всегда указывай директиву `concurrencyPolicy: Forbid`. Она заблокирует старт нового задания, пока старое не завершит работу.


2. **Как запустить CronJob вручную для проверки, не дожидаясь времени по расписанию?**
Мидл-лайфхак для дебага:
```bash
kubectl create job --from=cronjob/postgres-nightly-backup test-manual-run -n work

```


*Кластер мгновенно создаст разовый Job на основе шаблона твоего CronJob.*

---

## ✅ Критерии успешного выполнения лабораторной работы

1. `DaemonSet` успешно раскатал агенты на все доступные воркеры кластера.
2. `Job` успешно отработал проверку и перешел в финальный статус `Completed`.
3. Настроен автоматический `CronJob` с конкурентной политикой `Forbid`.
4. В твоем общем NFS-хранилище регулярно появляются файлы бэкапа базы данных Postgres.

---

Блок автоматизации инфраструктуры полностью закрыт! Это огромный шаг вперед.

Следующий этап по нашей программе — **Lab 14 (Lab 16 в исходном списке): Глубокий разбор Resource Limits, Requests и LimitRanges / ResourceQuotas**. Мы научимся жестко лимитировать не просто отдельные контейнеры, а выставим "потолок" ресурсов на всё пространство имен `work`, чтобы оно физически не могло утилизировать больше положенного.

Напиши, как дампы пойдут на NFS, и мы продолжим!
