# 🧪 Lab 7: Входная маршрутизация через Ingress Controller

## 📋 Обзор

Использование `NodePort` в продакшене ограничено: порты из диапазона `30000-32767` выглядят неэстетично, а выделение отдельного порта на каждое приложение быстро исчерпает ресурсы кластера. Для решения этой проблемы на уровне приложений (L7) применяется **Ingress**.

В этой лабораторной работе мы развернем на нашем bare-metal кластере (Rocky Linux 10) **NGINX Ingress Controller**, настроим виртуальный хостинг и организуем маршрутизацию на основе путей (Path-based routing) для нескольких приложений внутри пространства имен `work`.

```
                      [ Внешний трафик: k8s.work.local ]
                                      |
                                      v
                        [ Ingress NodePort: 30080 ]
                                      |
                         [ NGINX Ingress Controller ]
                               /             \
       path: /web             /               \           path: /api
                             v                 v
               [ Service: internal-web-svc ]  [ Service: api-svc ]
                             |                 |
                    [ Поды: mid-web-app ]    [ Поды: api-app ]

```

## 🎯 Цели

* Развернуть NGINX Ingress Controller, адаптированный под bare-metal архитектуру.
* Создать второе (микросервисное) приложение в неймсейсе `work` для демонстрации умной маршрутизации.
* Описать манифест `Ingress` с разделением трафика по URL-путям (`/` и `/api`).
* Настроить локальный DNS-резолв и протестировать доступ к кластеру по доменному имени.

---

## 📝 Пошаговые инструкции

### Шаг 1: Установка NGINX Ingress Controller (Bare-Metal вариант)

В облачных провайдерах Ingress-контроллер автоматически заказывает внешний LoadBalancer. На нашей физической схеме мы развернем его через манифест, который настроит контроллер как `NodePort`, открыв стандартные порты (обычно `30080` для HTTP и `30443` для HTTPS, если применить кастомные настройки, либо случайные порты).

**Выполнить на `k8s-master-01`:**

```bash
mkdir -p ~/k8s-lab/lab7 && cd ~/k8s-lab/lab7

# Применяем официальный bare-metal манифест NGINX Ingress (версия, совместимая с K8s v1.28)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml

# Мониторинг запуска контроллера (он разворачивается в собственном неймсейсе)
kubectl get pods -n ingress-nginx -w

```

Дождись, пока под `ingress-nginx-controller-xxxx` перейдет в статус `Running`.

**Определяем внешние порты контроллера:**

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller

```

В выводе в колонке `PORT(S)` найди маппинг портов. Например: `80:31255/TCP, 443:30718/TCP`. В данном случае порт **31255** — это твой HTTP-вход в кластер. Запомни его.

---

### Шаг 2: Создание второго приложения (`api-app`) в namespace `work`

Чтобы показать, как Ingress разделяет трафик на одном порту, создадим быстрое псевдо-API приложение, которое будет отвечать на пути `/api`.

Создай файл `api-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: work
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod-api
  template:
    metadata:
      labels:
        app: prod-api
    spec:
      containers:
      - name: api-backend
        # Используем образ httpenv, который удобно выводит системную информацию в JSON
        image: brendanburns/httpenv:v1.0.0
        ports:
        - containerPort: 8888
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: work
spec:
  type: ClusterIP # Нам НЕ нужен NodePort для этого приложения, Ingress дотянется сам
  selector:
    app: prod-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8888

```

Примени манифест:

```bash
kubectl apply -f api-app.yaml

```

---

### Шаг 3: Создание правил маршрутизации Ingress

Теперь свяжем наши два сервиса (`internal-web-svc` из Lab 6 и новый `api-svc`) под единым доменным именем `k8s.work.local`.

Создай файл `ingress-routing.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: work
  annotations:
    # Указываем, какой именно контроллер должен обрабатывать этот ресурс
    kubernetes.io/ingress.class: "nginx"
    # Полезная аннотация: позволяет корректно прокидывать пути в бэкенд
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - host: k8s.work.local # Наш виртуальный хост
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: internal-web-svc # Наш старый Nginx деплоймент
            port:
              number: 8080 # Внутренний порт сервиса ClusterIP
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8

```

Примени манифест:

```bash
kubectl apply -f ingress-routing.yaml

# Проверяем статус Ingress
kubectl get ingress main-ingress

```

---

### Шаг 4: Настройка локального DNS и проверка работы

Поскольку домена `k8s.work.local` не существует в глобальном интернете, нам нужно заставить твою рабочую машину (MacBook/ПК) доверять этому имени.

1. На своем рабочем компьютере открой файл `/etc/hosts` (или `C:\Windows\System32\drivers\etc\hosts` для Windows) под правами администратора.
2. Добавь строку, указывающую на **любую воркер-ноду** кластера (так как контроллер висит на NodePort на всех узлах):
```text
192.168.77.186 k8s.work.local

```



Теперь открывай терминал на своей локальной машине (не на мастере) и тестируй L7-балансировку (замени `31255` на HTTP-порт контроллера, полученный в Шаге 1):

```bash
# 1. Тестируем корневой путь -> должен ответить Nginx из internal-web-svc
curl http://k8s.work.local:31255/

# 2. Тестируем путь /api -> должен ответить JSON-бэкенд из api-svc
curl http://k8s.work.local:31255/api

```

---

## 🔍 Инженерный траблшутинг Ingress (Middle-level)

Если при отправке запросов ты получаешь ошибки, мидл-инженер не гадает, а идет смотреть логи самого контроллера.

1. **Ошибка `404 Not Found` (отдает именно страница NGINX Ingress):**
Это означает, что запрос дошел до контроллера, но контроллер не нашел совпадений в правилах.
* *Где искать проблему:* Проверь заголовок `Host` в своем запросе. Если ты стучишься по IP-адресу (`http://192.168.77.186:31255/`), контроллер выдаст 404, потому что в манифесте жестко прописано `host: k8s.work.local`.


2. **Ошибка `503 Service Temporarily Unavailable`:**
Контроллер знает о существовании домена и пути, но не может перенаправить трафик дальше.
* *Где искать проблему:* Проверь цепочку `Ingress -> Service -> Endpoints`. Скорее всего, у тебя упали поды бэкенда, либо порт `number` внутри манифеста Ingress не совпадает с реальным `port` сервиса ClusterIP.


3. **Как читать логи контроллера напрямую:**
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50 -f

```


*Там в реальном времени отображаются входящие апстрим-запросы и ошибки маппинга.*

---

## ✅ Критерии успешного выполнения лабораторной работы

1. Ingress Controller успешно развернут и слушает порты на физическом HA-слое.
2. Создан и применен ресурс `Ingress` в пространстве имен `work`.
3. Запросы на `k8s.work.local/` и `k8s.work.local/api` возвращают ответы от двух абсолютно разных приложений.
4. Вы понимаете разницу между зонами ответственности L4 (Service) и L7 (Ingress).

---

Сетевой блок полностью закрыт! Мы умеем разворачивать приложения, квотировать их, балансировать на L4 и маршрутизировать на L7.

Следующий блок по программе — **Storage и Volumes**. Готов перейти к **Lab 8: Persistent Volumes (PV), Persistent Volume Claims (PVC) и StorageClasses** для работы с постоянными данными (stateful), или хочешь здесь закрепить что-то еще?
