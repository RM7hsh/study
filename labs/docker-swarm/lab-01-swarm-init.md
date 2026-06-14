# 🧪 Lab 1: Инициализация Docker Swarm кластера

## 📋 Обзор
Эта лабораторная работа учит создавать и инициализировать Docker Swarm кластер.

## 🎯 Цели
- Инициализировать Swarm на первом узле
- Получить токен для присоединения worker nodes
- Понимать роль Manager и Worker узлов

## 📊 Требуемое окружение

**ВМ которые нужно развернуть:**
```
3 Virtual Machines (все с Rocky Linux 10):

1. swarm-manager-01
   - vCPU: 2
   - RAM: 2 GB
   - Storage: 20 GB
   - IP: 10.0.0.10 (выберите subnet)
   - Role: Manager (Leader)

2. swarm-manager-02
   - vCPU: 2
   - RAM: 2 GB
   - Storage: 20 GB
   - IP: 10.0.0.11
   - Role: Manager (Backup)

3. swarm-worker-01
   - vCPU: 2
   - RAM: 2 GB
   - Storage: 20 GB
   - IP: 10.0.0.20
   - Role: Worker
```

**Требования ко всем ВМ:**
- ✅ Rocky Linux 10 (минимум)
- ✅ Docker установлен и запущен
- ✅ SSH доступ между узлами
- ✅ Порты открыты (7946/tcp, 7946/udp, 4789/udp)
- ✅ Network connectivity между всеми узлами

---

## 📝 Пошаговые инструкции

### Шаг 1: Проверка состояния Docker на всех узлах

**Выполнить на всех ВМ:**
```bash
# Проверка, что Docker установлен и запущен
docker --version
sudo systemctl status docker

# Проверка что Swarm НЕ инициализирован
docker info | grep Swarm

# Ожидаемый результат:
# Swarm: inactive
```

**Что должно получиться:**
```
Docker version 24.0.x or higher
docker.service is running
Swarm: inactive
```

---

### Шаг 2: Инициализация Swarm на первом Manager узле

**Выполнить только на `swarm-manager-01`:**
```bash
# Инициализирование Swarm
# --advertise-addr = IP адрес для других узлов
docker swarm init --advertise-addr 10.0.0.10

# Команда вернет результат вида:
# Swarm initialized: current node (xxxxxx) is now a manager.
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxxxx 10.0.0.10:2377
```

**Что должно получиться:**
```
✅ Swarm mode активирован
✅ Выведен token для worker nodes
✅ Выведена команда для присоединения
```

---

### Шаг 3: Просмотр информации о кластере

**Выполнить на `swarm-manager-01`:**
```bash
# Посмотреть информацию о Swarm
docker info | grep -A 10 Swarm

# Ожидаемый результат:
# Swarm: active
#  NodeID: xxxxxxxxxxxxxxxxxxxxx
#  Is Manager: true
#  ClusterID: xxxxxxxxxxxxxxxxxxxxx
#  Managers: 1
#  Nodes: 1
```

**Просмотр узлов:**
```bash
docker node ls

# Ожидаемый результат:
# ID                    HOSTNAME            STATUS    AVAILABILITY  MANAGER STATUS
# xyz123 *             swarm-manager-01    Ready     Active         Leader
```

---

### Шаг 4: Получение токена для worker nodes

**Выполнить на `swarm-manager-01`:**
```bash
# Получить токен для worker nodes
docker swarm join-token worker

# Ожидаемый результат:
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx 10.0.0.10:2377

# Скопировать эту команду для следующего шага
```

---

### Шаг 5: Добавление первого worker node

**Выполнить на `swarm-worker-01`:**
```bash
# Выполнить команду которую вы скопировали на предыдущем шаге
# Замените SWMTKN-1-xxxxxxxxxxxxxxxx на реальный токен
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx 10.0.0.10:2377

# Ожидаемый результат:
# This node joined a swarm as a worker.
```

---

### Шаг 6: Добавление второго manager node

**Выполнить на `swarm-manager-01`:**
```bash
# Получить токен для manager nodes
docker swarm join-token manager

# Ожидаемый результат:
# To add a manager to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx 10.0.0.10:2377
```

**Выполнить на `swarm-manager-02`:**
```bash
# Выполнить команду для manager
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxx 10.0.0.10:2377

# Ожидаемый результат:
# This node joined a swarm as a manager.
```

---

### Шаг 7: Проверка финального состояния

**Выполнить на `swarm-manager-01`:**
```bash
# Список всех узлов
docker node ls

# Ожидаемый результат:
# ID                    HOSTNAME            STATUS    AVAILABILITY  MANAGER STATUS
# abc123 *             swarm-manager-01    Ready     Active         Leader
# def456               swarm-manager-02    Ready     Active         Reachable
# ghi789               swarm-worker-01     Ready     Active         -

# Проверка что узлы видят друг друга
docker node inspect --pretty swarm-manager-01
docker node inspect --pretty swarm-manager-02
docker node inspect --pretty swarm-worker-01
```

**Проверка на worker узле:**
```bash
# Выполнить на swarm-worker-01
sudo docker node ls

# Ожидаемый результат: ERROR
# Error response from daemon: this node is not a swarm manager.
# Use "docker swarm leave" to leave a swarm cluster.
# ✅ ЭТО НОРМАЛЬНО! Worker узлы не могут управлять кластером
```

---

## ✅ Проверка успешности выполнения

**Все условия должны быть выполнены:**
```
☑️ 3 узла в списке (docker node ls)
☑️ 2 Manager узла: 1 Leader + 1 Reachable
☑️ 1 Worker узел
☑️ Все узлы в статусе "Ready"
☑️ Все узлы имеют Availability "Active"
☑️ Worker узел не может выполнять команды управления
☑️ Узлы имеют IP адреса согласно плану
```

---

## 🔍 Диагностика проблем

### Проблема: Узел не присоединяется
```bash
# Проверка сетевого подключения
ping 10.0.0.10

# Проверка портов
sudo ss -tlnp | grep 2377
sudo ss -tlnp | grep 7946

# Проверка firewall
sudo firewall-cmd --list-ports

# Открытие портов если нужно
sudo firewall-cmd --add-port=2377/tcp --permanent
sudo firewall-cmd --add-port=7946/tcp --permanent
sudo firewall-cmd --add-port=7946/udp --permanent
sudo firewall-cmd --add-port=4789/udp --permanent
sudo firewall-cmd --reload
```

### Проблема: Узел не виден в списке
```bash
# Перезагрузка Docker демона
sudo systemctl restart docker

# Проверка логов
sudo journalctl -u docker -n 50
```

### Проблема: Manager узел downgraded
```bash
# Это может быть из-за потери кворума
# Проверка количества managers
docker node ls | grep -c "Manager Status"

# Должно быть 2 managers, иначе кластер может быть нестабилен
```

---

## 📚 Дополнительные команды для изучения

```bash
# Полная информация об узле
docker node inspect swarm-manager-01 --pretty

# Информация о самом узле
docker info

# Показать токен worker
docker swarm join-token worker

# Показать токен manager
docker swarm join-token manager

# Ротация токенов (для безопасности)
docker swarm join-token --rotate worker

# Статистика узлов
docker stats
```

---

## 🎓 Что мы выучили

✅ Инициализация Docker Swarm  
✅ Роли узлов: Manager vs Worker  
✅ Токены для присоединения узлов  
✅ Проверка статуса кластера  
✅ Высокая доступность (2 managers)  

---

## 🚀 Следующие шаги

Переход к **Lab 2**: Управление узлами (drain, promote, remove)
