# 🧪 Lab 2: Добавление Worker Nodes и Управление кластером

## 📋 Обзор
Эта лабораторная работа учит добавлять дополнительные worker nodes и управлять ими.

## 🎯 Цели
- Добавить еще 2 worker nodes
- Проверить балансировку нагрузки
- Научиться управлять узлами

## 📊 Требуемое окружение

**Дополнительные ВМ (к Lab 1):**
```
2 дополнительных Worker:

4. swarm-worker-02
   - vCPU: 2
   - RAM: 2 GB
   - Storage: 20 GB
   - IP: 10.0.0.21
   - Role: Worker

5. swarm-worker-03
   - vCPU: 2
   - RAM: 2 GB
   - Storage: 20 GB
   - IP: 10.0.0.22
   - Role: Worker

Итого должно быть 5 ВМ:
- 2x Manager
- 3x Worker
```

---

## 📝 Пошаговые инструкции

### Шаг 1: Подготовка новых worker узлов

**Выполнить на `swarm-worker-02` и `swarm-worker-03`:**
```bash
# Установка Docker (если еще не установлен)
sudo yum install -y docker

# Запуск Docker сервиса
sudo systemctl start docker
sudo systemctl enable docker

# Проверка
docker --version
docker ps
```

---

### Шаг 2: Получение токена worker node

**Выполнить на `swarm-manager-01`:**
```bash
# Получить текущий токен
docker swarm join-token worker

# Копируем команду вида:
# docker swarm join --token SWMTKN-1-xxxxx 10.0.0.10:2377
```

---

### Шаг 3: Присоединение новых worker узлов

**Выполнить на `swarm-worker-02`:**
```bash
# Присоединение к кластеру
docker swarm join --token SWMTKN-1-xxxxx 10.0.0.10:2377

# Ожидаемый результат:
# This node joined a swarm as a worker.
```

**Выполнить на `swarm-worker-03`:**
```bash
# Присоединение к кластеру
docker swarm join --token SWMTKN-1-xxxxx 10.0.0.10:2377

# Ожидаемый результат:
# This node joined a swarm as a worker.
```

---

### Шаг 4: Проверка состояния всего кластера

**Выполнить на `swarm-manager-01`:**
```bash
# Список всех узлов
docker node ls

# Ожидаемый результат:
# ID                    HOSTNAME            STATUS    AVAILABILITY  MANAGER STATUS
# abc123 *             swarm-manager-01    Ready     Active         Leader
# def456               swarm-manager-02    Ready     Active         Reachable
# ghi789               swarm-worker-01     Ready     Active         -
# jkl101               swarm-worker-02     Ready     Active         -
# mno112               swarm-worker-03     Ready     Active         -

# Всего: 5 узлов (2 Manager + 3 Worker)
```

---

### Шаг 5: Информация об узлах

**Выполнить на `swarm-manager-01`:**
```bash
# Подробная информация об узле
docker node inspect swarm-worker-02 --pretty

# Ожидаемый результат должен включать:
# ID:                     jkl101xxxx
# Version:
#  Index:                123
# CreatedAt:             2024-06-14T10:30:00.000000000Z
# UpdatedAt:             2024-06-14T10:35:00.000000000Z
# Hostname:              swarm-worker-02
# Status:
#  State:                ready
#  Addr:                 10.0.0.21
# Availability:          Active
# Manager Status:        -
# Platform:
#  OS:                   linux
#  Architecture:         x86_64
```

---

### Шаг 6: Метрики узлов

**Выполнить на `swarm-manager-01`:**
```bash
# Проверка ресурсов узлов
for node in $(docker node ls -q); do
  echo "Node: $(docker node inspect $node --format '{{.Description.Hostname}}')"
  docker node inspect $node --format '{{json .Description.Resources}}'
done

# Или просто просмотреть информацию о сокетах
docker info | grep -A 20 "Nodes"
```

---

### Шаг 7: Создание тестового сервиса на всех узлах

**Выполнить на `swarm-manager-01`:**
```bash
# Создание nginx сервиса с 5 репликами
# Они должны распределиться по всем worker узлам
docker service create \
  --name test-service \
  --replicas 5 \
  --publish 8080:80 \
  nginx:latest

# Ожидаемый результат:
# xxxxxxxxxxxx
```

---

### Шаг 8: Проверка распределения контейнеров

**Выполнить на `swarm-manager-01`:**
```bash
# Просмотр сервиса
docker service ls

# Ожидаемый результат:
# ID         NAME           MODE        REPLICAS   IMAGE         PORTS
# xxxx       test-service   replicated  5/5        nginx:latest  *:8080->80/tcp

# Просмотр задач (контейнеров)
docker service ps test-service

# Ожидаемый результат должен показать что контейнеры распределены:
# ID         NAME           IMAGE         NODE               DESIRED STATE  CURRENT STATE
# aaa        test-service.1 nginx:latest  swarm-worker-01    Running        Running
# bbb        test-service.2 nginx:latest  swarm-worker-02    Running        Running
# ccc        test-service.3 nginx:latest  swarm-worker-03    Running        Running
# ddd        test-service.4 nginx:latest  swarm-worker-01    Running        Running
# eee        test-service.5 nginx:latest  swarm-worker-02    Running        Running
```

**Проверка на узлах:**
```bash
# На каждом worker узле проверить контейнеры
docker ps

# На swarm-worker-01 должны быть контейнеры
# На swarm-worker-02 должны быть контейнеры
# На swarm-worker-03 должны быть контейнеры
```

---

### Шаг 9: Проверка load balancing

**Выполнить на любой ВМ:**
```bash
# Проверка что сервис доступен с любого узла
curl http://10.0.0.10:8080
curl http://10.0.0.20:8080
curl http://10.0.0.21:8080

# Все должны вернуть HTML страницу nginx
```

---

### Шаг 10: Дренирование узла для maintenance

**Выполнить на `swarm-manager-01`:**
```bash
# Перевести узел в режим "drain" (обслуживание)
# Новые контейнеры не будут развертываться на этом узле
docker node update --availability drain swarm-worker-03

# Проверка
docker node ls

# Ожидаемый результат:
# swarm-worker-03 должен иметь Availability: Drain

# Проверка задач
docker service ps test-service

# Контейнеры с worker-03 должны быть перемещены на другие узлы
```

---

### Шаг 11: Восстановление узла

**Выполнить на `swarm-manager-01`:**
```bash
# Вернуть узел в активное состояние
docker node update --availability active swarm-worker-03

# Проверка
docker node ls

# Ожидаемый результат:
# swarm-worker-03 должен иметь Availability: Active
```

---

### Шаг 12: Удаление тестового сервиса

**Выполнить на `swarm-manager-01`:**
```bash
# Удаление сервиса
docker service rm test-service

# Проверка
docker service ls

# Ожидаемый результат:
# Список пуст

# На worker узлах контейнеры должны быть удалены
docker ps
```

---

## ✅ Проверка успешности выполнения

**Все условия должны быть выполнены:**
```
☑️ 5 узлов всего (2 Manager + 3 Worker)
☑️ Все узлы в статусе "Ready"
☑️ Все узлы имеют Availability "Active"
☑️ Тестовый сервис развернут с 5 репликами
☑️ Контейнеры равномерно распределены по worker узлам
☑️ Load balancing работает (curl возвращает успех)
☑️ Drain/Active операции работают правильно
☑️ Контейнеры перемещались при drain узла
```

---

## 🔍 Диагностика проблем

### Проблема: Контейнеры не распределяются равномерно
```bash
# Проверка constraints на узлах
docker node inspect swarm-worker-01 --format '{{json .Spec.Labels}}'

# Проверка ресурсов
docker node inspect swarm-worker-01 --format '{{json .Description.Resources}}'
```

### Проблема: Сервис не доступен
```bash
# Проверка что контейнеры действительно запущены
docker service ps test-service

# Логи контейнера
docker service logs test-service

# Проверка портов
sudo ss -tlnp | grep 8080
```

---

## 📚 Дополнительные команды

```bash
# Переименование узла (для лучшей организации)
docker node update --label-add role=production swarm-worker-01

# Просмотр labels
docker node inspect swarm-worker-01 --format '{{json .Spec.Labels}}'

# Скрытие узла из UI но оставление в кластере
docker node update --availability pause swarm-worker-01

# Количество контейнеров на узле
docker node ls -q | xargs -I {} sh -c 'echo -n "{}: "; docker ps --filter "label=node={}" -q | wc -l'
```

---

## 🎓 Что мы выучили

✅ Добавление worker nodes в кластер  
✅ Распределение контейнеров между узлами  
✅ Проверка статуса и ресурсов узлов  
✅ Операции drain/active  
✅ Load balancing во Swarm  
✅ Масштабирование кластера  

---

## 🚀 Следующие шаги

Переход к **Lab 3**: Rolling Updates и Rollback
