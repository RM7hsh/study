# 🧪 Lab 11: Управление конфигурациями приложений через ConfigMaps

## 📋 Обзор

**ConfigMap** — это объект Kubernetes, который позволяет хранить строки, файлы или целые директории конфигураций отдельно от самих контейнеров. Их можно прокидывать внутрь подов двумя путями: как переменные окружения (`env`) или монтировать как файлы через специальный том (`volume`).

В этой лабораторной работе мы настроим кастомное веб-приложение в пространстве имен `work`. Мы вынесем конфигурационный файл веб-сервера и переменные окружения в `ConfigMap`, а также разберем важнейшую мидл-фичу — **горячую перезагрузку (Live Reload)** конфигов без перезапуска самих подов.

## 🎯 Цели

* Создать `ConfigMap` двумя способами: из литералов и декларативно из файла.
* Прокинуть переменные из ConfigMap в качестве `env` приложения.
* Смонтировать конфигурационный файл внутрь контейнера как Volume со строгими `requests/limits`.
* Изучить механизм автоматического обновления файлов (Symlink-update) при изменении ConfigMap.

---

## 📝 Пошаговые инструкции

### Шаг 1: Подготовка файлов и создание ConfigMap

Мы создадим `ConfigMap`, который будет содержать две вещи:

1. Одиночную переменную (`APP_COLOR`).
2. Целый кастомный HTML-шаблон, который мы подменим внутри Nginx.

Выполнить на `k8s-master-01`:

```bash
mkdir -p ~/k8s-lab/lab11 && cd ~/k8s-lab/lab11

# Создаем манифест нашего общего конфига configmap-app.yaml
cat > configmap-app.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: work
data:
  # 1. Простая ключ-значение строка
  APP_COLOR: "dark-mode"
  APP_EDITION: "Enterprise"
  
  # 2. Целый файл конфигурации/верстки
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>K8s Production App</title>
        <style>
            body { background-color: #2c3e50; color: #ecf0f1; font-family: sans-serif; text-align: center; padding-top: 50px; }
        </style>
    </head>
    <body>
        <h1>Привет от Rajabali! Кластер работает на Rocky Linux 10.</h1>
        <p>ConfigMap успешно смонтирован как файл.</p>
    </body>
    </html>
EOF

kubectl apply -f configmap-app.yaml

```

Проверим, как кластер сохранил данные:

```bash
kubectl describe configmap app-config -n work

```

---

### Шаг 2: Развертывание приложения с интеграцией ConfigMap

Теперь создадим `Deployment`, который заберет `APP_COLOR` в системные переменные, а файл `index.html` положит прямиком в дефолтную директорию Nginx, перетерев стандартную заглушку.

Создай файл `deployment-configmap.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-web
  namespace: work
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cm-web
  template:
    metadata:
      labels:
        app: cm-web
    spec:
      containers:
      - name: web-server
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        
        # 📥 Путь 1: Импорт отдельных ключей как Переменных Окружения (ENV)
        env:
        - name: APPLICATION_COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_COLOR
              
        # 📥 Путь 2: Монтирование файла из ConfigMap на дисковый уровень контейнера
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html # Куда Nginx смотрит за статикой
          
      volumes:
      - name: html-volume
        configMap:
          name: app-config # Указываем из какого CM брать данные
          items:
          - key: index.html
            path: index.html # Под каким именем сохранить файл на диске

```

Примени манифест:

```bash
kubectl apply -f deployment-configmap.yaml

# Ждем готовности подов
kubectl get pods -n work -l app=cm-web

```

---

### Шаг 3: Проверка импорта данных

Давай провалимся в один из подов и проверим, что обе механики (ENV и Volume) отработали корректно.

```bash
# Получаем имя пода
kubectl get pods -n work -l app=cm-web

# Проверяем переменную окружения внутри контейнера
kubectl exec -it <ИМЯ_ТВОЕГО_ПОДА> -n work -- env | grep APPLICATION_COLOR
# Должно вернуть: APPLICATION_COLOR=dark-mode

# Проверяем, что файл физически лежит на диске
kubectl exec -it <ИМЯ_ТВОЕГО_ПОДА> -n work -- cat /usr/share/nginx/html/index.html

```

---

### Шаг 4: Продвинутый мидл-кейс — Горячее обновление конфигов (Live Reload)

Что произойдет, если мы обновим `ConfigMap` прямо на лету, не удаляя деплоймент?

Давай проверим это руками. Открой файл `configmap-app.yaml` через `nano` и измени текст заголовка `<h1>` внутри `index.html` (например, напиши *"Версия конфига v2"*).

```bash
# Применяем обновленный ConfigMap
kubectl apply -f configmap-app.yaml

```

**Внимание, магия K8s:**
Контейнерные переменные окружения (`env`) обновляются *только при перезапуске* контейнера. Но файлы, смонтированные через `Volume`, Kubernetes обновляет автоматически!

Утилита `kubelet` каждые несколько десятков секунд сверяет хэши. Запусти команду чтения файла внутри пода с интервалом и подожди около 30–60 секунд:

```bash
while true; do kubectl exec -it <ИМЯ_ТВОЕГО_ПОДА> -n work -- cat /usr/share/nginx/html/index.html | grep h1; sleep 5; done

```

Ты увидишь, что текст внутри запущенного контейнера изменился **самостоятельно**, без рестарта пода (`RESTARTS` по-прежнему равен 0)! Это происходит потому, что K8s монтирует файлы через симлинки, и рантайм `containerd` просто перенаправляет ссылку на новую директорию с обновленным конфигом.

---

## 🔍 Инженерный траблшутинг ConfigMaps (Middle-level)

1. **Файлы в поде не обновляются «на лету»:**
* *Причина:* Если ты при монтировании тома ConfigMap использовал директиву `subPath` (чтобы подкинуть один файл в папку, не затирая остальные файлы внутри контейнера), **автообновление работать не будет**. Изменения применятся только после принудительного рестарта пода: `kubectl rollout restart deployment/configmap-web -n work`.


2. **Защита от случайных изменений (Immutable ConfigMaps):**
В больших кластерах случайное изменение конфига может сломать прод. В свежих версиях K8s можно намертво запретить изменять ConfigMap. Для этого в `metadata` добавляется поле:
```yaml
immutable: true

```


*Такой конфиг невозможно изменить без его удаления и пересоздания, что защищает приложения от непредвиденного Live Reload.*

---

## ✅ Критерии успешного выполнения лабораторной работы

1. Описан и применен ресурс `ConfigMap` в неймсейсе `work`.
2. Переменные успешно считываются контейнером через блок `valueFrom`.
3. Файл конфигурации подменил дефолтную страницу Nginx.
4. Успешно зафиксирован механизм Live Reload для смонтированного тома.
