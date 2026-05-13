# 07.01 — Мониторинг Apache Kafka: метрики, инструменты и алертинг

> **Bottom line:** Мониторинг Kafka — это не про «красивые графики». Это про способность заметить проблему **до того**, как упадут продакшн-пайплайны. Ключевые метрики: consumer lag, under-replicated partitions, ActiveControllerCount, ISR shrink rate и утилизация ресурсов брокера. Стек Prometheus + Grafana + JMX Exporter — де-факто стандарт индустрии.

---

## 1. Зачем мониторить Kafka

Kafka работает как центральная нервная система организации: десятки приложений пишут и читают данные, и если кластер «заболевает», последствия каскадом расходятся по всей компании. Проблема в том, что Kafka **тихо деградирует** — без мониторинга вы не узнаете о проблеме, пока не получите звонок от бизнеса.

**Что мониторинг Kafka должен давать:**

- **Видимость** (Observability): что происходит внутри кластера прямо сейчас
- **Раннее предупреждение**: аномалии до того, как они перерастут в инцидент
- **Диагностика**: быстрое определение первопричины при инциденте
- **Планирование ёмкости** (Capacity Planning): когда пора добавлять брокеров
- **Валидация SLA/SLO**: подтверждение, что throughput и latency в рамках договорённостей

### Уровни мониторинга

Мониторинг Kafka выстраивается на четырёх уровнях:

```
┌──────────────────────────────────────────────────┐
│ 4. Метрики приложения (producers/consumers)       │
├──────────────────────────────────────────────────┤
│ 3. Метрики брокера (throughput, latency, ISR)     │
├──────────────────────────────────────────────────┤
│ 2. Метрики ОС (CPU, память, диск, сеть)           │
├──────────────────────────────────────────────────┤
│ 1. Health-check (жив ли процесс? слушает ли порт?) │
└──────────────────────────────────────────────────┘
```

Если нет уровня 2 — вы не увидите, что диск заполняется на 95%. Если нет уровня 3 — не узнаете, что партиция стала under-replicated. Если нет уровня 4 — не поймёте, что consumer отстаёт на 10 миллионов сообщений.

---

## 2. Источник метрик: JMX

Kafka использует **Yammer Metrics** на серверной стороне и **Kafka Metrics** (встроенный реестр) на клиентской стороне. Оба фреймворка экспортируют метрики через **JMX** (Java Management Extensions) и могут быть сконфигурированы на отправку через pluggable reporters.

### Включение JMX

JMX **выключен по умолчанию** — это осознанное решение безопасности. Для включения задаётся переменная окружения `JMX_PORT` перед запуском брокера:

```bash
# В production — только с аутентификацией!
export JMX_PORT=9999
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote.port=$JMX_PORT \
  -Dcom.sun.management.jmxremote.authenticate=true \
  -Dcom.sun.management.jmxremote.ssl=true \
  -Dcom.sun.management.jmxremote.password.file=/path/to/jmxremote.password"

kafka-server-start.sh config/server.properties
```

В production **обязательно** включить аутентификацию и SSL для JMX. Без этого любой, кто может соединиться с JMX-портом, получит полный доступ к управлению JVM (включая возможность вызвать `Thread.dump()` или даже выполнить произвольные операции через MBeans).

### MBean-иерархия

Метрики организованы в иерархию MBeans (Managed Beans):

```
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=my_topic
│             │    │                   │
│             │    │                   └── имя метрики
│             │    └── тип компонента
│             └── домен (kafka.*)
```

**Основные домены и их назначение:**

| Домен | Что мониторит |
|-------|---------------|
| `kafka.server:type=BrokerTopicMetrics` | Throughput по топикам (BytesIn/Out, MessagesIn) |
| `kafka.server:type=ReplicaManager` | Состояние репликации (ISR, under-replicated) |
| `kafka.controller:type=KafkaController` | Активность контроллера |
| `kafka.controller:type=ControllerEventManager` | Очередь событий контроллера |
| `kafka.network:type=RequestMetrics` | Сетевые запросы (Produce, Fetch, latency) |
| `kafka.network:type=SocketServer` | Idle-процент сетевых процессоров |
| `kafka.server:type=KafkaRequestHandlerPool` | Idle-процент обработчиков запросов |
| `kafka.log:type=LogManager` | Состояние лог-директорий |

---

## 3. Ключевые метрики: что и зачем

Метрик в Kafka — сотни. Мониторить все — прямой путь к alert fatigue. Вот **минимально необходимый набор** для production, разделённый по сигналам.

### 3.1. Сигнал: «Кластер жив и здоров»

#### ActiveControllerCount

**MBean:** `kafka.controller:type=KafkaController,name=ActiveControllerCount`

**Почему важно:** Контроллер управляет лидерами партиций. Его отсутствие — кластер парализован. Два контроллера — split-brain (в KRaft невозможен по дизайну).

**Норма:** Ровно **1** на весь кластер.  
**Alert:** `ActiveControllerCount != 1`

#### OfflinePartitionsCount

**MBean:** `kafka.controller:type=KafkaController,name=OfflinePartitionsCount` (для старых версий), или `kafka.server:type=ReplicaManager,name=OfflineReplicaCount`

**Почему важно:** Офлайн-партиция = данные недоступны для чтения/записи. Это прямой инцидент.

**Норма:** `0`  
**Alert:** `> 0` — немедленный critical-алерт

#### UnderReplicatedPartitions

**MBean:** `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`

**Почему важно:** Партиция under-replicated, когда хотя бы одна реплика не входит в ISR (In-Sync Replica). Это означает, что при падении лидера возможна потеря данных или недоступность.

**Норма:** `0`  
**Alert:** `> 0` в течение > 5 минут → warning; `> 0` более 15 минут → critical

#### UnderMinIsrPartitionCount

**MBean:** `kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount`

**Почему важно:** Если ISR < `min.insync.replicas`, продюсеры с `acks=all` **не могут писать** в эту партицию. Это остановка записи.

**Норма:** `0`  
**Alert:** `> 0` → critical немедленно

### 3.2. Сигнал: «Throughput и нагрузка»

#### BytesInPerSec / BytesOutPerSec

**MBean:** `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` и `name=BytesOutPerSec`

**Почему важно:** Базовые показатели трафика. Аномальное падение или рост — сигнал проблемы.

**Норма:** Зависит от профиля кластера. Важнее тренд, чем абсолютное значение.  
**Alert:** Резкое падение throughput (>50% от baseline за 5 минут) → warning

#### MessagesInPerSec

**MBean:** `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec`

**Почему важно:** Количество сообщений в секунду. Резкое падение может означать проблему на стороне продюсеров или сети.

**Норма:** Зависит от нагрузки.  
**Alert:** Падение ниже 20% от скользящего среднего за 1 час → warning

#### RequestsPerSec

**MBean:** `kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce|FetchConsumer|FetchFollower`

**Почему важно:** Разделение по типам запросов позволяет понять, где проблема: продюсеры не пишут или консьюмеры не читают.

**Норма:** Зависит от нагрузки.  
**Alert:** Резкое изменение соотношения Produce/Fetch запросов.

### 3.3. Сигнал: «Consumer Lag»

**Это, возможно, самая важная метрика Kafka.**

Consumer lag — разница между последним записанным офсетом (latest offset) и последним прочитанным офсетом (consumer offset) для каждой партиции. Высокий lag означает, что консьюмер не успевает обрабатывать данные.

#### Источники consumer lag

1. **JMX напрямую с брокера:** `kafka.server:type=FetcherLagMetrics,name=ConsumerLag,clientId=*,topic=*,partition=*`
2. **`kafka-consumer-groups.sh`:** CLI-инструмент — `kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group my-group --describe`
3. **`kafka_exporter`** (от Danielqs): отдельный экспортер, собирает lag через AdminClient API
4. **Burrow** (от LinkedIn): специализированный монитор consumer lag с оценкой статуса группы (OK/WARNING/ERROR)

#### Пороги для алертов

Consumer lag — это **не абсолютная величина**. 10 000 сообщений lag для low-throughput топика — критично, для high-throughput топика (100K msg/s) — норма.

**Правильный подход — rate-based алертинг:**

```yaml
# Prometheus alert rule
- alert: KafkaConsumerLag
  expr: |
    (
      kafka_consumer_group_lag > 10000
    )
    and
    (
      rate(kafka_consumer_group_lag[5m]) > 0
    )
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Consumer group {{ $labels.consumer_group }} lagging"
```

Ключевая идея: алертить только если lag **растёт** (rate > 0), а не просто большой. Постоянный lag в 100K без роста — это штатная ситуация при постоянной нагрузке.

**Рекомендуемые пороги:**
- Lag растёт > 10 минут → warning
- Lag растёт > 30 минут → critical
- `time_lag` (lag в секундах, а не сообщениях) > 300 (5 минут) → warning

### 3.4. Сигнал: «Ресурсы брокера»

#### RequestHandlerAvgIdlePercent

**MBean:** `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent`

**Почему важно:** Показывает, насколько загружены потоки-обработчики. Idle < 0.2 означает, что брокер перегружен.

**Норма:** > 0.3 (30% idle)  
**Alert:** < 0.2 → warning; < 0.1 → critical

#### NetworkProcessorAvgIdlePercent

**MBean:** `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent`

**Почему важно:** Сетевые процессоры обрабатывают входящие соединения. Их перегрузка приводит к росту latency и отбрасыванию соединений.

**Норма:** > 0.3  
**Alert:** < 0.2 → warning

#### RequestQueueSize

**MBean:** `kafka.network:type=RequestChannel,name=RequestQueueSize`

**Почему важно:** Очередь входящих запросов. Если она растёт — брокер не справляется.

**Норма:** Близко к 0  
**Alert:** > 500 в течение 5 минут → warning

#### OS-level метрики (обязательно!)

| Ресурс | Что мониторить | Порог тревоги |
|--------|---------------|---------------|
| **CPU** | `cpu_usage_percent` | > 80% sustained > 10 мин |
| **Память** | `heap_used / heap_max` (JVM heap) | > 85% → warning |
| **Диск** | `disk_usage_percent` на log.dirs | > 80% → warning; > 90% → critical |
| **Сеть** | `net_bytes_sent/recv`, ошибки сетевого интерфейса | Ошибки > 0 → warning |
| **GC** | `jvm_gc_pause_seconds` (особенно Full GC) | P99 > 100ms → warning |
| **FD** | Открытые file descriptors | > 80% от `ulimit -n` → warning |

### 3.5. Сигнал: «ISR и репликация»

#### IsrShrinksPerSec / IsrExpandsPerSec

**MBean:** `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` и `name=IsrExpandsPerSec`

**Почему важно:** ISR shrink = реплика выпала из синхронизированного набора. Причина: брокер упал, сеть деградировала, GC pause, дисковая задержка. Частые shrink'и — кластер нестабилен.

**Норма:** `0` (за исключением плановых рестартов брокеров)  
**Alert:** Любой shrink → warning (если не плановый)

#### ReplicationBytesInPerSec

**MBean:** `kafka.server:type=BrokerTopicMetrics,name=ReplicationBytesInPerSec`

**Почему важно:** Трафик репликации. Аномальный рост может означать, что брокер догоняет после восстановления.

#### FailedIsrUpdatesPerSec

**MBean:** `kafka.server:type=ReplicaManager,name=FailedIsrUpdatesPerSec`

**Норма:** `0`  
**Alert:** > 0 → warning

### 3.6. Сигнал: «Ошибки и отклонения»

#### ErrorsPerSec

**MBean:** `kafka.network:type=RequestMetrics,name=ErrorsPerSec,request=*,error=*`

**Почему важно:** Считает ошибки по типам запросов. Рост ошибок — верный признак проблемы.

**Ключевые error-коды для алертинга:**
- `NOT_LEADER_FOR_PARTITION` — клиент шлёт запрос не тому брокеру (возможно, устарели метаданные)
- `OFFSET_OUT_OF_RANGE` — консьюмер запрашивает несуществующий офсет
- `NETWORK_EXCEPTION` — проблемы с сетью
- `REQUEST_TIMED_OUT` — брокер не ответил вовремя

#### TotalTimeMs / RequestQueueTimeMs / LocalTimeMs / RemoteTimeMs

**MBean:** `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce|FetchConsumer`

**Почему важно:** Раскладывает latency запроса на компоненты:
- `RequestQueueTimeMs` — ожидание в очереди (проблема: перегрузка брокера)
- `LocalTimeMs` — обработка на лидере (проблема: медленный диск/CPU)
- `RemoteTimeMs` — ожидание подтверждения от follower'ов при `acks=all` (проблема: медленная репликация)
- `ResponseSendTimeMs` — отправка ответа (проблема: сеть)

### 3.7. Специфические метрики KRaft-режима

С переходом на KRaft (Kafka 3.3+, production-ready с 3.6+) вместо ZooKeeper-метрик появляется новый набор:

| Метрика | MBean | Описание |
|---------|-------|----------|
| **ActiveControllerCount** | `kafka.controller:type=KafkaController` | То же, что и раньше |
| **CurrentMetadataVersion** | `kafka.server:type=MetadataLoader` | Версия загруженных метаданных |
| **LastAppliedRecordOffset** | `kafka.server:type=KRaftMetadataManager` | Последний применённый офсет метаданных |
| **EventQueueSize** | `kafka.controller:type=ControllerEventManager,name=EventQueueSize` | Очередь событий контроллера |
| **EventQueueTimeMs** | `kafka.controller:type=ControllerEventManager,name=EventQueueTimeMs` | Время ожидания событий |

**Что уходит:** Все `kafka.zookeeper:*` и `kafka.server:type=SessionExpireListener` метрики становятся нерелевантными.

### 3.8. Специфические метрики продюсера и консьюмера

**Для продюсера:**
| Метрика (клиентская) | Что показывает |
|----------------------|----------------|
| `record-send-rate` | Скорость отправки записей |
| `record-error-rate` | Доля ошибок отправки |
| `record-retry-rate` | Частота повторных отправок (растёт = проблемы) |
| `request-latency-avg` | Средняя latency запроса |
| `buffer-available-bytes` | Свободное место в буфере (0 = продюсер заблокирован) |
| `waiting-threads` | Потоки, ожидающие места в буфере (>0 = bottleneck) |

**Для консьюмера:**
| Метрика (клиентская) | Что показывает |
|----------------------|----------------|
| `records-lag-max` | Максимальный lag по партициям группы |
| `records-consumed-rate` | Скорость потребления |
| `fetch-rate` | Частота fetch-запросов |
| `bytes-consumed-rate` | Пропускная способность консьюмера |
| `fetch-latency-avg` | Средняя latency fetch-запроса |

---

## 4. Инструменты мониторинга

### 4.1. Архитектура Prometheus + Grafana (де-факто стандарт)

```
┌──────────┐    ┌───────────────┐    ┌──────────┐    ┌─────────┐
│  Kafka   │───▶│ JMX Exporter  │───▶│Prometheus│───▶│ Grafana │
│  Broker  │    │  (java agent) │    │          │    │         │
└──────────┘    └───────────────┘    └──────────┘    └─────────┘
                                                     │         │
┌──────────┐                                         │ Алерты  │
│  Kafka   │──▶ kafka_exporter ──────────────────────▶│AlertMgr │
│  Cluster │    (Danielqs)                           └─────────┘
└──────────┘
```

**Три компонента сбора метрик:**

1. **JMX Exporter** (java agent от Prometheus): запускается как `-javaagent` вместе с каждым брокером. Читает JMX-метрики JVM и экспортирует их как HTTP-endpoint `/metrics`.

2. **kafka_exporter** (danielqsj): отдельный процесс. Через Kafka AdminClient собирает consumer lag, топик-метрики, информацию о партициях. Опрашивает кластер, а не отдельный процесс.

3. **Node Exporter**: собирает метрики ОС (CPU, диск, сеть).

**Конфигурация JMX Exporter (kafka_broker.yml):**

```yaml
# jmx_exporter_config.yml
startDelaySeconds: 0
ssl: false
lowercaseOutputName: true
lowercaseOutputLabelNames: true

# Правила — какие MBeans экспортировать
rules:
  # Throughput
  - pattern: kafka.server<type=BrokerTopicMetrics, name=(BytesInPerSec|BytesOutPerSec|MessagesInPerSec)><>(Count|OneMinuteRate)
    name: kafka_server_brokertopicmetrics_$1_$2
    labels:
      topic: "$3"

  # Under-replicated partitions
  - pattern: kafka.server<type=ReplicaManager, name=(UnderReplicatedPartitions|UnderMinIsrPartitionCount)><>Value
    name: kafka_server_replicamanager_$1

  # Active Controller
  - pattern: kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value
    name: kafka_controller_activecontroller_count

  # Request handler idle
  - pattern: kafka.server<type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent><>OneMinuteRate
    name: kafka_server_requesthandler_idle_percent

  # ISR changes
  - pattern: kafka.server<type=ReplicaManager, name=(IsrShrinksPerSec|IsrExpandsPerSec)><>Count
    name: kafka_server_replicamanager_$1

  # Controller event queue
  - pattern: kafka.controller<type=ControllerEventManager, name=(EventQueueSize|EventQueueTimeMs)><>Value
    name: kafka_controller_eventmanager_$1

  # Network request metrics
  - pattern: kafka.network<type=RequestMetrics, name=RequestsPerSec, request=(.+)><>Count
    name: kafka_network_requests_total
    labels:
      request_type: "$1"

  - pattern: kafka.network<type=RequestMetrics, name=TotalTimeMs, request=(.+)><>Mean
    name: kafka_network_request_latency_mean
    labels:
      request_type: "$1"

  # JVM — память и GC
  - pattern: java.lang<type=Memory><HeapMemoryUsage>(used|max)
    name: jvm_memory_heap_$1

  - pattern: java.lang<type=GarbageCollector, name=(.+)><>(CollectionCount|CollectionTime)
    name: jvm_gc_$2
    labels:
      collector: "$1"
```

**Запуск брокера с JMX Exporter:**

```bash
export KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=7071:/opt/jmx_exporter/kafka_broker.yml"
kafka-server-start.sh config/server.properties
```

После этого брокер на порту `7071` отдаёт `/metrics` в формате Prometheus.

### 4.2. Grafana Dashboard

Для Kafka существует несколько production-ready дашбордов:

| Дашборд | Источник | Покрытие |
|---------|----------|----------|
| **Confluent jmx-monitoring-stacks** | [GitHub: confluentinc/jmx-monitoring-stacks](https://github.com/confluentinc/jmx-monitoring-stacks) | Kafka Cluster, ZK, KRaft, Connect, Schema Registry, ksqlDB, Producer/Consumer, Lag, Topics, Quotas, Tiered Storage, Flink |
| **Kafka Exporter Overview** | Grafana.com Dashboard #7589 | Общий обзор через kafka_exporter |
| **Kafka Dashboard (Strimzi)** | Встроен в Strimzi operator | Для Kubernetes-деплойментов |
| **AKHQ** | Kafka GUI с базовыми метриками | Альтернатива для малых кластеров |

**Минимальный набор панелей на дашборде:**

1. **Cluster Health** (верхняя строка):
   - Active Controller (0/1)
   - Under-Replicated Partitions
   - Offline Partitions
   - ISR Shrink/Expand rate

2. **Throughput** (вторая строка):
   - Bytes In / Out (суммарно и per-broker)
   - Messages In per second
   - Produce / Fetch request rate

3. **Consumer Lag** (третья строка):
   - Max consumer lag per group
   - Тренд lag за 1h/6h/24h
   - Количество консьюмер-групп с растущим lag

4. **Broker Resources** (четвёртая строка):
   - CPU, heap memory, disk usage per broker
   - Request Handler Idle %
   - Network Processor Idle %
   - GC pause time

5. **Request Latency** (пятая строка):
   - P50/P95/P99 produce latency
   - P50/P95/P99 fetch latency
   - Queue / Local / Remote time breakdown

### 4.3. Confluent Control Center (Enterprise)

Для организаций, использующих Confluent Platform, Control Center предоставляет opinionated-мониторинг «из коробки»:

- Автоматическая агрегация метрик брокеров
- Визуализация consumer lag с историей
- End-to-end latency мониторинг (producer → broker → consumer)
- Интеграция с Confluent Metrics Reporter
- System health overview с алертами
- Stream Lineage для отслеживания потоков данных

**Важно:** Control Center — это коммерческий продукт, входящий в Confluent Platform. Для open-source Kafka используется связка Prometheus + Grafana.

### 4.4. Burrow (LinkedIn)

Burrow — специализированный монитор consumer lag, созданный в LinkedIn:

**Отличие от kafka_exporter:** Burrow оценивает не абсолютный lag, а **состояние** (status) консьюмер-группы.

**Оценки Burrow:**
- `OK` — консьюмер догоняет или стабильно отстаёт на постоянную величину
- `WARNING` — lag растёт, но консьюмер всё ещё активен
- `ERROR` — партиция застопорилась (consumer не двигается) или группа не существует
- `STOP` — группа в состоянии STOP (Kafka 4.0+ новый протокол)

**HTTP API Burrow:**
```bash
# Список consumer groups
curl http://burrow:8000/v3/kafka/local/consumer

# Lag конкретной группы
curl http://burrow:8000/v3/kafka/local/consumer/my-group/lag

# Статус группы
curl http://burrow:8000/v3/kafka/local/consumer/my-group/status
```

Burrow экспортирует собственные метрики для Prometheus, которые можно добавить на общий дашборд.

### 4.5. Managed-решения

| Сервис | Что даёт |
|--------|----------|
| **Confluent Cloud** | Встроенная Observability (Metrics API + Cloud UI) |
| **AWS MSK** | CloudWatch метрики (Open Monitoring с Prometheus) |
| **Azure Event Hubs** | Azure Monitor метрики |
| **Google Cloud Pub/Sub** | Cloud Monitoring |
| **Datadog Kafka Integration** | Агент + преднастроенные дашборды |
| **New Relic Kafka** | JMX → NR агент |

---

## 5. Алертинг: правила и стратегия

### Принципы алертинга Kafka

1. **Алертить на симптомы, а не на причины.** «Consumer lag растёт» — симптом. «Диск брокера 3 заполнен» — причина. Первое алертит дежурного, второе — помогает диагностировать.
2. **Разделять severity.** Не всё требует будить инженера в 3 часа ночи.
3. **Использовать `for` (длительность).** Спайки метрик — норма. Алерт должен срабатывать только если условие сохраняется N минут.
4. **Группировать по сервису.** Все Kafka-алерты должны иметь общую метку `team: data-infra`.

### Prometheus Alert Rules (рекомендованный набор)

```yaml
groups:
  - name: kafka_critical
    rules:
      # Контроллер — немедленно
      - alert: KafkaNoActiveController
        expr: sum(kafka_controller_activecontroller_count) != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka: Нет активного контроллера (или split-brain)"
          description: "ActiveControllerCount = {{ $value }}. Кластер неуправляем."

      # Офлайн-партиции — немедленно
      - alert: KafkaOfflinePartitions
        expr: sum(kafka_server_replicamanager_OfflineReplicaCount) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka: Обнаружены офлайн-партиции: {{ $value }}"

      # minISR нарушен — немедленно
      - alert: KafkaUnderMinIsrPartitions
        expr: sum(kafka_server_replicamanager_UnderMinIsrPartitionCount) > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Kafka: Партиции с ISR < min.insync.replicas: {{ $value }}"
          description: "Продюсеры с acks=all НЕ МОГУТ писать в эти партиции."

  - name: kafka_warning
    rules:
      # Under-replicated
      - alert: KafkaUnderReplicatedPartitions
        expr: sum(kafka_server_replicamanager_UnderReplicatedPartitions) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: Under-replicated партиции: {{ $value }} (более 5 мин)"

      # Consumer lag растёт
      - alert: KafkaConsumerLagIncreasing
        expr: |
          kafka_consumer_group_lag > 10000
          and
          rate(kafka_consumer_group_lag[5m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: Lag растёт — группа {{ $labels.consumer_group }}"

      # Consumer lag в секундах > 5 минут
      - alert: KafkaConsumerTimeLag
        expr: kafka_consumer_group_time_lag_seconds > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: Time lag > 5мин — группа {{ $labels.consumer_group }}"

      # Перегрузка брокера
      - alert: KafkaBrokerOverloaded
        expr: kafka_server_requesthandler_idle_percent < 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: Брокер перегружен — idle < 20%"

      # ISR shrink (вне планового обслуживания)
      - alert: KafkaIsrShrink
        expr: rate(kafka_server_replicamanager_IsrShrinksPerSec[5m]) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: ISR shrink detected"

      # Диск заполняется
      - alert: KafkaDiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{mountpoint=~"/data/kafka.*"}
          / node_filesystem_size_bytes{mountpoint=~"/data/kafka.*"}) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka: Диск заполнен на {{ $value | humanizePercentage }}"

  - name: kafka_info
    rules:
      # Партиций больше N на брокер (несбалансированность)
      - alert: KafkaPartitionImbalance
        expr: |
          (
            max(kafka_server_replicamanager_PartitionCount)
            - min(kafka_server_replicamanager_PartitionCount)
          ) > 100
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "Kafka: Дисбаланс партиций между брокерами > 100"

      # Продюсеры ретраят
      - alert: KafkaProducerHighRetryRate
        expr: rate(kafka_producer_record_retry_total[5m]) > 10
        for: 10m
        labels:
          severity: info
        annotations:
          summary: "Kafka: Высокая частота retry у продюсеров"
```

### Каналы доставки алертов

| Severity | Действие |
|----------|----------|
| **critical** | PagerDuty/OpsGenie → звонок дежурному SRE + сообщение в #incidents Slack |
| **warning** | Slack #kafka-alerts + тикет в Jira |
| **info** | Только дашборд (не алертить людей) |

---

## 6. Health-check эндпоинты

Помимо метрик, каждый брокер должен предоставлять простой health-check:

### TCP port check

```bash
# Проверка, что брокер слушает порт
nc -zv kafka-broker-1 9092
```

### Kafka AdminClient check

```python
from kafka.admin import KafkaAdminClient

admin = KafkaAdminClient(bootstrap_servers="localhost:9092")
try:
    cluster_info = admin.describe_cluster()
    print(f"Cluster {cluster_info['cluster_id']}: "
          f"{len(cluster_info['brokers'])} brokers, "
          f"controller: broker {cluster_info['controller_id']}")
    # Healthy
except Exception as e:
    print(f"UNHEALTHY: {e}")
```

### Рекомендованные health-check для Kubernetes

```yaml
# kafka pod spec
livenessProbe:
  tcpSocket:
    port: 9092
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  exec:
    command:
      - /bin/bash
      - -c
      - |
        /opt/kafka/bin/kafka-broker-api-versions.sh \
          --bootstrap-server localhost:9092 > /dev/null 2>&1
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Важно:** readiness probe через `kafka-broker-api-versions.sh` проверяет, что брокер не просто слушает порт, а реально отвечает на Kafka-запросы. Это важно при старте: порт может открыться, но брокер ещё не загрузил метаданные.

---

## 7. Практические рекомендации

### 7.1. Настройка retention для Prometheus

Kafka генерирует много метрик (особенно per-partition). Prometheus TSDB может быстро разрастись.

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka-brokers'
    scrape_interval: 30s     # не чаще — JMX экспорт небыстрый
    scrape_timeout: 20s
    # retention для high-cardinality метрик
    metric_relabel_configs:
      # Дропаем per-partition метрики если > 1000 партиций
      - source_labels: [topic]
        regex: '.*'
        action: drop
        # или оставляем только ключевые
      - source_labels: [__name__]
        regex: 'kafka_server_brokertopicmetrics_(BytesInPerSec|BytesOutPerSec|MessagesInPerSec).*'
        action: keep
```

### 7.2. Кардинальность метрик — главная боль

Kafka создаёт метрики **per-topic, per-partition**. На кластере с 1000 партиций одна метрика превращается в 1000 временных рядов.

**Стратегии борьбы:**

1. **Агрегировать на уровне JMX Exporter** (схлопывать per-partition метрики в per-topic или per-broker)
2. **Дропать высококардинальные метрики** на уровне Prometheus `metric_relabel_configs`
3. **Использовать recording rules** — агрегировать per-partition метрики в Prometheus и алертить на агрегаты
4. **Выделенный Prometheus для Kafka** — не мешать метрики Kafka с метриками приложений

### 7.3. Мониторинг в multi-DC сценариях

При гео-репликации (MirrorMaker 2) добавляется специфика:

- **Replication lag между DC** — MirrorMaker 2 экспортирует метрики `MirrorSourceConnector`
- **Пропускная способность между ЦОДами** — сетевые метрики на канале репликации
- **Latency между ЦОДами** — влияет на производительность MM2

### 7.4. Мониторинг Tiered Storage (KIP-405)

При включённом Tiered Storage появляются дополнительные метрики:

| Метрика | Описание |
|---------|----------|
| `RemoteLogReaderAvgIdlePercent` | Загрузка читателей из remote storage |
| `RemoteCopyLagBytes` | Lag копирования в remote storage |
| `RemoteLogSizeBytes` | Объём данных в remote storage |

Алерты: если `RemoteCopyLagBytes` растёт — данные не успевают выгружаться в S3, риск потери данных при отказе локального диска.

### 7.5. Мониторинг KRaft metadata quorum

В KRaft-режиме метаданные хранятся в кворуме контроллеров. Специфические метрики:

| Метрика | Описание |
|---------|----------|
| `ActiveControllerCount` | Должен быть 1 |
| `CurrentState` (KRaftControllerChannelManager) | Состояние канала до других контроллеров |
| `CommitOffset` (KRaftMetadataManager) | Позиция committed offset |

Алерт: если метаданные не коммитятся (commit offset стоит на месте) — контроллер не может выбрать лидеров, кластер постепенно деградирует.

### 7.6. Чеклист для production-готовности

- [ ] JMX включен с аутентификацией + SSL
- [ ] JMX Exporter настроен на каждом брокере
- [ ] kafka_exporter (или Burrow) собирает consumer lag
- [ ] Node Exporter собирает метрики ОС
- [ ] Prometheus скрейпит все exporter'ы
- [ ] Grafana dashboard покрывает все ключевые метрики
- [ ] Alert rules покрывают критические сценарии
- [ ] Алерты настроены с разумными `for`-периодами (избегаем flapping)
- [ ] Кардинальность метрик под контролем
- [ ] Liveness + readiness probes настроены (Kubernetes)
- [ ] Есть playbook для Top-5 инцидентов
- [ ] Data retention Prometheus достаточен для тренд-анализа (>2 недель)

---

## 8. Частые ошибки

### ❌ Мониторить «всё подряд»

**Результат:** 5000 временных рядов, Prometheus в OOM, алерты никто не читает.

**Правильно:** Начать с 10-15 ключевых метрик, добавлять по мере необходимости.

### ❌ Алертить на абсолютный consumer lag без учёта тренда

**Результат:** Постоянный false-positive alert на высоконагруженных топиках.

**Правильно:** Алерт: `lag > threshold AND rate(lag) > 0`.

### ❌ Не мониторить JVM heap и GC

**Результат:** Full GC приводит к тому, что брокер выпадает из ISR. Данные теряются. А вы даже не знаете почему.

### ❌ Использовать только Prometheus без OS-метрик

**Результат:** Prometheus показывает «всё хорошо», а диск уже на 95%.

### ❌ Забыть про JMX-безопасность в production

**Результат:** JMX без аутентификации = злоумышленник может выполнить `MBeanServerConnection.invoke()` и положить кластер.

---

## 9. Сводная таблица критических метрик

| # | Метрика | Норма | Severity | `for` |
|---|---------|-------|----------|-------|
| 1 | `ActiveControllerCount` | `1` | critical | 1m |
| 2 | `OfflineReplicaCount` | `0` | critical | 1m |
| 3 | `UnderMinIsrPartitionCount` | `0` | critical | 2m |
| 4 | `UnderReplicatedPartitions` | `0` | warning→critical | 5m/15m |
| 5 | Consumer lag (rate-based) | `rate ≤ 0` | warning→critical | 10m/30m |
| 6 | `RequestHandlerAvgIdlePercent` | `> 0.3` | warning | 10m |
| 7 | `NetworkProcessorAvgIdlePercent` | `> 0.3` | warning | 10m |
| 8 | Disk usage | `< 80%` | warning→critical | 5m |
| 9 | JVM Heap usage | `< 85%` | warning | 5m |
| 10 | `IsrShrinksPerSec` (не плановый) | `0` | warning | 0m |

---

## Заключение

Мониторинг Kafka — это не роскошь, а обязательное условие эксплуатации production-кластера. Начните с минимального набора метрик (10-15), настройте алертинг на критические сигналы (ActiveController, under-replicated, consumer lag) и только потом расширяйте покрытие.

Лучший стек на сегодня: **JMX Exporter → Prometheus → Grafana + AlertManager + Burrow**. Для Confluent-пользователей Control Center закрывает часть задач, но интеграция с общей системой мониторинга (Prometheus) всё равно необходима для единой панели observability.

---

## Источники

1. Apache Kafka Official Documentation — Monitoring (v4.1, 4.2): https://kafka.apache.org/documentation/#monitoring
2. Confluent Documentation — Monitoring Kafka with JMX: https://docs.confluent.io/platform/current/kafka/monitoring.html
3. Confluent jmx-monitoring-stacks (GitHub): https://github.com/confluentinc/jmx-monitoring-stacks
4. Prometheus JMX Exporter: https://github.com/prometheus/jmx_exporter
5. kafka_exporter (danielqsj): https://github.com/danielqsj/kafka_exporter
6. LinkedIn Burrow: https://github.com/linkedin/Burrow
7. Redpanda — Kafka Monitoring Guide: https://www.redpanda.com/guides/kafka-performance-kafka-monitoring
8. Confluent Documentation — Broker and Controller Metrics: https://docs.confluent.io/platform/current/kafka/broker-metrics.html
9. DataDog — Monitoring Kafka Metrics: https://www.datadoghq.com/blog/monitoring-kafka-performance-metrics/
10. Grafana Cloud — Kafka Integration: https://grafana.com/docs/grafana-cloud/monitor-infrastructure/integrations/integration-reference/integration-kafka/
11. Effective Strategies for Monitoring and Alerting on Kafka Health (devops.aibit.im): https://devops.aibit.im/article/monitoring-alerting-kafka-health
12. Kafka Monitoring: Tools & Best Practices (AutoMQ Wiki): https://github.com/AutoMQ/automq/wiki/Kafka-Monitoring:-Tools-&-Best-Practices
