# Деплой и настройка Apache Kafka: от bare metal до production-кластера

**Трек 04 — Software** | Статья 2 из 4  
**Проект:** kafka | **Профиль:** A (Infrastructure Platform)  
**Дата:** 2026-05-13

---

## Содержание

1. [Обзор: три пути деплоя Kafka](#обзор-три-пути-деплоя-kafka)
2. [Планирование и требования](#планирование-и-требования)
3. [Bare Metal: пошаговая установка Kafka](#bare-metal-пошаговая-установка-kafka)
4. [Конфигурация server.properties — ключевые параметры](#конфигурация-serverproperties--ключевые-параметры)
5. [KRaft: деплой без ZooKeeper](#kraft-деплой-без-zookeeper)
6. [ZooKeeper-режим: деплой со ZooKeeper (legacy)](#zookeeper-режим-деплой-со-zookeeper-legacy)
7. [Multi-broker кластер: полная настройка](#multi-broker-кластер-полная-настройка)
8. [Docker: деплой через Docker Compose](#docker-деплой-через-docker-compose)
9. [Kubernetes: деплой через Strimzi Operator](#kubernetes-деплой-через-strimzi-operator)
10. [KRaft vs ZooKeeper: сравнительный анализ](#kraft-vs-zookeeper-сравнительный-анализ)
11. [Production Checklist](#production-checklist)
12. [Источники](#источники)

---

## Обзор: три пути деплоя Kafka

Apache Kafka можно развернуть тремя принципиально разными способами, и выбор зависит от инфраструктуры, команды и требований к отказоустойчивости.

**Ключевая мысль:** для новых проектов в 2026 году — только KRaft. ZooKeeper удалён из Kafka 4.0. Если вы начинаете с нуля, ZooKeeper не должен даже рассматриваться. Выбор сводится к тому, где запускать: bare metal/Docker (для полного контроля) или Kubernetes через Strimzi (для облачной инфраструктуры).

Три пути деплоя — это не три «равнозначные» опции, а три уровня зрелости инфраструктуры:

```
┌─────────────────────────────────────────────────┐
│           Kubernetes (Strimzi)                   │
│  • Автоматический rolling update                 │
│  • Self-healing через K8s                       │
│  • GitOps-ready (ArgoCD, Flux)                  │
│  • Масштабирование через CRD                    │
├─────────────────────────────────────────────────┤
│           Docker / Docker Compose                │
│  • Повторяемое окружение                        │
│  • Быстрый деплой dev/QA                        │
│  • Ограниченная production-пригодность           │
├─────────────────────────────────────────────────┤
│           Bare Metal / Виртуальные машины        │
│  • Максимальная производительность               │
│  • Полный контроль над ОС и железом              │
│  • Требует ручной автоматизации                  │
└─────────────────────────────────────────────────┘
```

---

## Планирование и требования

Перед установкой необходимо ответить на четыре вопроса, определяющих архитектуру кластера:

1. **Сколько брокеров?** Минимум 3 для production (обеспечивает `replication.factor=3`).
2. **KRaft или ZooKeeper?** Для новых проектов — только KRaft.
3. **Где хранить данные?** Отдельные SSD-диски для `log.dirs`, никогда не совмещать с системным диском.
4. **Как клиенты будут подключаться?** DNS-имена (FQDN), а не IP-адреса.

### Аппаратные требования (per broker)

| Компонент | Минимум (dev) | Рекомендация (production) | Критичная нагрузка |
|-----------|---------------|---------------------------|---------------------|
| CPU | 4 ядра | 8-16 ядер | 24+ ядер |
| RAM | 8 GB (heap 2 GB) | 32 GB (heap 6 GB) | 64 GB (heap 12 GB) |
| Диск | 100 GB SSD | 2× SSD NVMe, раздельные | 4+ NVMe в JBOD |
| Сеть | 1 Gbps | 10 Gbps | 25+ Gbps |

**Ключевой принцип sizing:** Kafka использует page cache (кэш страниц) ОС, а не собственный in-process cache. Размер page cache = оставшаяся после JVM heap память. RAM = heap + page cache + OS overhead.

Формула для приблизительной оценки памяти:

```
Необходимая RAM ≈ JVM_heap + (пропускная_способность_записи × 30_секунд)
```

30 секунд — время, в течение которого данные могут находиться в page cache до сброса на диск. Например, при записи 100 MB/s нужно минимум 3 GB page cache. [Kafka Hardware & OS docs]

### Программные требования

| Компонент | Требование |
|-----------|------------|
| Java | OpenJDK 17 (Kafka 4.0+), 11 (клиенты/Streams) |
| ОС | Linux (RHEL 8+, Ubuntu 20.04+). Windows не рекомендуется |
| Файловая система | XFS (предпочтительно) или EXT4 |
| Ports | 9092 (PLAINTEXT), 9093 (CONTROLLER/KRaft), 9094 (SSL), 9095 (SASL_SSL) |

### Планирование томов (Volume Planning)

**Базовый расчёт дискового пространства:**

```
Необходимый объём = Средняя_пропускная_способность × retention_часов × 3600 × replication_factor / сжатие
```

Пример: 10 MB/s × 168 часов (7 дней) × 3 (RF) / 0.5 (сжатие) = ~36 TB на кластер, или ~12 TB на брокер (при 3 брокерах).

**Правила размещения дисков:**
- Никогда не размещайте `log.dirs` на системном диске
- Используйте JBOD (Just a Bunch Of Disks) вместо RAID для Kafka-данных — Kafka имеет собственную репликацию, дублирование на уровне RAID избыточно и снижает производительность
- Разные `log.dirs` на разных физических устройствах дают параллельный I/O
- Монтируйте с опцией `noatime`

---

## Bare Metal: пошаговая установка Kafka

Установка на «голое железо» или виртуальные машины — базовый путь. Даёт полный контроль, требует ручной настройки всего.

### Шаг 1: Подготовка системы

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка Java 17
sudo apt install openjdk-17-jdk -y
java -version  # Должно показать 17.x

# Создание пользователя kafka
sudo useradd -r -s /bin/false kafka
sudo mkdir -p /opt/kafka /var/log/kafka /var/lib/kafka
```

### Шаг 2: Настройка системных лимитов

Kafka открывает множество файловых дескрипторов (по одному на каждый log segment + соединения). Дефолтных 1024 недостаточно.

```bash
# /etc/security/limits.conf
kafka soft nofile 65536
kafka hard nofile 65536
kafka soft nproc 32768
kafka hard nproc 32768

# /etc/sysctl.conf
vm.swappiness=1
vm.dirty_ratio=80
vm.dirty_background_ratio=5
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 134217728
net.ipv4.tcp_wmem=4096 65536 134217728
```

Параметры page cache (`vm.dirty_*`) критичны: Kafka полагается на отложенную запись через кэш ОС. Установка `vm.swappiness=1` предотвращает вытеснение page cache в swap.

Для кластеров с большим числом партиций — увеличьте `vm.max_map_count`:

```bash
# /etc/sysctl.conf — для 50000+ партиций на брокер
vm.max_map_count=262144
```

Каждый log segment потребляет 2 map areas (индекс + timeindex). При 50000 партиций и хотя бы одном сегменте — это уже 100000 map areas. Дефолтный лимит (~65535) будет превышен — брокер упадёт с `OutOfMemoryError (Map failed)`. [Kafka docs — Hardware & OS]

### Шаг 3: Загрузка и установка Kafka

```bash
# Скачивание (проверьте актуальную версию на kafka.apache.org/downloads)
KAFKA_VERSION=4.1.0
SCALA_VERSION=2.13
wget https://downloads.apache.org/kafka/${KAFKA_VERSION}/kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz

# Распаковка
sudo tar -xzf kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz -C /opt/kafka --strip-components=1

# Права
sudo chown -R kafka:kafka /opt/kafka /var/log/kafka /var/lib/kafka
```

### Шаг 4: Создание systemd-сервиса

```ini
# /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Broker
Documentation=https://kafka.apache.org
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment="KAFKA_HEAP_OPTS=-Xmx6G -Xms6G"
Environment="KAFKA_JMX_OPTS=-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=30
LimitNOFILE=65536
LimitNPROC=32768

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
```

---

## Конфигурация server.properties — ключевые параметры

Файл `config/server.properties` — основной конфигурационный файл брокера. Ниже — production-готовый шаблон с комментариями на русском.

```properties
# ============================================
# Идентификация брокера
# ============================================
# Уникальный ID брокера в кластере (1, 2, 3, ...)
broker.id=1

# ============================================
# Сетевые настройки (КРИТИЧНО!)
# ============================================
# listeners — на каких интерфейсах/портах брокер слушает
# advertised.listeners — что брокер сообщает клиентам для подключения
#
# ПРАВИЛО: advertised.listeners должен содержать DNS-имена,
# доступные клиентам. В Docker/K8s это ВСЕГДА отличается от listeners.
#
listeners=PLAINTEXT://kafka1.example.com:9092,SSL://kafka1.example.com:9094
advertised.listeners=PLAINTEXT://kafka1.example.com:9092,SSL://kafka1.example.com:9094

# Маппинг протоколов: какие listener'ы используют какой протокол
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_SSL:SASL_SSL,SASL_PLAINTEXT:SASL_PLAINTEXT

# Протокол для межброкерного взаимодействия
inter.broker.listener.name=PLAINTEXT
# В production с безопасностью — заменить на:
# inter.broker.listener.name=SSL

# ============================================
# Треды и сетевые буферы
# ============================================
# Сетевые треды: принимают соединения, читают/отправляют данные
num.network.threads=8
# I/O треды: обрабатывают запросы (запись в лог, чтение)
num.io.threads=16

# Буферы сокетов
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# ============================================
# Хранение данных
# ============================================
# Каталоги для данных. Каждый на отдельном физическом диске!
# Партиции распределяются round-robin по каталогам.
log.dirs=/data/kafka-logs-0,/data/kafka-logs-1

# Количество партиций по умолчанию для новых топиков
num.partitions=3

# ============================================
# Репликация и отказоустойчивость (КРИТИЧНО!)
# ============================================
# Фактор репликации по умолчанию (3 = данные на 3 брокерах)
default.replication.factor=3

# Минимальное число ISR (in-sync replicas) для подтверждения записи
# При acks=all продюсер ждёт подтверждения от min.insync.replicas
min.insync.replicas=2
# Правило: min.insync.replicas < default.replication.factor
# Типовые значения: RF=3 → minISR=2, RF=5 → minISR=3

# Запрет unclean leader election — лидером может стать ТОЛЬКО реплика из ISR
# Потеря данных хуже, чем недоступность. Production — всегда false.
unclean.leader.election.enable=false

# ============================================
# Внутренние топики (репликация обязательно!)
# ============================================
# Топик для хранения consumer offsets
offsets.topic.replication.factor=3
# Топик для хранения статуса транзакций
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# ============================================
# Хранение: retention и compaction
# ============================================
# Политика очистки по умолчанию: delete или compact
log.cleanup.policy=delete

# retention — хранение по времени (168 часов = 7 дней)
log.retention.hours=168
# retention — ограничение по размеру (опционально)
# log.retention.bytes=107374182400  # 100 GB на партицию

# Размер сегмента лога — после достижения создаётся новый сегмент
log.segment.bytes=1073741824  # 1 GB

# Проверка retention каждые 5 минут
log.retention.check.interval.ms=300000

# ============================================
# Управление топиками
# ============================================
# Запрет автоматического создания топиков
# Production — всегда false! Топики должны создаваться осознанно.
auto.create.topics.enable=false
# Запрет удаления топиков (опционально, зависит от политик)
delete.topic.enable=true

# ============================================
# Безопасность (базовый уровень)
# ============================================
# В production необходимо настроить SSL/SASL.
# Минимальный набор для mTLS:
# ssl.keystore.location=/var/private/ssl/kafka.keystore.jks
# ssl.keystore.password=changeit
# ssl.key.password=changeit
# ssl.truststore.location=/var/private/ssl/kafka.truststore.jks
# ssl.truststore.password=changeit
# ssl.client.auth=required

# ============================================
# ZooKeeper (ТОЛЬКО если не KRaft!)
# ============================================
# zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
# zookeeper.connection.timeout.ms=18000
```

### Конфигурация JVM

```bash
# Через переменные окружения при запуске:
export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"
export KAFKA_JVM_PERFORMANCE_OPTS="-server -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+DisableExplicitGC \
  -XX:G1HeapRegionSize=16M \
  -XX:MetaspaceSize=96m \
  -XX:MinMetaspaceFreeRatio=50 \
  -XX:MaxMetaspaceFreeRatio=80 \
  -Djava.awt.headless=true"
```

**Правила heap sizing:**
- Никогда не выделяйте больше 50% RAM под heap — остальное для page cache
- Типичный диапазон: 4-8 GB для умеренных нагрузок, 12-16 GB для высоких
- G1GC — стандарт для Kafka начиная с Java 9+
- `MaxGCPauseMillis=20` — агрессивная цель по паузам GC (Kafka чувствителен к latency)

---

## KRaft: деплой без ZooKeeper

**KRaft (Kafka Raft)** — встроенный протокол консенсуса, заменивший ZooKeeper для управления метаданными. Начиная с Kafka 3.3 — production-ready для новых кластеров. С версии 4.0 ZooKeeper удалён полностью.

### Архитектура KRaft-кластера

В KRaft-режиме брокеры могут выполнять три роли:

- **broker** — обработка данных (продюсеры, консюмеры, хранение партиций)
- **controller** — управление метаданными кластера (кворум RAFT)
- **broker,controller (combined)** — обе роли в одном процессе (только для dev/testing)

```
Production-архитектура KRaft (isolated mode):

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Controller│  │Controller│  │Controller│   ← Quorum RAFT (3 или 5 нод)
  │  node 1  │  │  node 2  │  │  node 3  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      │ (метаданные)
       ┌──────────────┼──────────────┐
       │              │              │
  ┌────┴─────┐  ┌────┴─────┐  ┌────┴─────┐
  │  Broker  │  │  Broker  │  │  Broker  │   ← Обработка данных
  │  node 1  │  │  node 2  │  │  node 3  │
  └──────────┘  └──────────┘  └──────────┘
```

### Конфигурация KRaft (server.properties, production, isolated mode)

**Брокеры** (3 ноды, только роль broker):

```properties
# broker.id уникален для каждой ноды (1, 2, 3, ...)
# node.id должен совпадать с broker.id
broker.id=1
node.id=1

# Роль: только broker. Контроллеры — отдельные процессы
process.roles=broker

# Слушаем клиентов на 9092. Контроллер-листенер broker'у не нужен
listeners=PLAINTEXT://broker1.example.com:9092
advertised.listeners=PLAINTEXT://broker1.example.com:9092

# Подключение к кворуму контроллеров
controller.quorum.voters=1@controller1.example.com:9093,2@controller2.example.com:9093,3@controller3.example.com:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT

# Данные
log.dirs=/var/lib/kafka/data

# Внутренние топики
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

**Контроллеры** (3 ноды, только роль controller):

```properties
node.id=1
process.roles=controller

# Контроллер слушает ТОЛЬКО внутренние RAFT-сообщения
listeners=CONTROLLER://controller1.example.com:9093
controller.quorum.voters=1@controller1.example.com:9093,2@controller2.example.com:9093,3@controller3.example.com:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT

# Для контроллеров обязательно указывать каталог метаданных
metadata.log.dir=/var/lib/kafka/metadata
```

### Форматирование хранилища и запуск

KRaft требует инициализации метаданных перед первым запуском. Один и тот же `cluster-id` должен использоваться на всех нодах.

```bash
# 1. Генерация cluster-id (один раз, на любой ноде)
CLUSTER_ID=$(/opt/kafka/bin/kafka-storage.sh random-uuid)
echo $CLUSTER_ID  # Сохраните! Понадобится для всех нод

# 2. Форматирование хранилища на КАЖДОЙ ноде
/opt/kafka/bin/kafka-storage.sh format \
  --config /opt/kafka/config/server.properties \
  --cluster-id $CLUSTER_ID

# 3. Запуск (сначала контроллеры, потом брокеры)
sudo systemctl start kafka
```

**Проверка кворума:**

```bash
# Статус метаданных
/opt/kafka/bin/kafka-metadata-quorum.sh \
  --bootstrap-server broker1:9092 describe --status
```

Вывод должен показать `LeaderId`, текущий кворум и `MaxFollowerLag`.

---

## ZooKeeper-режим: деплой со ZooKeeper (legacy)

> **⚠️ Legacy-режим.** ZooKeeper удалён из Kafka 4.0. Этот раздел сохранён для понимания legacy-систем. Для новых проектов используйте исключительно KRaft.

### Архитектура ZooKeeper-кластера

```
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ZooKeeper │  │ZooKeeper │  │ZooKeeper │   ← Ensemble (нечётное число)
  │  node 1  │  │  node 2  │  │  node 3  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └──────────────┼──────────────┘
                      │
  ┌───────────────────┼───────────────────┐
  │                   │                   │
  ▼                   ▼                   ▼
┌──────┐           ┌──────┐           ┌──────┐
│Broker│           │Broker│           │Broker│
│  1   │           │  2   │           │  3   │
└──────┘           └──────┘           └──────┘
```

### Конфигурация ZooKeeper

```properties
# config/zookeeper.properties (для каждой ZK-ноды)
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=true
admin.serverPort=8080

# Ensemble (на всех трёх нодах одинаковый!)
server.1=zk1.example.com:2888:3888
server.2=zk2.example.com:2888:3888
server.3=zk3.example.com:2888:3888

# Performance
tickTime=2000
initLimit=10
syncLimit=5
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```

**Создание myid-файла** (уникальный идентификатор в ensemble):

```bash
# На zk1:
echo "1" | sudo tee /var/lib/zookeeper/myid
# На zk2:
echo "2" | sudo tee /var/lib/zookeeper/myid
# На zk3:
echo "3" | sudo tee /var/lib/zookeeper/myid
```

### Конфигурация брокера с ZooKeeper

```properties
# В server.properties брокера (ZK-режим):
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
zookeeper.connection.timeout.ms=18000

# ВНИМАНИЕ: параметры process.roles, node.id, controller.quorum.voters
# в ZK-режиме НЕ ИСПОЛЬЗУЮТСЯ!
```

---

## Multi-broker кластер: полная настройка

Разберём деплой production-кластера из 3 брокеров в KRaft-режиме (isolated: 3 контроллера + 3 брокера).

### План развёртывания

```yaml
# Инвентарь кластера:
controllers:
  - controller1.example.com (node.id=1)
  - controller2.example.com (node.id=2)
  - controller3.example.com (node.id=3)

brokers:
  - broker1.example.com (broker.id=1)
  - broker2.example.com (broker.id=2)
  - broker3.example.com (broker.id=3)

# Порты:
#   CONTROLLER: 9093 (между контроллерами)
#   BROKER_PLAINTEXT: 9092 (клиенты)
```

### Шаг 1: DNS и hostname

```bash
# На каждой ноде — /etc/hosts (или настройка DNS)
192.168.1.11 controller1.example.com controller1
192.168.1.12 controller2.example.com controller2
192.168.1.13 controller3.example.com controller3
192.168.1.21 broker1.example.com broker1
192.168.1.22 broker2.example.com broker2
192.168.1.23 broker3.example.com broker3
```

### Шаг 2: Генерация cluster-id и форматирование

```bash
# На controller1:
CLUSTER_ID=$(/opt/kafka/bin/kafka-storage.sh random-uuid)
echo $CLUSTER_ID  # Например: 7f3b2a1e-9c4d-4e5f-8a6b-1c2d3e4f5a6b

# Скопировать CLUSTER_ID на ВСЕ 6 нод
# На КАЖДОЙ ноде выполнить:
/opt/kafka/bin/kafka-storage.sh format \
  --config /opt/kafka/config/server.properties \
  --cluster-id $CLUSTER_ID
```

### Шаг 3: Запуск (правильный порядок!)

```bash
# 1. Сначала ВСЕ контроллеры
# На controller1, controller2, controller3:
sudo systemctl start kafka

# 2. Проверяем кворум контроллеров
/opt/kafka/bin/kafka-metadata-quorum.sh \
  --bootstrap-server controller1:9093 describe --status
# Status: LeaderId: 1, ...

# 3. Затем ВСЕ брокеры
# На broker1, broker2, broker3:
sudo systemctl start kafka
```

### Шаг 4: Создание топика и проверка

```bash
# Создание топика с RF=3, 3 партиции
/opt/kafka/bin/kafka-topics.sh --create \
  --bootstrap-server broker1:9092,broker2:9092,broker3:9092 \
  --replication-factor 3 \
  --partitions 3 \
  --topic test-topic

# Проверка
/opt/kafka/bin/kafka-topics.sh --describe \
  --bootstrap-server broker1:9092 \
  --topic test-topic

# Тестовая запись
echo "Hello Production Kafka!" | /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server broker1:9092,broker2:9092,broker3:9092 \
  --topic test-topic

# Тестовое чтение
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server broker1:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 1
```

### Шаг 5: Тест отказоустойчивости

```bash
# Остановить broker1 (имитация сбоя)
sudo systemctl stop kafka

# Проверить состояние топика — все партиции должны иметь лидера
/opt/kafka/bin/kafka-topics.sh --describe \
  --bootstrap-server broker2:9092 \
  --topic test-topic

# Leader для всех партиций должен быть broker2 или broker3
# Упавшая нода показывается как "Replicas: 3,1,2  Isr: 3,2" (1 отсутствует в ISR)
```

---

## Docker: деплой через Docker Compose

Docker-деплой удобен для разработки и тестирования. Для production используется реже, но возможен при правильной конфигурации volumes и network.

### docker-compose.yml: 3 брокера, KRaft combined mode (dev/test)

```yaml
version: '3.8'

services:
  kafka1:
    image: apache/kafka:4.1.0
    container_name: kafka1
    hostname: kafka1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_LISTENERS: "PLAINTEXT://kafka1:9092,CONTROLLER://kafka1:9093"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://localhost:9092"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka1:9093,2@kafka2:9093,3@kafka3:9093"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_LOG_RETENTION_HOURS: 168
      CLUSTER_ID: "7f3b2a1e-9c4d-4e5f-8a6b-1c2d3e4f5a6b"
    ports:
      - "9092:9092"
    volumes:
      - kafka1-data:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka2:
    image: apache/kafka:4.1.0
    container_name: kafka2
    hostname: kafka2
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_LISTENERS: "PLAINTEXT://kafka2:9092,CONTROLLER://kafka2:9093"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://localhost:9093"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka1:9093,2@kafka2:9093,3@kafka3:9093"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      CLUSTER_ID: "7f3b2a1e-9c4d-4e5f-8a6b-1c2d3e4f5a6b"
    ports:
      - "9093:9092"
    volumes:
      - kafka2-data:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka3:
    image: apache/kafka:4.1.0
    container_name: kafka3
    hostname: kafka3
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_LISTENERS: "PLAINTEXT://kafka3:9092,CONTROLLER://kafka3:9093"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://localhost:9094"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka1:9093,2@kafka2:9093,3@kafka3:9093"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      CLUSTER_ID: "7f3b2a1e-9c4d-4e5f-8a6b-1c2d3e4f5a6b"
    ports:
      - "9094:9092"
    volumes:
      - kafka3-data:/var/lib/kafka/data
    networks:
      - kafka-net

  # Опционально: kafka-ui для визуализации
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-kraft
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka1:9092,kafka2:9092,kafka3:9092
    networks:
      - kafka-net
    depends_on:
      - kafka1
      - kafka2
      - kafka3

volumes:
  kafka1-data:
  kafka2-data:
  kafka3-data:

networks:
  kafka-net:
    driver: bridge
```

**Важные замечания по Docker-деплою:**
- `advertised.listeners` для каждого контейнера пробрасывается на `localhost` с разными портами — так клиенты с хоста могут подключаться
- `CLUSTER_ID` должен быть одинаковым для всех брокеров — сгенерируйте через `kafka-storage.sh random-uuid` и зафиксируйте
- Для production НЕ используйте combined mode (`broker,controller`) — разделите роли
- Volumes обязательны для сохранения данных между перезапусками
- В production добавьте `ulimits: nofile: 65536` на каждый сервис

См. также эталонные Docker Compose-файлы от Conduktor: [github.com/conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose) — 3.6K+ звёзд, покрывают различные сценарии.

---

## Kubernetes: деплой через Strimzi Operator

Strimzi — CNCF-проект (Incubating), оператор Kubernetes для управления Kafka-кластерами. Превращает Kafka в native K8s-ресурс: вместо ручной настройки вы описываете желаемое состояние в YAML, а оператор его поддерживает.

### Установка Strimzi Operator

```bash
# Добавление Helm-репозитория
helm repo add strimzi https://strimzi.io/charts/
helm repo update

# Установка оператора в namespace kafka
kubectl create namespace kafka
helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace kafka \
  --set watchAnyNamespace=true
```

Проверка:

```bash
kubectl get pods -n kafka
# Должен появиться pod strimzi-cluster-operator-*
```

### Деплой Kafka-кластера (KRaft, 3 брокера + 3 контроллера)

```yaml
# kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 20Gi
        deleteClaim: false

---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false

---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 4.1.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: external
        port: 9094
        type: loadbalancer
        tls: false
    config:
      default.replication.factor: 3
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      auto.create.topics.enable: "false"
      log.retention.hours: 168
      num.partitions: 3
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```bash
kubectl apply -f kafka-cluster.yaml

# Мониторинг развёртывания
kubectl get pods -n kafka -w
```

**Что делает Strimzi автоматически:**
- Разворачивает брокеры и контроллеры как StatefulSet'ы
- Настраивает PersistentVolumeClaims для хранения данных
- Управляет TLS-сертификатами (внутренние и внешние)
- Настраивает listeners и advertised.listeners
- Обеспечивает rolling update при изменении конфигурации
- Topic Operator: создаёт/управляет топиками через `KafkaTopic` CRD
- User Operator: создаёт/управляет пользователями и ACL через `KafkaUser` CRD

### Создание топика через Strimzi CRD

```yaml
# my-topic.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000  # 7 дней
    cleanup.policy: delete
```

```bash
kubectl apply -f my-topic.yaml
kubectl get kafkatopics -n kafka
```

### Strimzi: production-рекомендации

1. **Storage:** используйте `type: persistent-claim` с `deleteClaim: false` — данные не удаляются при удалении StatefulSet
2. **Resources:** задайте `resources.limits` и `resources.requests` для каждого NodePool
3. **Anti-affinity:** Strimzi по умолчанию распределяет поды по разным нодам (`podAntiAffinity`)
4. **Metrics:** включите `metricsConfig` для интеграции с Prometheus
5. **TLS:** включите шифрование между брокерами (`tls: true`)
6. **Cruise Control:** Strimzi поддерживает Cruise Control для автоматической ребалансировки партиций

---

## KRaft vs ZooKeeper: сравнительный анализ

### Эволюция от ZooKeeper к KRaft

| Период | Версия Kafka | Статус ZooKeeper | Статус KRaft |
|--------|-------------|-------------------|---------------|
| 2011-2021 | 0.7 — 2.7 | Единственный режим | — |
| 2021 | 2.8 | Default | Early Access (KIP-500) |
| 2022 | 3.3 | Default | Production-ready для новых кластеров |
| 2024 | 3.9 | Deprecated | Рекомендован для новых проектов |
| 2025 | 4.0 | **Удалён** | Default, единственный режим |

### Техническое сравнение

| Характеристика | ZooKeeper | KRaft |
|----------------|-----------|-------|
| **Компоненты** | ZK ensemble (3-5 нод) + Kafka brokers | Только Kafka (controllers + brokers) |
| **Процессы для мониторинга** | 2 типа (ZK + Kafka) | 1 тип (Kafka) |
| **Модель данных метаданных** | Древовидная (ZNode) | Event-sourced log (метадата-топик) |
| **Failover контроллера** | Загрузка состояния из ZK (секунды) | Мгновенный (логи в памяти) |
| **Максимум партиций** | ~200K на кластер | Миллионы на кластер |
| **Single security model** | 2 разные модели (ZK + Kafka) | Единая модель |
| **Конфигурация** | `zookeeper.connect` + `server.properties` | Только `server.properties` |
| **Стартовый порог** | ZK ensemble + 1 брокер | 1 процесс (combined mode dev) |
| **Простота начала работы** | Требует понимания ZK | Быстрый старт, одно приложение |

### Ключевое архитектурное отличие: контроллер

**ZooKeeper-контроллер (legacy):**
1. Брокер избирается контроллером через ZooKeeper
2. Контроллер загружает ВСЕ метаданные из ZooKeeper
3. При сбое контроллера: новый контроллер загружает состояние заново (задержка)
4. Метаданные между брокерами синхронизируются через RPC

**KRaft-контроллер:**
1. Кворум RAFT-контроллеров управляет метаданными
2. Event-sourced лог: все изменения записываются в метадата-топик
3. При сбое: новый лидер уже имеет все коммиченные записи в памяти
4. Метаданные реплицируются через RAFT-лог — тот же механизм, что и данные

**Практический эффект:** KRaft устраняет «режим догоняния» (catch-up mode) — контроллер не должен загружать состояние после failover'а. Время восстановления сокращается с секунд до миллисекунд. [Confluent — KRaft Overview]

---

## Production Checklist

Чек-лист для проверки готовности production-кластера. Пройдите по каждому пункту перед приёмом трафика.

### 📋 Планирование и железо

- [ ] Минимум 3 брокера (для RF=3)
- [ ] Минимум 3 контроллера в KRaft isolated mode (или 3 ZK в ensemble)
- [ ] Каждый брокер на отдельной физической машине / availability zone
- [ ] 8+ CPU ядер, 32+ GB RAM, SSD-диски для `log.dirs`
- [ ] Выделенные сетевые интерфейсы (10 Gbps)
- [ ] DNS-имена (FQDN) для всех брокеров и контроллеров

### 🐧 ОС и система

- [ ] `vm.swappiness=1` (минимизировать swap для page cache)
- [ ] `vm.max_map_count=262144` (для 50K+ партиций)
- [ ] File descriptors: минимум 65536 (`nofile`)
- [ ] XFS с `noatime` для data-дисков
- [ ] Java 17 (брокеры), 11 (клиенты)

### ⚙️ Конфигурация брокера

- [ ] `broker.id` уникален для каждого брокера
- [ ] `listeners` и `advertised.listeners` настроены корректно (FQDN!)
- [ ] `default.replication.factor=3`
- [ ] `min.insync.replicas=2` (при RF=3)
- [ ] `unclean.leader.election.enable=false`
- [ ] `auto.create.topics.enable=false`
- [ ] `offsets.topic.replication.factor=3`
- [ ] `transaction.state.log.replication.factor=3`
- [ ] `transaction.state.log.min.isr=2`
- [ ] `log.retention.hours` соответствует бизнес-требованиям
- [ ] `log.retention.check.interval.ms=300000` (5 минут)
- [ ] `log.dirs` на отдельных физических дисках, не на системном

### 🛡️ Безопасность

- [ ] TLS/SSL для межброкерного взаимодействия (`inter.broker.listener.name`)
- [ ] mTLS или SASL для клиентского доступа
- [ ] ACL настроены (принцип наименьших привилегий)
- [ ] Пароли/сертификаты не в конфигах, а в защищённом хранилище (Vault/secrets)
- [ ] Файервол: открыты только порты 9092-9095

### 📊 Мониторинг

- [ ] JMX включён (порт 9999)
- [ ] Prometheus JMX Exporter настроен
- [ ] Grafana-дашборды: broker throughput, under-replicated partitions, consumer lag
- [ ] Алерты: UnderReplicatedPartitions > 0, ActiveControllerCount ≠ 1, OfflinePartitions > 0
- [ ] Мониторинг использования диска (alert на 85%+)

### 🔄 Операционные процедуры

- [ ] Документирована процедура добавления нового брокера
- [ ] Документирована процедура замены упавшего брокера
- [ ] Настроен механизм бэкапа конфигураций (GitOps)
- [ ] Kafka Cruise Control или аналог для ребалансировки
- [ ] Процедура обновления версии Kafka протестирована (rolling upgrade)

### 🔬 Функциональное тестирование

- [ ] Создание/удаление топиков работает
- [ ] Запись/чтение с требуемой пропускной способностью
- [ ] Отказ одного брокера: сервис продолжает работу
- [ ] Отказ одного контроллера: кворум сохраняется
- [ ] Consumer group rebalance работает корректно
- [ ] Миграция с ZooKeeper на KRaft протестирована (для legacy-систем)

### 🚨 Процедуры аварийного восстановления

- [ ] RTO (Recovery Time Objective) и RPO (Recovery Point Objective) определены
- [ ] Процедура восстановления кластера из бэкапа протестирована
- [ ] MirrorMaker 2 настроен для cross-DC репликации (при необходимости)

---

## Источники

1. [Apache Kafka Documentation — Hardware and OS](https://kafka.apache.org/39/operations/hardware-and-os/) — официальная документация по требованиям к железу и ОС
2. [Apache Kafka 4.0.0 Release Announcement](https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/) — официальный анонс Kafka 4.0, удаление ZooKeeper
3. [Confluent Developer — KRaft: Apache Kafka Without ZooKeeper](https://developer.confluent.io/learn/kraft/) — обзор архитектуры KRaft от Confluent
4. [KLogic — Kafka Cluster Setup Guide](https://klogic.io/guides/kafka-cluster-setup-guide/) — production-ready deployment tutorial (2025)
5. [Harry The DevOps Guy — Setup Production-ready Kafka Cluster with KRaft](https://harrythedevopsguy.github.io/articles/post/2024/10/05/Setup-Production-Ready-Kafka-Cluster-with-kraft.html) — step-by-step KRaft deployment
6. [GitHub — conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose) — эталонные docker-compose конфигурации (3.6K+ звёзд)
7. [Strimzi Documentation — Deploying and Managing](https://strimzi.io/docs/operators/latest/full/deploying) — официальная документация Strimzi
8. [Conduktor Blog — KRaft Explained: Kafka Without ZooKeeper](https://www.conduktor.io/blog/kraft-explained-kafka-without-zookeeper) — глубокий разбор KRaft vs ZooKeeper
9. [Apache Kafka KRaft Overview](https://kafka.apache.org/35/operations/kraft/) — официальная документация по KRaft-режиму
10. [Apache Kafka GitHub — config/server.properties](https://github.com/apache/kafka/blob/trunk/config/server.properties) — эталонный конфигурационный файл брокера (trunk)
11. [Jacek Laskowski — The Internals of Apache Kafka: Properties](https://jaceklaskowski.gitbooks.io/apache-kafka/kafka-properties.html) — детальный справочник по конфигурационным параметрам
