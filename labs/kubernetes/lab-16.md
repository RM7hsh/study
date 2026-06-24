Безопасность процессов, сети и ресурсов настроена. Теперь пора разобраться с тем, кто имеет право всем этим управлять. В Kubernetes за это отвечает **RBAC (Role-Based Access Control)** — контроль доступа на основе ролей.

Главный инсайт Middle-инженера: **в Kubernetes нет объекта «User» (Пользователь) в базе данных etcd**. Кластер доверяет внешним сертификатам (X.509), подписанным корневым CA кластера, или токенам. Наша задача — сгенерировать такой сертификат для нового инженера, а затем жестко ограничить его полномочия внутри пространства имен `work`.

Переходим к **Lab 16**.

---

# 🧪 Lab 16: Управление доступом и безопасность через RBAC

## 📋 Обзор

Схема работы RBAC строится на трех кирпичиках:

1. **Subject** — кто запрашивает доступ (конкретный User, Группа или системный `ServiceAccount`).
2. **Role / ClusterRole** — какие действия (`verbs`: get, list, update, delete) над какими ресурсами (`resources`: pods, services, deployments) разрешены. `Role` действует внутри неймспейса, `ClusterRole` — на уровне всего кластера.
3. **RoleBinding / ClusterRoleBinding** — клей, который связывает субъекта с ролью.

В этой лабораторной работе мы создадим учетную запись для разработчика `developer-ivan`, выпишем ему персональные SSL-сертификаты через встроенный API Kubernetes, создадим ограниченную роль для пространства имен `work` и проверим защиту от несанкционированных действий.

## 🎯 Цели

* Сгенерировать приватный ключ и Certificate Signing Request (CSR) для нового пользователя.
* Утвердить сертификат средствами Kubernetes API.
* Создать изолированную `Role` внутри пространства имен `work` с запретом на удаление ресурсов.
* Настроить кастомный `kubeconfig` и протестировать права доступа.

---

## 📝 Пошаговые инструкции

### Шаг 1: Генерация криптографических ключей для пользователя

Все действия по генерации ключей выполняются на мастере, где есть утилита `openssl`.

**Выполнить на `k8s-master-01`:**

```bash
mkdir -p ~/k8s-lab/lab16 && cd ~/k8s-lab/lab16

# 1. Генерируем приватный ключ для Ивана
openssl genrsa -out ivan.key 2048

# 2. Создаем запрос на сертификат (CSR). 
# CN (Common Name) — это будет его Username в кластере. 
# O (Organization) — это его группа (полезно для массовых правил).
openssl req -new -key ivan.key -out ivan.csr -subj "/CN=developer-ivan/O=developers"

```

---

### Шаг 2: Отправка и утверждение CSR в Kubernetes

Теперь мы должны попросить корневой удостоверяющий центр нашего кластера подмахнуть этот сертификат. Для этого отправим специальный объект `CertificateSigningRequest` прямо в API.

Переведем наш `.csr` файл в base64 строку без переносов:

```bash
CSR_BASE64=$(cat ivan.csr | base64 | tr -d '\n')

```

Создай манифест запроса `k8s-csr.yaml`:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ivan-csr
spec:
  request: INPUT_YOUR_BASE64_HERE # Сюда подставится наша строка
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400 # Сертификат будет жить ровно 24 часа
  usages:
  - client auth

```

Подмени заглушку реальным хэшем (или отредактируй файл через `nano`) и примени его:

```bash
sed -i "s|INPUT_YOUR_BASE64_HERE|$CSR_BASE64|g" k8s-csr.yaml
kubectl apply -f k8s-csr.yaml

# Проверяем статус запроса (он должен быть в состоянии Pending)
kubectl get csr ivan-csr

```

**Утверждаем сертификат под правами администратора кластера:**

```bash
kubectl certificate approve ivan-csr

# Проверяем статус снова — теперь он Approved,Issued
kubectl get csr ivan-csr

```

Выгружаем готовый подписанный CRT-сертификат из кластера:

```bash
kubectl get csr ivan-csr -o jsonpath='{.status.certificate}' | base64 --decode > ivan.crt

```

---

### Шаг 3: Создание Роли (Role) и Привязки (RoleBinding)

Теперь Иван может аутентифицироваться в кластере, но у него нет прав даже на команду `kubectl get nodes`. Давай создадим для него роль внутри нашего неймспейса `work`, которая разрешает просматривать и редактировать поды/деплойменты, но **запрещает их удалять**.

Создай файл `rbac-ivan.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-limited-role
  namespace: work # Роль заперта строго внутри нашего пространства имен
rules:
- apiGroups: ["", "apps"] # Пустая группа "" означает core-ресурсы (поды, сервисы), "apps" — деплойменты
  resources: ["pods", "deployments", "services"]
  # Разрешаем просмотр, создание и изменение, но НЕ пишем сюда глагол "delete"!
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-ivan-to-role
  namespace: work
subjects:
- kind: User
  name: developer-ivan # Имя должно в точности совпадать с CN из SSL-сертификата!
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-limited-role # Связываем с созданной выше ролью
  apiGroup: rbac.authorization.k8s.io

```

Примени манифест:

```bash
kubectl apply -f rbac-ivan.yaml

```

---

### Шаг 4: Настройка изолированного конфигурационного файла (Kubeconfig)

Соберем для Ивана персональный `config` файл, внедрив туда выданные сертификаты кластера.

```bash
# 1. Добавляем данные о кластере
kubectl config set-cluster k8s-ha-cluster --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --server=https://192.168.77.180:8443 --kubeconfig=ivan.config

# 2. Добавляем учетные данные Ивана (ключ и сертификат)
kubectl config set-credentials developer-ivan --client-certificate=ivan.crt --client-key=ivan.key --embed-certs=true --kubeconfig=ivan.config

# 3. Создаем контекст для работы в namespace work по умолчанию
kubectl config set-context ivan-context --cluster=k8s-ha-cluster --user=developer-ivan --namespace=work --kubeconfig=ivan.config

# 4. Активируем этот контекст внутри файла
kubectl config use-context ivan-context --kubeconfig=ivan.config

```

Файл `ivan.config` готов. В реальной жизни ты бы отдал этот файл сотруднику на его рабочий ноутбук/MacBook.

---

### Шаг 5: Тестирование ограничений (Аудит прав)

Проверим работу ограничений прямо с мастера, принудительно подсовывая утилите `kubectl` конфиг Ивана через флаг `--kubeconfig`.

```bash
# 1. Проверяем список подов в разрешенном пространстве имен work
kubectl get pods --kubeconfig=ivan.config
# Результат: Успешный вывод списка подов!

# 2. Пытаемся посмотреть поды в дефолтном неймсейсе или kube-system
kubectl get pods -n kube-system --kubeconfig=ivan.config
# Результат: Жесткий отказ API-сервера!
# Error from server (Forbidden): pods is forbidden: User "developer-ivan" cannot list resource "pods" in API group "" in the namespace "kube-system"

# 3. Самый главный тест — попытка удалить под внутри пространства имен work
# Возьми имя любого живого пода из вывода первого пункта
kubectl delete pod <ИМЯ_ПОДА> --kubeconfig=ivan.config
# Результат: Блокировка! Роль сработала идеально.
# Error from server (Forbidden): pods "<имя>" is forbidden: User "developer-ivan" cannot delete resource "pods" in API group "" in the namespace "work"

```

Права доступа ограничены в полном соответствии с принципом наименьших привилегий (Least Privilege).

---

## 🔍 Инженерный траблшутинг RBAC (Middle-level)

Частая проверка, которую мидл делает перед выкаткой RBAC, чтобы не переключать конфиги туда-обратно — использование встроенной команды маскарада `auth can-i`.

```bash
# Проверить, может ли Иван удалять поды в неймсейсе work
kubectl auth can-i delete pods --as=developer-ivan -n work
# Вернет: no

# Проверить, может ли Иван смотреть сервисы в неймсейсе work
kubectl auth can-i get services --as=developer-ivan -n work
# Вернет: yes

```

*Инструмент `--as` позволяет администратору быстро симулировать запросы от лица любого пользователя кластера.*

---

## ✅ Критерии успешного выполнения лабораторной работы

1. Выпущен валидный SSL-сертификат для пользователя `developer-ivan`, подписанный CA кластера.
2. Объект `Role` жестко изолирован рамками `namespace: work`.
3. Создан рабочий файл конфигурации `ivan.config` с контекстом по умолчанию.
4. Инструмент `auth can-i` и прямые тесты подтверждают запрет на деструктивные действия (удаление ресурсов) и доступ в другие неймспейсы.

---

Слой безопасности доступа (RBAC) полностью закрыт! Наш кластер защищен на всех уровнях: от ресурсов и сети до учетных записей инженеров.

Следующий шаг — финальные темы Advanced-блока: **Lab 17: Мониторинг и обсервабилити кластера (Prometheus / Grafana)** или автоматизация деплоя через **Helm**. Напиши, куда двигаемся дальше.
