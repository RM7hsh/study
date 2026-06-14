# 🧪 Lab 3: Rolling Updates и Rollback

## 📋 Обзор
Эта лабораторная работа учит выполнять обновления сервисов без downtime.

## 🎯 Цели
- Создать сервис с начальной версией
- Выполнить rolling update на новую версию
- Выполнить rollback если необходимо
- Настроить параметры обновления

## 📊 Требуемое окружение

**Используем кластер из Lab 1-2:**
```
- 2x Manager nodes
- 3x Worker nodes
- Все должны быть Ready
```

---

## 📝 Пошаговые инструкции

### Шаг 1: Создание первого сервиса v1.0

**Выполнить на `swarm-manager-01`:**
```bash
# Создание веб-приложения версии 1.0
docker service create \
  --name web-app \
  --replicas 3 \
  --publish 80:8080 \
  --label version=1.0 \
  httpd:2.4

# Ожидаемый результат:
# Сервис создан
# xxxxxxxxxxxx
```

---

### Шаг 2: Проверка статуса сервиса

**Выполнить на `swarm-manager-01`:**
```bash
# Просмотр сервиса
docker service ls

# Ожидаемый результат:
# ID         NAME      MODE        REPLICAS   IMAGE        PORTS
# xxxx       web-app   replicated  3/3        httpd:2.4    *:80->8080/tcp

# Просмотр всех задач
docker service ps web-app

# Ожидаемый результат:
# Все контейнеры должны быть в Running

# Тестирование сервиса
curl http://10.0.0.10:80

# Ожидаемый результат:
# HTML страница Apache
```

---

### Шаг 3: Подготовка новой версии образа

**Выполнить на `swarm-manager-01`:**
```bash
# Проверка что у нас есть нужный образ
docker pull httpd:2.4.59

# Можно создать собственный образ если хотите:
# docker build -t myapp:2.0 .
# docker push myapp:2.0 (в private registry)

# Или создать локальный для демо
# docker tag httpd:2.4.59 localhost/httpd:2.0
```

---

### Шаг 4: Rolling Update с базовыми параметрами

**Выполнить на `swarm-manager-01`:**
```bash
# Обновление сервиса на новую версию
# По умолчанию обновляет по одному контейнеру
docker service update \
  --image httpd:2.4.59 \
  --update-delay 5s \
  --update-parallelism 1 \
  web-app

# Ожидаемый результат:
# web-app
```

---

### Шаг 5: Мониторинг процесса обновления

**Выполнить на `swarm-manager-01` (во время обновления):**
```bash
# Просмотр статуса обновления в реальном времени
watch 'docker service ps web-app'

# Или в отдельной сессии:
while true; do
  docker service ps web-app
  sleep 2
  clear
done

# Ожидаемый результат:
# Контейнеры будут по одному переходить в состояние Shutdown и затем Running
# Процесс займет около 15+ секунд (3 контейнера * 5 сек задержка)
```

---

### Шаг 6: Проверка что сервис остается доступным

**Выполнить на другой терминальной сессии:**
```bash
# Во время обновления постоянно тестируем
while true; do
  curl -s http://10.0.0.10:80 | head -n 1
  echo " - $(date +%H:%M:%S)"
  sleep 1
done

# Ожидаемый результат:
# HTML содержимое должно быть доступно ВСЕ ВРЕМЯ обновления
# ✅ Zero downtime
```

---

### Шаг 7: Проверка финального состояния

**Выполнить на `swarm-manager-01`:**
```bash
# После завершения обновления
docker service ps web-app

# Ожидаемый результат:
# Все контейнеры должны быть с IMAGE: httpd:2.4.59
# Все контейнеры в Running
# Нет контейнеров в Shutdown

# Проверка информации о сервисе
docker service inspect web-app --pretty
```

---

### Шаг 8: Откат (Rollback) на предыдущую версию

**Выполнить на `swarm-manager-01`:**
```bash
# Выполнение rollback
docker service rollback web-app

# Ожидаемый результат:
# web-app

# Мониторинг процесса rollback
watch 'docker service ps web-app'

# Ожидаемый результат:
# Контейнеры будут обновляться обратно на httpd:2.4
```

---

### Шаг 9: Продвинутое обновление с параметрами

**Выполнить на `swarm-manager-01`:**
```bash
# Обновление с большим параллелизмом и более долгой задержкой
docker service update \
  --image httpd:2.4.59 \
  --update-delay 10s \
  --update-parallelism 2 \
  --update-failure-action pause \
  web-app

# Параметры:
# --update-delay 10s           - 10 секунд между обновлениями групп
# --update-parallelism 2       - обновлять по 2 контейнера одновременно
# --update-failure-action pause - остановить обновление если контейнер не запустился

# Мониторинг
watch 'docker service ps web-app'
```

---

### Шаг 10: Масштабирование во время обновления

**Выполнить на `swarm-manager-01`:**
```bash
# Сначала откатим обратно
docker service rollback web-app

# Дождемся завершения
sleep 20

# Теперь масштабируем до 6 реплико
docker service scale web-app=6

# Проверка
docker service ps web-app

# Ожидаемый результат:
# 6 контейнеров в Running
```

---

### Шаг 11: Обновление большого количества контейнеров

**Выполнить на `swarm-manager-01`:**
```bash
# Обновление с новыми параметрами
docker service update \
  --image httpd:2.4.59 \
  --update-delay 3s \
  --update-parallelism 2 \
  web-app

# Мониторинг
watch 'docker service ps web-app'

# Ожидаемый результат:
# 6 контейнеров будут обновляться по 2 одновременно
# Общее время: примерно 3 * 3 сек = 9 секунд
```

---

### Шаг 12: История обновлений

**Выполнить на `swarm-manager-01`:**
```bash
# Просмотр обновлений (из логов)
docker service inspect web-app --format '{{json .UpdateStatus}}'

# Информация о сервисе включая UpdateConfig
docker service inspect web-app --pretty

# Просмотр содержимого для разбора
docker service inspect web-app | grep -A 20 UpdateStatus
```

---

### Шаг 13: Очистка

**Выполнить на `swarm-manager-01`:**
```bash
# Удаление сервиса
docker service rm web-app

# Проверка
docker service ls

# Ожидаемый результат:
# Список пуст
```

---

## ✅ Проверка успешности выполнения

**Все условия должны быть выполнены:**
```
☑️ Сервис создан и работает с 3 репликами
☑️ Rolling update выполнен без downtime
☑️ Во время обновления сервис оставался доступным
☑️ Все контейнеры обновлены на новую версию
☑️ Rollback работает корректно
☑️ Параметры обновления настроены правильно
☑️ Масштабирование и обновление работают вместе
☑️ При обновлении 6 контейнеров они обновлялись параллельно
```

---

## 🔍 Диагностика проблем

### Проблема: Обновление зависает
```bash
# Проверка статуса контейнеров
docker service ps web-app

# Проверка логов контейнера
docker service logs web-app

# Если контейнер в статусе "Preparing" - может быть проблема с образом
# Проверка образа на worker узле
ssh worker-01
docker images | grep httpd
```

### Проблема: Сервис становится недоступным
```bash
# Проверка количества Running контейнеров
docker service ps web-app --filter "desired-state=running" | wc -l

# Должно быть хотя бы 1

# Проверка health
docker service inspect web-app | grep -A 10 HealthConfig
```

---

## 📚 Дополнительные команды

```bash
# Просмотр только вышедших из строя контейнеров
docker service ps web-app --filter "desired-state=shutdown"

# Получение только айди контейнеров
docker service ps web-app -q

# Информация о каждом контейнере
docker service ps web-app --no-trunc

# Фильтрация по узлам
docker service ps web-app --filter "node=swarm-worker-01"

# Экспорт конфигурации
docker service inspect web-app --format '{{json .}}' > web-app-config.json
```

---

## 🎓 Что мы выучили

✅ Создание сервисов с репликами  
✅ Rolling updates без downtime  
✅ Параметры обновления (delay, parallelism)  
✅ Rollback на предыдущую версию  
✅ Мониторинг процесса обновления  
✅ Масштабирование с обновлениями  
✅ Zero downtime deployment  

---

## 🚀 Следующие шаги

Переход к **Lab 4**: Networks и Service Discovery
