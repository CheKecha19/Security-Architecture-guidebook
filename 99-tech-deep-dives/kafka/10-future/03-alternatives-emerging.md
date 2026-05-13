# 10.3 — Emerging конкуренты и потенциальные замены Kafka

> **Bottom Line:** Kafka не умирает и не будет заменён одним «Kafka-killer'ом». Но архитектурный ландшафт меняется фундаментально. Три категории emerging-решений претендуют на ниши Kafka: (1) Kafka-совместимые платформы с иной архитектурой (Redpanda, WarpStream, AutoMQ), (2) принципиально другие модели (Pulsar, NATS, Bufstream), и (3) «стриминговые базы данных» (RisingWave, Materialize), которые вообще убирают необходимость в отдельном брокере. Выбор зависит от того, какую именно работу Kafka выполняет у вас: event bus, queue, stream processor или data backbone.

---

## 1. Карта ландшафта: кто и как конкурирует с Kafka

### 1.1 Три категории emerging-решений

Вместо одной «альтернативы Kafka» рынок породил три разных класса решений:

```
┌─────────────────────────────────────────────────────────────┐
│              EMERGING KAFKA ALTERNATIVES                     │
├───────────────────┬─────────────────┬───────────────────────┤
│ Kafka-совместимые │ Другая модель   │ Стриминговые БД       │
│ (тот же протокол, │ (свой протокол, │ (убирают нужду в      │
│  иная архитектура)│  иная парадигма)│  отдельном брокере)   │
├───────────────────┼─────────────────┼───────────────────────┤
│ • Redpanda        │ • Pulsar        │ • RisingWave          │
│ • WarpStream      │ • NATS          │ • Materialize         │
│ • AutoMQ          │ • Bufstream     │ • ClickHouse          │
│ • Confluent Warp  │ • RabbitMQ      │   (Materialized Views)│
│ • Aiven Diskless  │   Streams       │ • ksqlDB (изнутри     │
│ • Amazon MSK      │ • StreamNative  │   экосистемы)         │
│                   │   Ursa          │                       │
└───────────────────┴─────────────────┴───────────────────────┘
```

Каждая категория решает разную проблему Kafka:

- **Kafka-совместимые** — «Kafka, но меньше operational pain»
- **Другая модель** — «Kafka не нужна, у нас другая парадигма»
- **Стриминговые БД** — «Kafka — лишнее звено, данные обрабатываются на месте»

### 1.2 Ключевой вопрос: какую именно работу Kafka выполняет у вас?

Прежде чем рассматривать альтернативы, честно ответьте:

| Роль Kafka | Примеры | Заменима? |
|------------|---------|------------|
| **Event Bus** (микросервисы обмениваются событиями) | OrderCreated → PaymentService | ✅ Да, NATS превосходит |
| **Message Queue** (обработка задач) | Job queue для видео-транскодинга | ✅ Да, Queues for Kafka / RabbitMQ |
| **Stream Processing back-end** (Kafka Streams / Flink) | Clickstream → real-time агрегации | ⚠️ Частично (RisingWave) |
| **Data Backbone** (единый pipeline для всех данных) | Все события компании → Kafka → N consumers | ❌ Пока незаменима |
| **Log / Audit Store** (долговременный лог) | Audit trail 90 дней | ✅ Tiered Storage / S3-based решения |
| **CDC Pipeline** (Change Data Capture) | PostgreSQL → Kafka → Data Lake | ❌ Kafka Connect незаменим |

**Вывод:** не существует одного «Kafka-killer'а». Есть решения, превосходящие Kafka в конкретных use-case, но проигрывающие в экосистемной зрелости.

---

## 2. Kafka-совместимые платформы: тот же протокол, другая архитектура

### 2.1 Redpanda — «Kafka без JVM, на C++»

**Redpanda** — полностью переписанный на C++ брокер, совместимый с Kafka API. Запущен в 2020, сейчас в v24.x.

**Ключевые архитектурные отличия от Apache Kafka:**

| Характеристика | Apache Kafka | Redpanda |
|---------------|-------------|----------|
| Язык | Java / Scala | C++ (Seastar framework) |
| JVM | Да (10+ GB heap) | Нет |
| Консенсус | KRaft (Raft) | Собственный Raft |
| Партиционирование | Log segments (.log/.index) | Собственный лог-формат |
| Конфигурация тюнинга | 150+ параметров | ~20 параметров |
| Запуск | 30+ секунд (JVM warmup) | ~3 секунды |
| Kafka Connect | ✅ Полная поддержка | ❌ Только через внешний кластер |
| Tiered Storage | ✅ KIP-405 (S3) | ✅ S3 tiering |
| Schema Registry | Внешний (Confluent) | Встроенный (Redpanda Schema Registry) |
| Rack Awareness | ✅ | ✅ |
| Transactions (EOS) | ✅ | ✅ |
| Лицензия | Apache 2.0 | BSL (ограничения для SaaS) |

**Когда Redpanda лучше Kafka:**

1. **Latency-critical workloads** — C++/Seastar даёт стабильно низкую latency без GC-пауз
2. **Edge / IoT / embedded** — отсутствие JVM позволяет запускать на устройствах с 512 MB RAM
3. **Operational simplicity** — 90% меньше конфигурационных параметров
4. **Быстрые перезапуски** — 3 секунды vs 30+ секунд

**Когда Redpanda НЕ подходит:**

1. **Kafka Connect pipelines** — Redpanda не поддерживает Kafka Connect нетивно (только через отдельный Apache Kafka кластер или Managed Connect)
2. **Большая экосистема** — Kafka Streams, ksqlDB, Debezium могут иметь edge-баги из-за отличий в поведении, несмотря на API-совместимость
3. **Enterprise compliance** — BSL-лицензия может быть проблемой для аудита opensource-соответствия
4. **Позиционирование смещается** — Redpanda переориентируется на Agentic AI use-cases, что может снизить фокус на core Kafka-замене

### 2.2 WarpStream — «Kafka без дисков»

**WarpStream** (приобретён Confluent в 2024) — реализация Kafka-совместимого протокола поверх S3, написанная на Go. Данные никогда не попадают на локальные диски.

**Архитектурная модель:**
```
Producer → Agent (stateless) → S3 (durable storage)
Consumer ← Agent (stateless) ← S3
           ↑
     Control Plane (metadata, SASL/TLS termination)
```

**Ключевые особенности:**

| Характеристика | Особенность |
|---------------|-------------|
| Хранение | Исключительно S3 |
| Агенты | Stateless, масштабируются мгновенно |
| Задержка | ~400–600ms P99 (стандартные топики) |
| Lightning Topics | S3 Express One Zone → ниже latency, но без ordering guarantees и transactions |
| Транзакции | ❌ Не поддерживаются |
| Compacted Topics | ❌ Не поддерживаются |
| Kafka Connect | Через внешний кластер |
| BYOC (Bring Your Own Cloud) | ✅ Можно деплоить в свой VPC |
| Zero Data Loss | ✅ Синхронная репликация через S3 |
| Ценообразование | По uncompressed data volume |

**Когда WarpStream лучше Kafka:**

1. **Cost-sensitive workloads** — S3 дешевле EBS в 3–5×
2. **Эластичное масштабирование** — агенты stateless, масштабируются за секунды
3. **Disaster recovery** — все данные в S3, нет нужды в MirrorMaker
4. **BYOC deployments** — полный контроль над VPC и данными при managed-сервисе
5. **Низкий DevOps overhead** — нет партиций, нет ребалансировки, нет выбора инстансов

**Когда WarpStream НЕ подходит:**

1. **Low-latency use cases** — 400-600ms неприемлемо для real-time bidding, fraud detection
2. **Transactions (EOS)** — не поддерживаются, что критично для финансов
3. **Compacted topics** — не поддерживаются, нет CDC/state-stores
4. **Kafka Connect на том же кластере** — нужен отдельный Apache Kafka или MSK
5. **Ценообразование по uncompressed data** — если вы сжимаете данные (что большинство и делает), платите за «воздух»

### 2.3 AutoMQ — «Kafka на S3 с сохранением полной совместимости»

**AutoMQ** — форк Apache Kafka на Java, который заменяет только storage layer: вместо локальных дисков используется S3 + pluggable WAL (Write-Ahead Log). Уникальность: полная совместимость с Kafka Connect и Streams из коробки.

**Архитектурная модель:**
```
Producer → AutoMQ Broker (stateless) → WAL (EBS/S3/NFS, configurable) → S3 (async batch)
Consumer ← AutoMQ Broker (stateless) ──────────────────────────────────────┘
```

**Ключевые особенности:**

| Характеристика | Особенность |
|---------------|-------------|
| Кодовая база | Форк Apache Kafka (Java) |
| Тест-совместимость | Проходит все 2000+ тестов Apache Kafka |
| Kafka Connect | ✅ Полная поддержка |
| Kafka Streams | ✅ Полная поддержка |
| Транзакции (EOS) | ✅ Полная поддержка |
| Compacted Topics | ✅ Полная поддержка |
| WAL backend | EBS (sub-10ms), S3 (~500ms), NFS |
| Tiered Storage | Автоматический (WAL → S3) |
| Strimzi Operator | ✅ Поддерживается |
| Лицензия | Apache 2.0 |

**Когда AutoMQ лучше Kafka:**

1. **Полная Kafka-совместимость при diskless экономике** — единственная платформа, сохраняющая 100% совместимость при S3-based архитектуре
2. **Переменные workload с latency-требованиями** — WAL-уровень позволяет выбрать latency vs cost для разных топиков: EBS WAL для low-latency, S3 WAL для cost-optimized
3. **Kafka Connect heavy deployments** — не надо поднимать отдельный кластер для Connect, как в Redpanda/WarpStream
4. **Enterprise миграция с Apache Kafka** — минимальный риск, тот же код, тот же API

**Когда AutoMQ НЕ подходит:**

1. **Экосистема пока молода** — меньше production-кейсов, чем у Kafka/Redpanda/Confluent
2. **WAL добавляет сложность** — хоть и pluggable, это дополнительный компонент
3. **Отсутствие managed-сервиса на всех облаках** — не такой mature, как Confluent Cloud / MSK

### 2.4 Aiven Diskless Kafka (KIP-1150)

**Aiven** реализует diskless-вариант Apache Kafka через community KIP-1150: брокеры остаются стандартными Kafka-брокерами, но путь репликации перенаправляется в S3, а не на локальные диски. Leaderless архитектура с PostgreSQL-координатором для batch-метаданных.

Преимущества: полная совместимость с Apache Kafka, снижение cross-AZ трафика (данные не реплицируются между AZ локально).

Недостатки: зрелость KIP-1150 (в процессе), зависимость от PostgreSQL для координации.

---

## 3. Принципиально другие модели: не Kafka-совместимые

### 3.1 Apache Pulsar — «Kafka с разделением compute и storage изначально»

**Apache Pulsar** — распределённая система обмена сообщениями, архитектурно отличающаяся разделением serving (брокеры) и storage (BookKeeper bookies).

**Архитектурное сравнение Kafka vs Pulsar:**

```
Kafka:                              Pulsar:
┌─────────────────────┐             ┌──────────────┐     ┌──────────────┐
│ Брокер = Compute +  │             │  Pulsar      │     │  BookKeeper  │
│ Storage (один узел) │             │  Broker      │────→│  Bookie      │
│                     │             │  (stateless) │     │  (storage)   │
│ Диск с данными      │             └──────────────┘     └──────────────┘
└─────────────────────┘
```

**Когда Pulsar подходит:**

| Ситуация | Почему Pulsar |
|----------|--------------|
| Multi-tenant streaming (100+ клиентов) | Pulsar изначально проектировался для multi-tenancy (Yahoo!) |
| Геораспределённая репликация | Geo-replication built-in, не через MirrorMaker |
| Сильные гарантии порядка при масштабировании | Key-shared subscription + individual ack |
| Tiered Storage из коробки | Отделение BookKeeper от брокера даёт естественный storage-tier |
| Queue + Stream в одной системе | Pulsar поддерживает queue (shared) и stream (exclusive/failover) подписки |

**Когда Pulsar НЕ подходит:**

| Ситуация | Почему НЕ Pulsar |
|----------|-----------------|
| Простые стриминговые сценарии | Операционная сложность ×2 (брокеры + bookies) |
| Маленькая команда | Администрирование Pulsar сложнее, чем Kafka и тем более Redpanda/WarpStream |
| Экосистема Kafka-инструментов | Хотя Pulsar поддерживает Kafka-on-Pulsar (KoP), это прослойка, не native |
| Kafka Connect / Streams | Не поддерживается или с большими ограничениями |

**Текущее состояние (2026):** Pulsar остаётся нишевым выбором для multi-tenant и геораспределённых сценариев. StreamNative (главный коммерческий вендор) представил **StreamNative Ursa** — lakehouse-native streaming на Iceberg/Delta, что может дать второе дыхание экосистеме.

### 3.2 NATS — «Event Bus, а не Event Store»

**NATS** — легковесная система обмена сообщениями, написанная на Go. Принципиальное отличие: NATS позиционируется как **event bus** (маршрутизация сообщений), а не как **event store** (долговременное хранение и replay, как Kafka).

**Сравнение по ключевым возможностям:**

| Возможность | Apache Kafka | NATS (JetStream) |
|------------|-------------|-------------------|
| Модель | Event log + storage | Pub/Sub + ephemeral messages |
| Хранение | Постоянное (retention по времени/размеру) | JetStream добавляет persistence, но модель другая |
| Replay | ✅ Перечитать партицию с offset'а | ❌ Сообщения уходят в consumer сразу |
| Retention | Дни/месяцы/годы | Ограничено (JetStream storage) |
| Масштабирование | Партиции | Горизонтальное через clustering |
| Количество клиентов (consumers) | Ограничено партициями | Неограничено (fan-out) |
| Kafka Connect | ✅ 200+ коннекторов | ❌ Не поддерживается |
| Kafka Streams | ✅ | ❌ |

**Когда NATS лучше Kafka:**

1. **Microservice mesh** — быстрая, лёгкая маршрутизация запрос-ответ (request-reply из коробки)
2. **IoT / Edge** — ультранизкий footprint (запускается на ARM, 32MB RAM)
3. **Real-time notifications** — fan-out тысячам клиентов без партиций
4. **Cloud-native K8s** — де-факто стандарт для межсервисной коммуникации в K8s (включая встроенную поддержку в некоторых платформах)
5. **Простота** — запустить кластер NATS = 1 команда

**Когда NATS НЕ подходит:**

1. **Event sourcing / audit trail** — нет долговременного хранения и replay
2. **Data pipelines** — нет Kafka Connect, schema registry, Debezium
3. **Stream processing** — нет аналога Kafka Streams / ksqlDB
4. **CDC (Change Data Capture)** — нет Debezium-интеграции
5. **Крупные enterprise deployments** — экосистема значительно меньше

### 3.3 RabbitMQ Streams — «очередь с логом»

**RabbitMQ** добавил поддержку **Streams** (начиная с 3.9) — append-only лог, похожий на Kafka-топик, но внутри RabbitMQ.

**Когда это может заменить Kafka:**
1. Если вы уже используете RabbitMQ для очередей и хотите добавить streaming без второго брокера
2. Простые сценарии: логгирование, сбор метрик с небольшой задержкой

**Ограничения:**
- Производительность значительно ниже Kafka (не рассчитан на миллионы msg/s)
- Нет Kafka Connect / Kafka Streams
- Нет tiered storage
- Меньшая экосистема

### 3.4 Bufstream — «Iceberg-first streaming»

**Bufstream** — S3-native streaming с PostgreSQL или Spanner как metadata backend. Нацелен на Data Lakehouse сценарии, где Iceberg-таблицы — primary data store.

Пока niche — для команд, у которых streaming — вспомогательный инструмент для наполнения Data Lakehouse.

---

## 4. Стриминговые базы данных: а нужна ли Kafka вообще?

### 4.1 Новая категория: Streaming Database

Для ряда use-cases отдельный message broker может быть избыточен. Если данные нужны только для аналитики/дэшбордов, стриминговая БД объединяет ingestion, processing и serving в одной системе:

```
Традиционная архитектура:             Streaming Database:
Source → Kafka → Flink → DB → Query    Source → RisingWave (ingest+process+serve) → Query
                                          или
                                        Source → Materialize (ingest+process+serve) → Query
```

### 4.2 RisingWave

**RisingWave** — стриминговая база данных, PostgreSQL-совместимая по SQL. Принимает данные из Kafka/Pulsar/Kinesis/NATS напрямую, обрабатывает и материализует инкрементальные вьюхи.

**Ключевые возможности:**
- PostgreSQL wire protocol (любой psql-клиент работает)
- Materialized Views с автоматическим обновлением
- Встроенная поддержка оконных функций, JOIN'ов, агрегаций
- State store с checkpointing'ом в S3
- Horizontal scaling

**Сценарий замены Kafka:** если Kafka использовался как pipeline для real-time аналитики, RisingWave может заменить связку Kafka + Flink + ClickHouse одной системой.

### 4.3 Materialize

**Materialize** — аналогичная концепция, но с фокусом на инкрементальную материализацию сложных SQL-запросов. Совместимость с PostgreSQL.

**Отличие от RisingWave:** более сильный фокус на корректность (strict serializability), меньше — на максимальный throughput.

### 4.4 ClickHouse Materialized Views + Kafka Engine

**ClickHouse** — колоночная аналитическая БД — имеет нативную интеграцию с Kafka через `Kafka` engine и `MaterializedView`:

```sql
CREATE TABLE kafka_events
ENGINE = Kafka
SETTINGS kafka_broker_list = 'broker:9092',
         kafka_topic_list = 'events',
         kafka_group_name = 'clickhouse',
         kafka_format = 'JSONEachRow';

CREATE MATERIALIZED VIEW mv_events TO target_table AS
SELECT ... FROM kafka_events;
```

Это не полная замена Kafka, но для сценариев «Kafka как шина для аналитики» ClickHouse может читать из Kafka нативно, без отдельного consumer-сервиса.

### 4.5 Когда стриминговая БД заменяет Kafka

| Сценарий | Можно заменить на | Почему |
|----------|-------------------|--------|
| Real-time дашборды | RisingWave / ClickHouse MV | Данные сразу в БД, минуя Kafka |
| Непрерывная аналитика (агрегации по окнам) | RisingWave / Materialize | SQL-based streaming без отдельного Flink |
| Kafka для наполнения data warehouse | Tableflow (Iceberg) | Автоматическая материализация топиков |
| CDC → Kafka → аналитика | RisingWave + CDC source | Direct ingestion, без промежуточного Kafka |

### 4.6 Когда стриминговая БД НЕ заменяет Kafka

| Сценарий | Почему нет |
|----------|-----------|
| Микросервисная коммуникация | Стриминговая БД — не шина для межсервисной передачи |
| Multiple independent consumers | Kafka даёт изоляцию consumer groups |
| Долговременное хранение + replay | Kafka retention — стандарт для event sourcing |
| Экосистема (Connect, Schema Registry, Streams) | Нет аналогов для Kafka Connect за пределами Kafka |
| Политика безопасности данных | Kafka ACL + TLS дают гранулярный контроль на уровне топика |

---

## 5. Сравнительная таблица: все альтернативы в одном месте

| Платформа | Kafka-совместимость | Модель хранения | Латентность (P99) | Транзакции | Connect | Лицензия | Лучший use-case |
|-----------|-------------------|----------------|-------------------|------------|---------|----------|----------------|
| **Apache Kafka** | 100% | Локальные диски | <10ms | ✅ | ✅ | Apache 2.0 | Data backbone, CDC, event sourcing |
| **Redpanda** | 95%+ | Локальные диски + S3 | <5ms | ✅ | ⚠️ внешний | BSL | Low-latency, edge, operational simplicity |
| **WarpStream** | 80% | S3 only | 400-600ms | ❌ | ⚠️ внешний | Source available | Cost-optimized, elastic, BYOC |
| **AutoMQ** | 100% | S3 + WAL | 10ms (EBS WAL) | ✅ | ✅ | Apache 2.0 | 100% совместимость + diskless |
| **Confluent Cloud** | 100% | Managed | <10ms | ✅ | ✅ (managed) | SaaS | Enterprise managed Kafka |
| **Amazon MSK** | 100% | EBS | <10ms | ✅ | ✅ (MSK Connect) | Managed | AWS-native Kafka |
| **Pulsar** | ~80% (через KoP) | BookKeeper + Tiered | <10ms | ✅ | ❌ | Apache 2.0 | Multi-tenant, geo-replication |
| **NATS** | 0% (свой протокол) | JetStream | <1ms | ❌ | ❌ | Apache 2.0 | Microservice mesh, IoT |
| **RisingWave** | Consumer only | S3 state store | Зависит от источника | N/A | ❌ | Apache 2.0 | Streaming analytics без Kafka |
| **Materialize** | Consumer only | Локальный + S3 | Зависит от источника | N/A | ❌ | BSL | SQL-based real-time materialized views |

---

## 6. Фреймворк принятия решения: когда уходить с Kafka?

### 6.1 Чёткие сигналы к миграции

| Сигнал | Куда смотреть |
|--------|--------------|
| Счета за EBS + cross-AZ трафик > $5K/мес | WarpStream, AutoMQ, Tiered Storage |
| Operational overhead (патчи, ребалансировка, тюнинг) съедает >40% времени команды | Confluent Cloud, Redpanda |
| Нужна быстрая эластичность (spiky workloads) | WarpStream, AutoMQ |
| Нужен ultra-low latency (<5ms P99) | Redpanda |
| Нужна полная Kafka-совместимость + diskless | AutoMQ, Aiven Diskless |
| Вам нужен event bus, а не event store | NATS |
| Вы используете Kafka только как шину для аналитики | RisingWave, ClickHouse MV |
| Multi-tenant SaaS с сотнями изолированных клиентов | Pulsar |

### 6.2 Когда ОСТАТЬСЯ на Kafka

| Ситуация | Почему |
|----------|--------|
| Kafka Connect экосистема критична (Debezium, 200+ коннекторов) | Никто не имеет полного паритета по Connect |
| Kafka Streams / ksqlDB на production | Только Kafka-native |
| Schema Registry с Avro/Protobuf/JSON Schema | Стандарт экосистемы |
| Strict exactly-once семантика | Не все альтернативы поддерживают |
| Уже построен enterprise governance вокруг Kafka | Стоимость миграции > экономия |
| Большая команда с экспертизой в Kafka | Неоправданный риск смены технологии |

### 6.3 Гибридный подход (самый распространённый)

2026 год показывает, что большинство компаний не мигрируют ПОЛНОСТЬЮ с Kafka, а диверсифицируют:

```
Kafka (core backbone)
    ├── Redpanda (edge clusters, low-latency path)
    ├── WarpStream (cost-optimized archival topics)
    ├── NATS (microservice request-reply)
    └── RisingWave (real-time analytics, минуя Kafka для read path)
```

---

## 7. Прогноз: каким будет мир streaming в 2028–2030

### 7.1 Три сценария развития

**Сценарий A: Kafka everywhere (консервативный)**
- Apache Kafka остаётся стандартом, KRaft стабилизируется, Tiered Storage становится must-have
- Альтернативы занимают ниши (Redpanda в edge, NATS в microservice mesh, WarpStream в cost-optimized)
- Confluent продолжает доминировать в коммерческом сегменте

**Сценарий B: Diskless takeover (вероятный)**
- Object storage становится достаточно быстрым (S3 Express One Zone, Azure Blob Premium)
- Diskless архитектуры (WarpStream, AutoMQ) становятся дефолтом для новых проектов
- Kafka-протокол сохраняется, но отмирание локальных дисков меняет операционную модель

**Сценарий C: Streaming DBs revolution (ambitious)**
- Стриминговые БД (RisingWave, Materialize) созревают до замены Kafka для 60%+ use-cases
- Kafka остаётся только для data backbone и CDC
- Схема: Source → Streaming DB (ingest + process + serve) без промежуточного брокера

### 7.2 Наиболее вероятный исход: «Kafka Protocol becomes TCP of data streaming»

Как TCP стал универсальным транспортным протоколом (при множестве реализаций — Linux kernel TCP, QUIC, mTCP), так **Kafka-протокол становится стандартом** data streaming:

- Apache Kafka — эталонная реализация
- Redpanda — high-performance реализация (C++)
- WarpStream — cloud-native реализация (S3-based)
- AutoMQ — fully compatible fork realization (diskless)
- Confluent Cloud — enterprise managed реализация

Разные реализации для разных архитектурных потребностей — но общий протокол.

---

## 8. Источники

1. **Kafka Alternatives Compared (2026): AutoMQ vs Confluent Cloud vs WarpStream vs Redpanda vs MSK** — [AutoMQ Blog](https://www.automq.com/blog/kafka-alternatives-compared-2026)
2. **Best Kafka Alternatives in 2026: Compared by Use Case** — [Estuary](https://estuary.dev/blog/kafka-alternatives/)
3. **Kafka Alternatives: Redpanda, AutoMQ & WarpStream** — [CloudRPS](https://cloudrps.com/blog/kafka-alternatives-redpanda-automq-warpstream/)
4. **Event-Driven Architecture in 2026: Kafka vs. Pulsar vs. Redpanda** — [Simplified Learning](https://simplifiedlearningblog.com/kafka-vs-pulsar-vs-redpanda/)
5. **Redpanda vs NATS vs Apache Kafka — Event Streaming 2026** — [PkgPulse](https://www.pkgpulse.com/blog/redpanda-vs-nats-vs-apache-kafka-event-streaming-2026)
6. **The Streaming Database Landscape in 2026: A Complete Guide** — [RisingWave](https://risingwave.com/blog/streaming-database-landscape-2026-complete-guide/)
7. **Apache Kafka alternatives: comparison guide** — [Redpanda](https://www.redpanda.com/guides/kafka-alternatives)
8. **Streaming Data Processing Tools Compared** — [Tinybird](https://www.tinybird.co/blog/kafka-alternatives-for-streaming-data-processing-8-tools-compared)
9. **The Rise of Diskless Kafka: Rethinking Brokers, Storage, and the Kafka Protocol** — [Kai Waehner](https://www.kai-waehner.de/blog/2025/08/11/the-rise-of-diskless-kafka-rethinking-brokers-storage-and-the-kafka-protocol/)
10. **Real-Time Streaming 2026 — From Kafka to AI Context Engines** — [Simon Cullen](https://insights.simon-cullen.com/real-time-streaming-2026/)
11. **Apache Kafka vs Pulsar vs Redpanda (Medium)** — [TheDataForge](https://thedataforge.medium.com/kafka-vs-pulsar-vs-redpanda-57d527789db7)
12. **Real-time analytics platforms: a practical comparison for 2026** — [ClickHouse](https://clickhouse.com/resources/engineering/real-time-analytics-platforms-a-practical-comparison)

---

*Связанные статьи:*
- [01-roadmap.md](01-roadmap.md) — Roadmap Apache Kafka
- [02-trends.md](02-trends.md) — Тренды и индустриальная динамика
- [03-alternatives-emerging.md](03-alternatives-emerging.md) — эта статья
- [../03-tech/03-vs-alternatives.md](../03-tech/03-vs-alternatives.md) — Kafka vs альтернативы
- [../09-real-world/03-anti-patterns.md](../09-real-world/03-anti-patterns.md) — Анти-паттерны использования Kafka
- [../05-development-api/03-patterns.md](../05-development-api/03-patterns.md) — Паттерны разработки с Kafka
