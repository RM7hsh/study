# 🧪 Lab 9: Динамическое хранилище через собственный NFS Server и StorageClass

## 📋 Обзор

В этой лабораторной работе мы развернем полноценный NFS-сервер (в качестве примера выделим под него отдельную машину или одну из нод), свяжем его с вашим DNS Unbound, установим в Kubernetes специальный контроллер — **NFS Subdir External Provisioner**, и создадим **StorageClass**.

После этого разработчикам больше не придется просить админа создать PV вручную. Достаточно будет просто создать PVC — и кластер сам создаст папку на NFS-сервере и подмонтирует её.

## 🎯 Цели

* Развернуть и настроить NFS-сервер на Rocky Linux 10.
* Добавить DNS-запись для хранилища в Unbound DNS.
* Настроить ноды кластера для работы с NFS.
* Развернуть автоматический провижонер томов в пространстве имен `work`.
* Проверить динамическое создание PV при запросе PVC.

---

## 📝 Пошаговые инструкции

### Шаг 1: Настройка NFS-сервера (Выполняется на сервере хранения)

*Если у тебя есть отдельная ВМ — делай там. Если нет, для тестов можно поднять NFS-сервер прямо на `k8s-master-01` (но в проде так не делают из-за дисковой нагрузки).*

```bash
# 1. Установка пакетов NFS
sudo dnf install -y nfs-utils

# 2. Создание директории, которую будем шерить в кластер
sudo mkdir -p /srv/nfs/k8s-storage
sudo chmod 777 /srv/nfs/k8s-storage

# 3. Настройка прав экспорта (/etc/exports)
# Разрешаем доступ всей нашей подсети 192.168.77.0/24
echo "/srv/nfs/k8s-storage 192.168.77.0/24(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports

# 4. Запуск и включение службы
sudo systemctl daemon-reload
sudo systemctl enable --now nfs-server

# 5. Проверка экспорта
sudo showmount -e localhost

```

---

### Шаг 2: Интеграция с DNS Unbound

Зайди на свой DNS-сервер (`192.168.77.7`) и добавь `A-запись` для твоего NFS-сервера.
Например, если NFS-сервер имеет IP `192.168.77.190` (или IP мастера, если развернул на нем):

```text
nfs.work.local  IN A  192.168.77.190

```

*Убедись, что с мастер-ноды команда `ping nfs.work.local` успешно разрешается в правильный IP.*

---

### Шаг 3: Подготовка ВСЕХ нод Kubernetes (Критически важный шаг)

Контейнеры не умеют монтировать NFS напрямую, если на самом хосте (ноде Rocky Linux) нет нужных утилит. **Выполни эту команду на ВСЕХ 3 мастерах и 2 воркерах:**

```bash
sudo dnf install -y nfs-utils

```

*Если этого не сделать, поды, использующие NFS, намертво зависнут в статусе `ContainerCreating` с ошибкой протокола.*

---

### Шаг 4: Развертывание NFS-Provisioner в кластере

Мы развернем автоматический провижонер томов. Создай рабочую директорию на мастере `mkdir -p ~/k8s-lab/lab9 && cd ~/k8s-lab/lab9`.

Создай единый манифест для деплоя провижонера `nfs-provisioner.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: work
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: work
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: work
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: "nfs.work.local" # Твое DNS имя из Unbound!
            - name: NFS_PATH
              value: /srv/nfs/k8s-storage
      volumes:
        - name: nfs-client-root
          nfs:
            server: "nfs.work.local"
            path: /srv/nfs/k8s-storage

```

Примени манифест:

```bash
kubectl apply -f nfs-provisioner.yaml

# Проверяем, что под провижонера успешно запустился
kubectl get pods -l app: nfs-client-provisioner

```

---

### Шаг 5: Создание StorageClass (Инфраструктурный уровень)

Теперь создадим абстракцию `StorageClass`. Она указывает кластеру: «Если кто-то просит класс хранилища `nfs-dynamic`, зови наш NFS-провижонер».

Создай файл `storage-class.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-dynamic
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # Должно совпадать с PROVISIONER_NAME из Деплоя
reclaimPolicy: Delete # При удалении PVC папка на NFS будет автоматически зачищаться
volumeBindingMode: Immediate

```

Примени:

```bash
kubectl apply -f storage-class.yaml

# Проверяем созданные классы хранилищ
kubectl get sc

```

---

### Шаг 6: Тестирование динамического PVC (Магия автоматизации)

Теперь проверим, как работает Middle-DevOps подход. Мы создадим обычный PVC, не описывая под него никакой PV.

Создай файл `test-dynamic-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc-test
  namespace: work
spec:
  accessModes:
    - ReadWriteMany # Огромный плюс NFS! Один диск могут читать/писать МНОГО подов на РАЗНЫХ нодах!
  storageClassName: nfs-dynamic # Вызываем наш новый StorageClass
  resources:
    requests:
      storage: 1Gi

```

Примени манифест:

```bash
kubectl apply -f test-dynamic-pvc.yaml

```

**Мгновенно проверяем статус:**

```bash
kubectl get pvc dynamic-pvc-test -n work
kubectl get pv

```

**Что произошло:** Статус PVC мгновенно стал `Bound`. Если ты введешь `kubectl get pv`, то увидишь, что Kubernetes автоматически сгенерировал PV с именем `pvc-xxxxxxxx-xxxx-...`, выделил ему 1Gi и связал с твоим клеймом.

Если зайти на сам **NFS-сервер** в папку `/srv/nfs/k8s-storage/`, ты увидишь там автоматически созданную директорию формата `work-dynamic-pvc-test-pvc-xxxxxxx`.

---

## 🔍 Траблшутинг NFS на Middle-уровне

Если PVC висит в `Pending`, или под, использующий PVC, не запускается:

1. **Проблема:** Под провижонера падает с ошибкой `Permission denied`.
* *Решение:* На NFS-сервере в файле `/etc/exports` обязательно должны быть опции `no_root_squash` и `no_subtree_check`, а на саму папку должны стоять полные права (`chmod 777`).


2. **Проблема:** Под приложения висит в `ContainerCreating` с ошибкой `mount failed: Connection refused`.
* *Решение:* Либо на нодах забыли поставить пакет `nfs-utils`, либо локальный фаервол хоста блокирует порты NFS. Для проверки на NFS-сервере временно отключи firewalld (`sudo systemctl stop firewalld`).



---

## ✅ Критерии успешного выполнения лабораторной работы

1. Поднят рабочий NFS-сервер, имя которого резолвится через ваш Unbound DNS.
2. В неймсейсе `work` успешно работает под `nfs-client-provisioner`.
3. Создан глобальный объект `StorageClass` с именем `nfs-dynamic`.
4. Создание PVC типа `ReadWriteMany` автоматически генерирует соответствующий `PersistentVolume` без ручного вмешательства.

---

Это мощный прорыв в архитектуре твоего стенда! Теперь наш кластер обладает общим сетевым дисковым пространством с поддержкой режима **ReadWriteMany (RWX)**.

Мы полностью готовы к работе со сложными приложениями. Переходим к **Lab 10: StatefulSets и развертывание масштабируемых баз данных**? Жду отмашку!
