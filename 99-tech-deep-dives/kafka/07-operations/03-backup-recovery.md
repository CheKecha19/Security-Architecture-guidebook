# 07.03 — Резервное копирование и восстановление Apache Kafka: стратегии, инструменты и Disaster Recovery

> **Bottom line:** Репликация (replication factor) — это НЕ бэкап. Это защита от отказа одного брокера, но не от логической ошибки, случайного удаления топика или выхода из строя всего дата-центра. Полноценная стратегия резервного копирования и восстановления Kafka требует трёх уровней защиты: бэкап данных (data backup), бэкап метаданных (metadata/configuration backup) и георепликация (cross-region DR через MirrorMaker). Без каждого из этих уровней вы рискуете необратимой потерей данных при инциденте любого масштаба — от ошибки разработчика до физического разрушения ЦОДа. Эта статья даёт практический фреймворк: что бэкапить, как восстанавливать, как планировать RTO/RPO и как действовать при сбоях компонентов кластера.

---

## 1. Почему репликация — это не бэкап

Самое распространённое заблуждение в эксплуатации Kafka: «у нас replication factor = 3, значит данные защищены». Это опасная полуправда.

**Что защищает репликация (replication factor):**

- Отказ одного или двух брокеров (при RF=3 и min.insync.replicas=2)
- Потерю диска на одном брокере
- Плановое обслуживание (rolling restart) без даунтайма

**Что репликация НЕ защищает:**

- **Случайное удаление топика** — `kafka-topics.sh --delete` распространяется на все реплики
- **Баг в коде продюсера** — «мусорные» сообщения мгновенно реплицируются на все копии
- **Ошибку сериализации схемы** — повреждённые данные расходятся по всем брокерам
- **Tombstone-записи в Kafka Streams** — маркеры удаления реплицируются синхронно
- **Полный отказ дата-центра** — пожар, наводнение, отключение электричества
- **Логическую ошибку администратора** — например, изменение retention.ms на 1 час для топика с критичными данными
- **Компрометацию кластера** — злоумышленник с доступом к брокерам может повредить все реплики

**Фундаментальное отличие:** репликация синхронна с живой системой. Любая проблема в primary-партиции моментально распространяется на реплики. Настоящий бэкап должен быть **иммутабельным** (неизменяемым после создания) и **изолированным** от работающего кластера.

### Аналогия

Представьте документ на вашем компьютере:
- **Репликация** — это как сохранить копию файла в соседнюю папку. Если вы случайно удалите оригинал или запишете в него мусор, копия тоже пострадает.
- **Бэкап** — это как отправить снапшот файла на отдельный жёсткий диск и убрать его в сейф. Даже если компьютер сгорит, снапшот останется.

---

## 2. Три уровня защиты данных Kafka

Полноценная стратегия резервного копирования строится на трёх независимых уровнях:

```
┌─────────────────────────────────────────────────────────────┐
│ УРОВЕНЬ 3: Cross-Region Disaster Recovery (geo-replication) │
│ MirrorMaker 2, Cluster Linking                              │
│ Защита от: потеря дата-центра, региональная катастрофа      │
├─────────────────────────────────────────────────────────────┤
│ УРОВЕНЬ 2: Metadata & Configuration Backup                  │
│ Топик-конфиги, ACL, квоты, Schema Registry, KRaft/ZK данные │
│ Защита от: потеря конфигурации, невозможность восстановления │
├─────────────────────────────────────────────────────────────┤
│ УРОВЕНЬ 1: Data Backup (Point-in-Time Recovery)             │
│ Бэкап сообщений + consumer group offsets в S3/GCS/Blob      │
│ Защита от: логические ошибки, случайное удаление, баги      │
└─────────────────────────────────────────────────────────────┘
```

Каждый уровень решает свой класс проблем. Пропуск любого уровня оставляет брешь в защите.

---

## 3. Data Backup: стратегии резервного копирования данных

### 3.1 Что именно нужно бэкапить

**Данные (Kafka logs):**
- Сегменты логов для каждой партиции (.log, .index, .timeindex, .snapshot)
- Это основной объём — сообщения, которые хранятся в топиках

**Состояние consumer groups (offsets):**
- Внутренний топик `__consumer_offsets` — критичен для восстановления
- Без него consumer groups после восстановления не знают, с какого смещения продолжать чтение

**Почему offsets критичны:**

Представьте: вы восстановили все сообщения из бэкапа, но не сохранили offset-ы consumer groups. Теперь у consumer-ов два плохих варианта:

- `--reset-to-earliest` → перечитать всё → дубликаты во всех downstream-системах
- `--reset-to-latest` → пропустить всё до текущего момента → потеря данных между точкой бэкапа и «сейчас»
- Угадать смещение → полагаться на удачу (неприемлемо для продакшена)

**Пример правильного восстановления:** в 14:29 consumer group `payment-processor` прочитала offset 492 000 топика `payments.initiated`. При восстановлении на 14:29 consumer возобновляет чтение ровно с offset 492 000 — ни одного пропущенного или продублированного сообщения.

### 3.2 Стратегии бэкапа данных

#### Стратегия А: Файловые снапшоты (Filesystem Snapshots)

**Как работает:** снапшот файловой системы на уровне брокера (LVM, ZFS, cloud-volume snapshots).

**Плюсы:**
- Быстрое создание (мгновенный снапшот)
- Полный образ данных брокера
- Минимальная нагрузка на Kafka

**Минусы:**
- Требует координации: все брокеры должны быть «заморожены» для консистентности
- Нельзя сделать инкрементальный бэкап — каждый снапшот полный
- Привязан к конкретной файловой системе / облачному провайдеру
- Сложность выбора точки восстановления (нельзя восстановить «на 14:29», только «последний снапшот»)

**Когда использовать:** небольшие кластеры, где допустимо создание ежечасных/ежедневных полных снапшотов.

#### Стратегия Б: Потоковый бэкап в Object Storage (предпочтительная)

**Как работает:** специализированный инструмент (Kafka Connect S3 Sink, OSO kafka-backup, кастомный consumer) непрерывно читает сообщения из топиков и записывает их в объектное хранилище (S3, GCS, Azure Blob) с сохранением offset-ов consumer groups.

**Плюсы:**
- Point-in-Time Recovery: восстановление на любой момент времени
- Инкрементальный: пишутся только новые сообщения
- Независимость от файловой системы брокеров
- Дешёвое хранение (S3 Standard ≈ $0.023/GB/мес vs EBS ≈ $0.08/GB/мес)
- Возможность сжатия (zstd, lz4, gzip)

**Минусы:**
- Дополнительная нагрузка на кластер (consumer читает все партиции)
- Требует настройки и мониторинга (лаг бэкап-consumer-а = ваш фактический RPO)
- Зависимость от внешнего сервиса (S3/GCS)

**Когда использовать:** продакшен-кластеры любого размера, когда нужно point-in-time восстановление.

#### Стратегия В: Кластер-тень (Shadow Cluster)

**Как работает:** второй Kafka-кластер, куда MirrorMaker 2 или Cluster Linking непрерывно реплицирует данные.

**Плюсы:**
- Данные всегда в «родном» Kafka-формате
- Быстрое переключение consumer-ов при отказе основного кластера

**Минусы:**
- **Удвоение стоимости инфраструктуры** (второй кластер с compute + storage)
- **Реплицирует проблему, а не только данные** (баг продюсера → мусор в обоих кластерах)
- Не даёт point-in-time recovery — только «состояние на сейчас»
- Не защищает от логических ошибок

**Когда использовать:** как дополнительный уровень для geo-DR, но НЕ как замену бэкапу.

### 3.3 Инструменты для бэкапа данных

| Инструмент | Тип | Хранилище | Offset Recovery | Сжатие | Лицензия |
|-----------|------|-----------|-----------------|--------|----------|
| **OSO kafka-backup** | CLI (Rust) | S3, GCS, Azure Blob, Local FS | ✅ Да (__consumer_offsets snapshot) | zstd | Open-source (Apache 2.0) |
| **Kafka Connect S3 Sink** | Connect Connector | AWS S3 | ❌ Нет (только сообщения) | gzip, snappy, lz4, zstd | Confluent Community License |
| **Kafka Connect GCS Sink** | Connect Connector | Google Cloud Storage | ❌ Нет | gzip, snappy | Confluent Community License |
| **MirrorMaker 2** | Connect-based | Другой Kafka-кластер | ✅ Через MirrorCheckpointConnector | Настраивается | Apache 2.0 |
| **Confluent Cluster Linking** | Встроенный | Другой Kafka-кластер | ✅ Да (offset sync) | Настраивается | Confluent Enterprise |
| **Custom script (Burry и др.)** | Скрипт | S3 / File | Частично | Настраивается | Разная |

#### Пример конфигурации Kafka Connect S3 Sink для бэкапа

```json
{
  "name": "s3-backup-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "4",
    "topics.regex": "orders.*|payments.*",
    "s3.bucket.name": "kafka-backups-prod",
    "s3.region": "eu-west-1",
    "s3.part.size": "5242880",
    "flush.size": "10000",
    "rotate.interval.ms": "60000",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
    "partition.duration.ms": "3600000",
    "locale": "en-US",
    "timezone": "UTC",
    "schema.compatibility": "NONE",
    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "key.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "s3-backup-dlq"
  }
}
```

**Важные параметры:**
- `flush.size` — размер пачки перед записью в S3 (баланс latency vs cost — каждый PUT в S3 стоит денег)
- `rotate.interval.ms` — максимальное время до ротации файла (определяет RPO)
- `partitioner.class=TimeBasedPartitioner` — структура партиций по времени для быстрого point-in-time поиска

#### Пример конфигурации OSO kafka-backup

```yaml
# backup.yaml
mode: backup
backup_id: "prod-daily-001"

source:
  bootstrap_servers: ["kafka-broker-1:9092", "kafka-broker-2:9092", "kafka-broker-3:9092"]

topics:
  include: ["orders-*", "payments-*", "events-*"]
  exclude: ["*-internal", "*-debug"]

storage:
  backend: s3
  bucket: kafka-backups-prod
  region: eu-west-1
  prefix: prod/

backup:
  compression: zstd
  segment_max_bytes: 134217728  # 128 MB
```

**Структура хранения бэкапа:**

```
s3://kafka-backups-prod/prod/prod-daily-001/
├── manifest.json                 # Метаданные бэкапа
├── state/
│   └── offsets.db                # Снапшот consumer group offsets
└── topics/
    └── payments.captured/
        ├── partition=0/
        │   ├── segment-0001.zst  # 128 MB сегмент (сжат zstd)
        │   └── segment-0002.zst
        └── partition=1/
            └── segment-0001.zst
```

Дата-партицированная структура позволяет точечно находить данные за любой временной интервал.

### 3.4 Point-in-Time Recovery: практический сценарий

**Инцидент:** пятница, 14:47. Обнаружено, что с 14:30 баг в коде продюсера пишет «мусорные» данные в топик `payments.captured`. Повреждённые данные начали распространяться в data warehouse, fraud detection, рекомендательную систему.

**Процесс восстановления:**

```bash
# 1. Останавливаем продюсера (14:48)

# 2. Определяем точку восстановления (14:49)
# Баг начался в 14:30 → восстанавливаем на 14:29

# 3. Конфигурация восстановления
```

```yaml
# restore.yaml
mode: restore
backup_id: "prod-daily-001"

target:
  bootstrap_servers: ["kafka-broker-1:9092", "kafka-broker-2:9092"]

storage:
  backend: s3
  bucket: kafka-backups-prod
  region: eu-west-1
  prefix: prod/

restore:
  time_window_end: 1732886940000  # 2025-11-28 14:29:00 UTC в epoch millis

  consumer_groups:
    - payment-processor
    - fraud-detector
    - warehouse-ingest
```

```bash
# 4. Запускаем восстановление (14:50)
kafka-backup restore --config restore.yaml
```

**Что происходит при восстановлении:**

1. Инструмент читает дата-партицированные сегменты из S3
2. Фильтрует все сообщения после 14:29
3. Записывает «чистые» сообщения обратно в production-кластер
4. Находит offset consumer group на 14:29 (offset 492 000)
5. Сбрасывает consumer group `payment-processor` на offset 492 000

**Результат (14:55):**
- Сообщения с 14:00 до 14:29 восстановлены ✅
- Consumer group offsets сброшены корректно ✅
- Downstream-системы возобновляют работу ✅

**Потеряно:** 18 минут данных (14:29–14:47) — но это повреждённые данные, которые в любом случае нельзя использовать.

**Альтернатива без бэкапа:** восстановление из бэкапа часовой/дневной давности, либо полная перестройка данных из upstream-систем (дни работы).

---

## 4. Metadata & Configuration Backup

### 4.1 Что входит в metadata backup

**Конфигурация топиков:**
- Названия, количество партиций, replication factor
- Параметры: `retention.ms`, `retention.bytes`, `cleanup.policy`, `compression.type`, `min.insync.replicas`
- Конфигурация compaction (для compacted topics)

**ACL (Access Control Lists):**
- Список правил доступа: какой principal к каким ресурсам
- Критично для безопасности после восстановления

**Квоты (Quotas):**
- Продюсер/consumer throughput limits
- Request rate limits

**Schema Registry:**
- Все схемы и их версии (Avro, Protobuf, JSON Schema)
- Настройки совместимости (BACKWARD, FORWARD, FULL)

**Данные ZooKeeper (для ZK-based кластеров):**
- `/brokers/ids` — зарегистрированные брокеры
- `/brokers/topics` — метаданные топиков
- `/config` — конфигурации
- `/admin` — административные операции
- `/controller` — текущий контроллер

**Данные KRaft (для KRaft-based кластеров):**
- Топик `__cluster_metadata` — вся метадата кластера
- Файлы metadata.log.dir на контроллерах
- Snapshot-ы контроллеров

### 4.2 Инструменты для бэкапа конфигурации

#### Бэкап конфигурации топиков и ACL

```bash
# Ручной бэкап — дамп всех топиков и их конфигураций
kafka-topics.sh --bootstrap-server localhost:9092 --describe > topics-backup-$(date +%Y%m%d).txt

# Бэкап ACL
kafka-acls.sh --bootstrap-server localhost:9092 --list > acls-backup-$(date +%Y%m%d).txt

# Полный дамп конфигурации через kafka-configs.sh
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --all --entity-type topics > configs-backup-$(date +%Y%m%d).txt
```

#### Автоматизация через JulieOps (GitOps для Kafka)

```yaml
# topology.yaml — описание желаемого состояния кластера
context: "backup-context"
projects:
  - name: "prod-project"
    topics:
      - name: "orders.created"
        config:
          replication.factor: "3"
          min.insync.replicas: "2"
          retention.ms: "604800000"  # 7 дней
          cleanup.policy: "delete"
      - name: "payments.captured"
        config:
          replication.factor: "3"
          min.insync.replicas: "2"
          retention.ms: "2592000000"  # 30 дней
          cleanup.policy: "delete"
    acls:
      - principal: "User:orders-service"
        host: "*"
        operation: "Write"
        resource: "Topic:orders.created"
```

**Преимущество JulieOps:** конфигурация кластера хранится в Git → всегда доступна для восстановления, версионируется, ревьюится.

#### Бэкап ZooKeeper

```bash
# Снапшот данных ZooKeeper
zkCli.sh -server zk1:2181 <<EOF
get /brokers/ids
get /brokers/topics
get /config/topics
get /admin
quit
EOF

# Или полный бэкап директории данных ZooKeeper
tar -czf zk-backup-$(date +%Y%m%d-%H%M).tar.gz /var/lib/zookeeper/data/
```

**Важно:** при восстановлении ZooKeeper нельзя просто скопировать data-директорию на работающий ensemble. Необходимо:
1. Остановить все ZK-ноды
2. Восстановить данные на каждой ноде
3. Запустить ensemble

Для Kafka 4.0+ (KRaft-only) ZooKeeper-бэкап больше не актуален — см. следующий раздел.

#### Бэкап KRaft metadata

```bash
# Проверка состояния метаданных KRaft
kafka-metadata-quorum.sh --bootstrap-server broker1:9092 describe --status

# Дамп содержимого __cluster_metadata в файл (для отладки)
kafka-metadata-shell.sh --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000-0000000000.checkpoint
```

**Ключевые аспекты KRaft backup:**

- KRaft хранит метаданные в виде Raft-лога (log segments + snapshots) в `metadata.log.dir`
- Snapshot-ы создаются автоматически controller-ами при достижении определённого размера лога
- Для восстановления достаточно snapshot-а + лог-сегментов после него
- **Бэкап KRaft = бэкап `metadata.log.dir` на всех controller-ах**
- Файл `meta.properties` содержит `cluster.id` и `node.id` — критичен

```bash
# Бэкап KRaft metadata
tar -czf kraft-metadata-backup-$(date +%Y%m%d-%H%M).tar.gz /var/lib/kafka/data/
```

#### Бэкап Schema Registry

```bash
# Через REST API — экспорт всех схем
curl -s "http://schema-registry:8081/subjects" | jq -r '.[]' | while read subject; do
  curl -s "http://schema-registry:8081/subjects/$subject/versions" | \
    jq -r '.[]' | while read version; do
    mkdir -p "schemas/${subject}"
    curl -s "http://schema-registry:8081/subjects/${subject}/versions/${version}/schema" \
      > "schemas/${subject}/v${version}.json"
  done
done
```

### 4.3 Чеклист: что должно быть в metadata backup

| Компонент | Что бэкапить | Метод | Частота |
|-----------|-------------|-------|---------|
| Topic configs | Названия, партиции, retention, cleanup.policy | `kafka-configs.sh --describe` / JulieOps | При каждом изменении + daily |
| ACL | Все правила доступа | `kafka-acls.sh --list` / JulieOps | При каждом изменении + daily |
| Quotas | Throughput/request limits | `kafka-configs.sh --describe --entity-type users` | Daily |
| ZooKeeper (если используется) | Data directory | `tar` + rsync | Daily |
| KRaft | `metadata.log.dir` на всех контроллерах | `tar` + rsync | Daily |
| Schema Registry | Все субъекты и версии | REST API / бэкап БД | Daily |
| Cluster ID | `meta.properties` | Включён в бэкап KRaft/ZK | При форматировании |
| Сертификаты TLS | Keystore, truststore, CA | Внешний secrets management | При обновлении |

---

## 5. Cross-Region Disaster Recovery через MirrorMaker

### 5.1 Архитектурные паттерны георепликации

**Паттерн 1: Active-Passive (основной + резервный)**

```
┌──────────────────────┐         ┌──────────────────────┐
│    PRIMARY (DC1)     │         │   SECONDARY (DC2)    │
│                      │  MM2    │                      │
│  Producers active ───┼────────►│  Data replicated     │
│  Consumers active    │  ──────►│  Consumers standby   │
│                      │         │                      │
└──────────────────────┘         └──────────────────────┘
```

- Все продюсеры и consumer-ы работают с PRIMARY
- MirrorMaker 2 непрерывно реплицирует данные в SECONDARY
- При отказе PRIMARY — consumer-ы переключаются на SECONDARY
- **Плюсы:** простая операционная модель, нет конфликтов данных
- **Минусы:** SECONDARY простаивает (оплата простаивающей инфраструктуры)

**Паттерн 2: Active-Active (оба кластера активны)**

```
┌──────────────────────┐         ┌──────────────────────┐
│    CLUSTER A (DC1)   │         │   CLUSTER B (DC2)    │
│                      │  MM2    │                      │
│  Producers local     │◄───────►│  Producers local     │
│  Consumers local     │  ──────►│  Consumers local     │
│  Topics: orders,     │         │  Topics: orders,     │
│  a.payments, ...     │         │  b.payments, ...     │
└──────────────────────┘         └──────────────────────┘
```

- Оба кластера обслуживают production-трафик
- MirrorMaker реплицирует в обе стороны
- Каждый регион обрабатывает своих клиентов (lower latency)
- **Плюсы:** эффективное использование ресурсов, низкая latency
- **Минусы:** сложность (конфликты, репликационные петли, двойная запись)

### 5.2 MirrorMaker 2: ключевые компоненты

MM2 состоит из трёх типов коннекторов Kafka Connect:

| Коннектор | Назначение |
|-----------|-----------|
| **MirrorSourceConnector** | Реплицирует топики из source-кластера в target |
| **MirrorCheckpointConnector** | Транслирует consumer group offsets между кластерами |
| **MirrorHeartbeatConnector** | Мониторит latency репликации |

### 5.3 Конфигурация MirrorMaker 2 для DR

```properties
# mm2-dr.properties — конфигурация MirrorMaker 2 для Disaster Recovery
clusters = primary, secondary

# === PRIMARY CLUSTER (DC1 - Москва) ===
primary.bootstrap.servers = kafka-dc1-1:9092,kafka-dc1-2:9092,kafka-dc1-3:9092
primary.security.protocol = SASL_SSL
primary.sasl.mechanism = PLAIN
primary.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="mm2-user" password="${MM2_PASSWORD}";

# === SECONDARY CLUSTER (DC2 - Новосибирск) ===
secondary.bootstrap.servers = kafka-dc2-1:9092,kafka-dc2-2:9092,kafka-dc2-3:9092
secondary.security.protocol = SASL_SSL
secondary.sasl.mechanism = PLAIN
secondary.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="mm2-user" password="${MM2_PASSWORD}";

# === Направление репликации: primary → secondary ===
primary->secondary.enabled = true
primary->secondary.topics = orders\..*, payments\..*, events\..*

# Replication policy — префиксы для предотвращения петель
primary->secondary.replication.policy.class = \
  org.apache.kafka.connect.mirror.DefaultReplicationPolicy

# Синхронизация конфигурации топиков
primary->secondary.sync.topic.configs.enabled = true
primary->secondary.sync.topic.acls.enabled = true

# Производительность
primary->secondary.tasks.max = 64
primary->secondary.producer.override.compression.type = lz4
primary->secondary.producer.override.batch.size = 131072  # 128KB
primary->secondary.producer.override.linger.ms = 10
primary->secondary.consumer.override.fetch.min.bytes = 1048576  # 1MB

# Consumer group offset sync
primary->secondary.emit.checkpoints.enabled = true
primary->secondary.emit.heartbeats.enabled = true
primary->secondary.checkpoints.topic.replication.factor = 3

# Отказоустойчивость
primary->secondary.producer.override.retries = 3
primary->secondary.producer.override.acks = 1
```

### 5.4 Выбор replication policy: DefaultReplicationPolicy vs IdentityReplicationPolicy

**DefaultReplicationPolicy (рекомендуется для DR):**

```bash
# Топики на SECONDARY получают префикс source-кластера:
# primary.orders.created
# primary.payments.captured
```

**Плюсы:**
- Предотвращает репликационные петли
- Чёткое происхождение данных (понятно, откуда пришло)
- Возможность будущего перехода на active-active

**Минусы:**
- Consumer-ы после failover должны читать `primary.*`-топики
- Требуется перенастройка consumer-ов при переключении

**IdentityReplicationPolicy:**

```bash
# Топики на SECONDARY сохраняют оригинальные имена
# orders.created
# payments.captured
```

**Плюсы:**
- Потребители не нуждаются в перенастройке
- Проще failover

**Минусы:**
- Бесконечные репликационные петли, если включить оба направления
- Не подходит для active-active

> **Рекомендация:** выбирайте политику ДО запуска в продакшен. Смена политики позже требует переименования топиков и перенастройки всех consumer-ов.

### 5.5 Failover и Failback с MirrorMaker 2

#### Failover (primary → secondary)

```bash
# 1. Останавливаем продюсеров на PRIMARY (если возможно)
# 2. Убеждаемся, что MM2 drain-ит последние сообщения в SECONDARY
# 3. Переключаем consumer-ы на SECONDARY
#    - DefaultReplicationPolicy: читать primary.orders.created вместо orders.created
#    - IdentityReplicationPolicy: без изменений

# 4. Проверяем offset-ы consumer groups
kafka-consumer-groups.sh --bootstrap-server secondary:9092 \
  --group payment-processor --describe

# 5. Запускаем продюсеров на SECONDARY (если active-active)
```

#### Failback (secondary → primary)

Это самая сложная часть — часто недооценивается в DR-планах.

```bash
# 1. Валидируем здоровье PRIMARY:
kafka-metadata-quorum.sh --bootstrap-server primary:9092 describe --status
kafka-topics.sh --bootstrap-server primary:9092 --describe --under-replicated-partitions

# 2. Разворачиваем ВРЕМЕННЫЙ обратный MM2: secondary → primary
#    (новый коннектор, реплицирует только данные, записанные за время outage)

# 3. Ждём, пока обратная репликация догонит (lag → 0)

# 4. Сначала переключаем CONSUMER-ы на PRIMARY
# 5. Затем переключаем PRODUCER-ы на PRIMARY
#    (последовательность важна: consumer-ы должны быть готовы раньше продюсеров)

# 6. После стабилизации — останавливаем обратный MM2
# 7. Возобновляем прямой MM2 (primary → secondary)
```

**Ключевое правило failback:** consumer-ы переезжают ПЕРВЫМИ, продюсеры — ВТОРЫМИ. Это предотвращает потерю сообщений, произведённых до того, как consumer-ы готовы их читать.

### 5.6 Мониторинг MM2 для DR

```xml
<!-- JMX метрики MirrorMaker 2 для Prometheus JMX Exporter -->
<rule pattern="kafka.connect.mirror:type=MirrorSourceConnector,target=(\w+),topic=(\w+),partition=(\d+)">
  <metric name="mm2_replication_lag" type="gauge">
    <labels>
      <label name="target" value="$1"/>
      <label name="topic" value="$2"/>
      <label name="partition" value="$3"/>
    </labels>
  </metric>
</rule>
```

**Критические метрики MM2 для DR:**

| Метрика | Значение | Alert |
|---------|----------|-------|
| `mm2_replication_lag` (P99) | < 5 сек нормально | > 30 сек → WARNING |
| `mm2_replication_lag` (P99) | — | > 120 сек → CRITICAL |
| Source vs Target message rate diff | < 10% нормально | > 10% → WARNING |
| MirrorCheckpointConnector lag | < 10 сек нормально | > 60 сек → WARNING |
| MM2 consumer lag на source | < 1000 сообщений | > 10000 → WARNING |

---

## 6. RTO/RPO планирование

### 6.1 Определения

**RTO (Recovery Time Objective)** — максимальное допустимое время восстановления сервиса. Отвечает на вопрос: «сколько минут/часов мы можем быть недоступны?»

**RPO (Recovery Point Objective)** — максимальная допустимая потеря данных. Отвечает на вопрос: «сколько минут/часов данных мы готовы потерять?»

### 6.2 Классификация рабочих нагрузок (Workload Tiers)

Не все топики одинаково критичны. Классифицируйте их по трём категориям:

**Tier 1 — Mission-Critical (обязательный DR):**
- Платёжные транзакции (`payments.*`)
- Данные аутентификации и авторизации
- Fraud detection в реальном времени
- Критичные бизнес-события (создание заказа, отгрузка)

| Параметр | Значение |
|----------|----------|
| RPO | ≤ 30 секунд |
| RTO | < 5 минут |
| Репликация | Active-Active или Active-Passive с синхронной MM2 |
| Бэкап | Непрерывный потоковый бэкап + 5-минутные снапшоты |
| Тестирование | Ежеквартальные game days с реальным failover |

**Tier 2 — Business-Critical (рекомендованный DR):**
- Аналитические пайплайны клиентов (`analytics.*`)
- Рекомендательные системы
- Отчёты и дашборды
- Не-critical уведомления

| Параметр | Значение |
|----------|----------|
| RPO | ≤ 5 минут |
| RTO | < 30 минут |
| Репликация | Active-Passive с асинхронной MM2 |
| Бэкап | Потоковый бэкап + ежечасные снапшоты |
| Тестирование | Ежемесячные проверки replication lag |

**Tier 3 — Non-Critical (опциональный DR):**
- Development/test окружения
- Внутренние дашборды
- Отладочные/экспериментальные топики
- Логи низкого приоритета

| Параметр | Значение |
|----------|----------|
| RPO | ≤ 24 часа |
| RTO | < 4 часа (в рабочее время) |
| Репликация | Опционально |
| Бэкап | Ежедневные снапшоты |
| Тестирование | При необходимости |

### 6.3 Взаимосвязь RTO/RPO с архитектурой бэкапа

```
RPO (потеря данных)    Метод бэкапа            Стоимость
─────────────────────  ──────────────────────  ────────
< 1 минуты             Синхронная MM2          $$$$
< 5 минут              Асинхронная MM2         $$$
< 1 час                S3 потоковый бэкап      $$
< 24 часа              Ежедневные снапшоты     $
```

**Важный нюанс:** фактический RPO определяется НЕ только replication lag MM2. Настройки продюсера также влияют:

```properties
# Эти параметры определяют, сколько данных продюсер буферизует в памяти
# При отказе кластера буферизованные данные ТЕРЯЮТСЯ!

# delivery.timeout.ms = 120000 (по умолчанию 2 минуты!)
# Это значит: если продюсер не может доставить сообщение за 2 минуты,
# он выбросит исключение и сообщение потеряно.

# Если ваш процесс обнаружения инцидента + принятия решения > 2 минут,
# фактический RPO ХУЖЕ, чем replication lag!
```

**Настройка для критичных нагрузок:**

```properties
# Увеличиваем delivery.timeout.ms = 600000 (10 минут)
# — даём время на обнаружение и failover

# Увеличиваем buffer.memory = 134217728 (128MB, по умолчанию 32MB)
# — больше места для буферизации при недоступности кластера

# enable.idempotence = true
# — предотвращает дубликаты при ретраях после failover

# max.in.flight.requests.per.connection = 5
# — баланс ordering vs throughput
```

### 6.4 Экономика DR: когда овчинка стоит выделки

**Прямые издержки простоя:**

- Потерянная выручка (для финтеха: миллионы рублей в час)
- SLA-штрафы клиентам
- Регуляторные штрафы (ЦБ РФ, PCI-DSS)
- Сверхурочные команды реагирования

**Скрытые издержки:**

- Отток клиентов к конкурентам
- Ущерб репутации бренда
- Потеря рыночной доли (восстановление занимает годы)
- Рост стоимости привлечения новых клиентов

**Формула принятия решения:**

```
Стоимость DR-инфраструктуры < (Вероятность инцидента × Стоимость простоя) → Внедрять
```

Для regulated industries (банки, платежи, телеком) DR — не опция, а обязательное требование.

---

## 7. Восстановление из сбоя: пошаговые процедуры

### 7.1 Сценарий 1: Отказ одного брокера (не DR)

**Симптомы:** брокер не отвечает, under-replicated partitions, ISR уменьшился.

**Реакция Kafka (автоматическая):**
1. Контроллер обнаруживает отказ через `zookeeper.session.timeout.ms` (ZK) или heartbeat (KRaft)
2. Назначает новых лидеров для партиций, где отказавший брокер был лидером
3. ISR удаляет отказавший брокер → репликация продолжается без него
4. Клиенты получают обновлённые метаданные и переподключаются к новым лидерам

**Действия администратора:**

```bash
# 1. Проверить состояние кластера
kafka-topics.sh --bootstrap-server broker1:9092 --describe --under-replicated-partitions

# 2. Проверить ISR для проблемных партиций
kafka-topics.sh --bootstrap-server broker1:9092 --describe --topic critical-topic

# 3. Если брокер вернулся — он автоматически догонит репликацию
# 4. Если брокер умер навсегда — заменить оборудование и добавить новый брокер

# 5. Добавление нового брокера (замена)
# Новый брокер с уникальным broker.id, указать bootstrap.servers существующего кластера
# Kafka автоматически начнёт переносить партиции на новый брокер

# 6. Контролировать прогресс переноса
kafka-reassign-partitions.sh --bootstrap-server broker1:9092 \
  --verify --reassignment-json-file reassignment.json
```

**RTO:** ~0 минут (автоматический failover при RF≥3)  
**RPO:** ~0 сообщений (потеряны только uncommitted сообщения продюсеров)  
**Усилия:** минимальные (только замена железа)

### 7.2 Сценарий 2: Отказ контроллера (ZK-based кластер)

**Симптомы:** `ActiveControllerCount = 0` или метрика скачет.

**В ZooKeeper-кластере:**
1. ZooKeeper обнаруживает отказ контроллера по session timeout
2. Выбирается новый контроллер (ZK election)
3. Новый контроллер перечитывает метаданные из ZooKeeper
4. Восстанавливает управление кластером

**Действия администратора:**

```bash
# Диагностика — какой брокер сейчас контроллер
echo "get /controller" | kafka-zookeeper-shell.sh zk1:2181

# Если контроллеров нет — проверить ZooKeeper
echo "stat" | nc zk1 2181

# Принудительный перевыбор (в экстренном случае)
# Перезапустить брокер, который был предыдущим контроллером
```

**RTO:** обычно 5–15 секунд (ZK election)  
**RPO:** 0 (метаданные в ZooKeeper)

### 7.3 Сценарий 3: Отказ контроллера (KRaft-based кластер)

**Симптомы:** нет активного контроллера, метрика `ActiveControllerCount = 0`.

**В KRaft-кластере:**
1. KRaft quorum обнаруживает, что лидер (активный контроллер) недоступен
2. Запускается Raft election среди оставшихся контроллеров
3. Новый лидер начинает обслуживать запросы
4. Брокеры получают уведомление о новом контроллере

**Действия администратора:**

```bash
# Проверить состояние KRaft quorum
kafka-metadata-quorum.sh --bootstrap-server broker1:9092 describe --status

# Вывод покажет:
# - Кто текущий лидер (LeaderId)
# - Кто входит в quorum (VoterIds)
# - Статус репликации метаданных

# Если quorum потерян:
# 1. Определить, сколько контроллеров живо
# 2. Если живо большинство (> N/2) — quorum восстановится автоматически
# 3. Если потеряно большинство — катастрофический сценарий (см. 7.5)
```

**RTO:** < 1 секунда (Raft election в KRaft значительно быстрее ZK)  
**RPO:** 0 (метаданные реплицированы в Raft логе)

### 7.4 Сценарий 4: Потеря большинства ZooKeeper нод

**Симптомы:** Kafka-брокеры не могут подключиться к ZooKeeper, кластер недоступен.

**Восстановление:**

```bash
# 1. Восстановить ZooKeeper из последнего бэкапа
systemctl stop zookeeper  # на всех нодах

# 2. Очистить data dir на всех ZK нодах
rm -rf /var/lib/zookeeper/data/version-2/*

# 3. Восстановить данные из бэкапа на ОДНОЙ ноде
tar -xzf zk-backup-20251128.tar.gz -C /

# 4. Запустить эту ноду
systemctl start zookeeper

# 5. Проверить
echo "stat" | nc zk1 2181

# 6. Запустить остальные ZK ноды — они синхронизируются с первой

# 7. Перезапустить Kafka-брокеры
systemctl restart kafka  # на всех брокерах

# 8. Проверить кластер
kafka-topics.sh --bootstrap-server broker1:9092 --list
```

**RTO:** 15–30 минут (если бэкап ZK свежий и процедура отлажена)  
**RPO:** до последнего бэкапа ZK (обычно daily → до 24 часов метаданных)

### 7.5 Сценарий 5: Потеря большинства KRaft контроллеров

**Симптомы:** quorum потерян, кластер в read-only режиме или недоступен.

Это самый опасный сценарий — без большинства контроллеров KRaft НЕ МОЖЕТ выбрать лидера и обслуживать изменения.

**Восстановление:**

```bash
# 1. Оценить ущерб: сколько контроллеров живо?
# Если живо >= большинства → автоматическое восстановление
# Если нет → ручное восстановление

# 2. Определить контроллер с самыми свежими метаданными
# Проверить последний snapshot в metadata.log.dir на каждом контроллере:
ls -la /var/lib/kafka/data/__cluster_metadata-0/*.checkpoint

# 3. Использовать kafka-metadata-shell.sh для проверки целостности
kafka-metadata-shell.sh --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000500-0000000001.checkpoint

# 4. Создать новый quorum из уцелевших/восстановленных контроллеров
# Используя данные с контроллера с наиболее свежим snapshot-ом

# 5. Форматировать новый quorum:
CLUSTER_ID="<cluster-id из meta.properties>"
# На каждом восстанавливаемом контроллере:
kafka-storage.sh format --cluster-id $CLUSTER_ID --standalone --config controller.properties

# 6. Для остальных контроллеров (добавляются как новые):
kafka-storage.sh format --cluster-id $CLUSTER_ID --config controller.properties --no-initial-controllers

# 7. Запустить контроллеры и брокеры
# 8. Восстановить топик-конфигурации из бэкапа metadata
# 9. Верифицировать целостность данных
```

**Профилактика:** бэкапируйте `metadata.log.dir` на всех контроллерах! Без свежего бэкапа восстановление после потери большинства контроллеров может быть невозможно.

### 7.6 Сценарий 6: Полная потеря кластера (Disaster)

**Симптомы:** дата-центр недоступен, все брокеры и контроллеры offline.

**Восстановление (план А — DR-кластер):**

```bash
# 1. Активировать DR-кластер (SECONDARY)
# 2. Проверить replication lag на момент отказа
kafka-consumer-groups.sh --bootstrap-server secondary:9092 --describe

# 3. Если MM2 был настроен с MirrorCheckpointConnector:
#    - Consumer offsets уже доступны на SECONDARY
#    - Consumer-ы могут продолжить с последней синхронизированной позиции

# 4. Перенаправить DNS / load balancer на SECONDARY
# 5. Запустить consumer-ы на SECONDARY
# 6. Запустить продюсеры на SECONDARY

# 7. Верифицировать работу всех Tier 1 приложений
# 8. Затем Tier 2, Tier 3
```

**Восстановление (план Б — из S3 бэкапа):**

```bash
# Если DR-кластера нет, а есть потоковый бэкап в S3:

# 1. Развернуть новый Kafka-кластер в другом регионе
#    (желательно заранее подготовить infrastructure-as-code)

# 2. Восстановить конфигурацию топиков из metadata backup
kafka-topics.sh --bootstrap-server new-cluster:9092 --create \
  --topic payments.captured \
  --partitions 12 --replication-factor 3 \
  --config retention.ms=2592000000

# 3. Восстановить ACL
# (применить сохранённый дамп ACL)

# 4. Восстановить данные из S3
kafka-backup restore --config restore-to-new-cluster.yaml

# 5. Восстановить Schema Registry
# (из бэкапа схем)

# 6. Запустить consumer-ы с восстановленными offset-ами

# RTO: 30–60 минут (при готовом IaC и процедуре)
# RPO: до последнего flush в S3 (обычно 1–5 минут)
```

### 7.7 Сводная таблица сценариев и RTO/RPO

| Сценарий | RTO | RPO | Автоматическое восстановление | Нужен бэкап |
|----------|-----|-----|------------------------------|------------|
| Отказ 1 брокера (RF=3) | ~0 мин | ~0 | ✅ Да | Нет |
| Отказ 2 брокеров одной партиции (RF=3) | ~0 мин | ~0 | ❌ Нужен minISR=2 | Нет |
| Отказ ZK контроллера | 5–15 сек | 0 | ✅ Да | Нет |
| Отказ KRaft контроллера | < 1 сек | 0 | ✅ Да | Нет |
| Потеря majority ZK нод | 15–30 мин | До 24ч метаданных | ❌ Нет | Да (ZK backup) |
| Потеря majority KRaft контроллеров | 30–60 мин | До 24ч метаданных | ❌ Нет | Да (KRaft backup) |
| Потеря дата-центра (DR-кластер) | 5–15 мин | < 5 сек (MM2) | ❌ Нет | Да (MM2) |
| Потеря дата-центра (S3 restore) | 30–60 мин | 1–5 мин | ❌ Нет | Да (S3 backup) |
| Логическая ошибка (баг продюсера) | 10–30 мин | Потери испорченных данных | ❌ Нет | Да (PITR backup) |

---

## 8. Tiered Storage как дополнительный уровень защиты

Начиная с Kafka 3.6 (KIP-405), доступен **Tiered Storage** — механизм автоматического переноса «холодных» лог-сегментов в удалённое хранилище (S3, HDFS).

### 8.1 Как Tiered Storage помогает в DR

```
Локальное хранилище брокера (горячие данные)
    │
    │ сегменты старше local.retention.ms
    ▼
Удалённое хранилище — S3/GCS/HDFS (холодные данные)
```

**Преимущества для DR:**

- Данные автоматически дублируются в S3 → защита от потери локальных дисков
- Восстановление в 115 раз быстрее при пересоздании брокера (данные в S3 уже есть)
- Стоимость хранения в 3–9 раз ниже, чем 3× репликация на EBS
- Снижает RTO для полного восстановления кластера

### 8.2 Конфигурация Tiered Storage для целей бэкапа

```properties
# server.properties
remote.log.storage.system.enable = true
remote.log.metadata.manager.listener.name = PLAINTEXT

# Класс RemoteStorageManager (нужна имплементация — например, Confluent S3 plugin)
remote.log.storage.manager.class.name = io.confluent.tiered.storage.ConfluentTieredStorage

# === Конфигурация топика ===
# Создаём топик с tiered storage
kafka-topics.sh --create --topic orders.created \
  --bootstrap-server broker1:9092 \
  --config remote.storage.enable=true \
  --config local.retention.ms=3600000 \    # 1 час на локальном диске
  --config retention.ms=2592000000          # 30 дней всего (S3 + локально)
```

**Ограничения Tiered Storage (на 2025–2026):**
- Не поддерживает compacted topics
- Только append-only логи
- Требует внешнего плагина RemoteStorageManager (нет встроенной имплементации в Apache Kafka)
- Не заменяет полноценный PITR-бэкап (данные в S3 — это «текущее состояние», а не иммутабельный снапшот)

---

## 9. Тестирование DR: от плана к реальности

DR-план, который не тестировался — это не план, а гипотеза. Исследование Conduktor (2025) показывает: большинство организаций инвестировали в репликацию, но только 20% регулярно тестируют полный failover.

### 9.1 Частота и типы тестирования

| Тип тестирования | Частота | Что проверяет |
|-----------------|---------|---------------|
| **Проверка replication lag** | Ежедневно (автоматически) | MM2 connector здоров, lag < порога |
| **Табличные учения (Tabletop exercise)** | Ежемесячно | DR runbook актуален, роли понятны |
| **Chaos тестирование staging** | Ежемесячно | Имитация отказа брокера/контроллера/сети |
| **Game Day — production failover** | Ежеквартально | Полный цикл: обнаружение → failover → failback |
| **Полномасштабное DR-тестирование** | Ежегодно | Имитация полной потери дата-центра |

### 9.2 Game Day: сценарий проведения

```bash
# 1. Оповещение: объявить о плановых учениях за 2 недели
# 2. Подготовка: сверить конфигурации primary ↔ secondary
# 3. «Взрыв»: симулировать отказ PRIMARY (остановить MM2, затем брокеры)
# 4. Обнаружение: засечь время от «взрыва» до первой страницы алерта
# 5. Failover: переключить consumer-ы на SECONDARY (засечь время)
# 6. Валидация: проверить все Wave 1 приложения на SECONDARY
# 7. Failback: вернуть трафик на PRIMARY (засечь время)
# 8. Post-mortem: разобрать, что пошло не так, обновить runbook
```

### 9.3 Метрики для Game Day

**Измеряйте фактические значения, сравнивайте с целевыми:**

| Метрика | Цель | Факт |
|---------|------|------|
| Время обнаружения (TTD) | < 1 мин | ____ |
| Время failover (TTF) | < 5 мин | ____ |
| Время failback (TTB) | < 15 мин | ____ |
| Потерянные сообщения | 0 | ____ |
| Дублированные сообщения | 0 (или в пределах at-least-once) | ____ |
| Успешно восстановленные consumer groups | 100% Wave 1 | ____ |

**Красный флаг:** если ваша цель RTO — 15 минут, а на Game Day получается 90 — у вас не разногласие, у вас разрыв между планом и реальностью.

### 9.4 Аудит конфигурационного дрейфа

Одна из главных причин провала DR — конфигурационный дрейф между primary и secondary:

```bash
# Автоматизированная проверка конфигурационного дрейфа
# Сравниваем retention.ms топиков между кластерами

for topic in $(kafka-topics.sh --bootstrap-server primary:9092 --list); do
  primary_retention=$(kafka-configs.sh --bootstrap-server primary:9092 \
    --describe --entity-type topics --entity-name $topic 2>/dev/null | grep retention.ms)
  secondary_retention=$(kafka-configs.sh --bootstrap-server secondary:9092 \
    --describe --entity-type topics --entity-name "primary.$topic" 2>/dev/null | grep retention.ms)

  if [ "$primary_retention" != "$secondary_retention" ]; then
    echo "DRIFT DETECTED: $topic — PRIMARY=$primary_retention SECONDARY=$secondary_retention"
  fi
done
```

---

## 10. Антипаттерны и типичные ошибки DR

| # | Антипаттерн | Почему опасно | Как исправить |
|---|------------|---------------|---------------|
| 1 | **«RF=3 = бэкап»** | Репликация не защищает от логических ошибок | Разделять: репликация для HA, бэкап для DR |
| 2 | **Только Shadow Cluster** | Стоимость ×2, нет PITR, реплицирует баги | Добавить S3-бэкап для point-in-time восстановления |
| 3 | **Бэкап без consumer offsets** | Consumer-ы не знают, откуда читать после восстановления | Всегда бэкапить `__consumer_offsets` или использовать инструменты с offset recovery |
| 4 | **Нетестированный runbook** | DR-план становится liabilities вместо assets | Game Days ежеквартально, версионировать runbook |
| 5 | **Конфигурационный дрейф** | SECONDARY не готов принять трафик | Автоматическая сверка конфигов между кластерами |
| 6 | **Ignore producer timeouts** | delivery.timeout.ms = 120s при detection window 5+ мин → молчаливая потеря данных | Настраивать таймауты продюсеров под реальный процесс failover |
| 7 | **Единственный MM2 connector** | Head-of-line blocking: высоконагруженный топик блокирует репликацию остальных | Разделять MM2 коннекторы по профилям нагрузки |
| 8 | **Failback без drain** | Не дождались нулевого лага → потеря сообщений при обратном переключении | Drain → verify lag=0 → switch |
| 9 | **Бэкап на том же оборудовании** | Пожар/наводнение уничтожает и primary, и бэкап | Физически разделять: разные DC, разные аккаунты облака |
| 10 | **Ручной failover на 30+ сервисов** | RTO масштабируется с количеством сервисов, а не с качеством автоматизации | Централизованный endpoint/прокси для переключения клиентов |

---

## 11. Чеклист готовности DR

### Data Backup

- [ ] Настроен непрерывный бэкап данных в S3/GCS/Azure Blob
- [ ] Бэкап включает consumer group offsets
- [ ] Бэкап мониторится (lag бэкап-consumer-а = фактический RPO)
- [ ] Point-in-Time Recovery протестирован на staging
- [ ] Определены retention-периоды для бэкапов (7/30/90/365 дней)
- [ ] Бэкапы зашифрованы (AES-256, KMS)

### Metadata Backup

- [ ] Настроен daily backup конфигураций топиков
- [ ] Настроен daily backup ACL
- [ ] Настроен daily backup Schema Registry
- [ ] Настроен daily backup ZooKeeper (если используется) ИЛИ KRaft metadata
- [ ] Конфигурация кластера хранится как код (JulieOps / Terraform / Ansible)
- [ ] Сертификаты TLS и ключи — в защищённом secrets management

### Cross-Region DR

- [ ] MirrorMaker 2 развёрнут и реплицирует все Tier 1 & 2 топики
- [ ] MirrorCheckpointConnector активен
- [ ] Replication lag мониторится с алертами
- [ ] Конфигурационный дрейф между кластерами отслеживается
- [ ] План failover задокументирован и проверен на Game Day
- [ ] План failback задокументирован и проверен на Game Day
- [ ] Второй кластер имеет достаточную ёмкость (1.5× нормальной нагрузки)

### RTO/RPO

- [ ] Определены Tier 1/2/3 для всех рабочих нагрузок
- [ ] Для каждого Tier установлены целевые RTO и RPO
- [ ] Фактические RTO/RPO измерены на Game Day
- [ ] Параметры продюсеров (delivery.timeout.ms, buffer.memory) согласованы с RPO
- [ ] Процесс принятия решения о failover документирован (кто, когда, как быстро)

### Процедуры

- [ ] Задокументирован процесс восстановления брокера
- [ ] Задокументирован процесс восстановления контроллера (ZK / KRaft)
- [ ] Задокументирован процесс восстановления ZooKeeper
- [ ] Задокументирован процесс восстановления KRaft quorum
- [ ] Задокументирован процесс восстановления из S3 бэкапа
- [ ] Все runbook-и версионируются и ревьюятся

### Тестирование

- [ ] Game Day проводится ежеквартально
- [ ] Тестируется полный цикл: обнаружение → failover → failback
- [ ] Измеряются фактические TTD, TTF, TTB
- [ ] Проверяется восстановление ВСЕХ Wave 1 приложений
- [ ] Runbook обновляется после каждого Game Day
- [ ] Chaos-тестирование на staging проводится ежемесячно

---

## 12. Ссылки по теме

- [06.03 — Масштабирование Kafka](../06-performance/03-scalability.md) — partitioning, geo-replication, tiered storage
- [07.01 — Мониторинг Kafka](01-monitoring.md) — метрики для DR, алертинг replication lag
- [07.02 — Логирование Kafka](02-logging.md) — аудит-логи, трейсинг для DR
- [07.05 — Автоматизация Kafka](05-automation.md) — IaC, GitOps, автоматизация failover
- [04.02 — Развёртывание Kafka](../04-software/02-implementation.md) — production-деплой, multi-broker cluster

## Источники

1. Confluent — «Apache Kafka Backup: Everything You Need To Know», confluent.io/learn/kafka-backup, 2025
2. Conduktor — «Kafka Disaster Recovery: The Complete Strategy Beyond Replication», conduktor.io/blog, 2025
3. OSO — «How to Back Up Your Kafka Cluster: A Guide to Point-in-Time Recovery», oso.sh/blog, December 2025
4. OSO — «Building Bulletproof Disaster Recovery for Apache Kafka: A Field-Tested Architecture», oso.sh/blog, May 2025
5. OSO — «How to Build a Well-Architected Kafka Backup Strategy: Six Pillars», oso.sh/blog, March 2026
6. BigData Boutique — «Kafka MirrorMaker 2: Deployment, Gotchas, and Disaster Recovery Failback Playbook», bigdataboutique.com/blog, February 2026
7. Apache Kafka Documentation — «Tiered Storage», kafka.apache.org/41/operations/tiered-storage, 2025
8. Apache Kafka Documentation — «KRaft Operations», kafka.apache.org/40/operations/kraft, 2025
9. Apache Kafka Documentation — «Geo-Replication (Cross-Cluster Data Mirroring)», kafka.apache.org/37/operations, 2025
10. Confluent — «Disaster Recovery for Multi-Datacenter Apache Kafka Deployments», confluent.io/blog, updated March 2025
11. Confluent — «Testing & Maintaining Apache Kafka DR and HA Readiness», confluent.io/blog, 2025
12. Software Patterns Lexicon — «Mastering Kafka Backup and Restore Mechanisms», softwarepatternslexicon.com, 2025
13. Red Hat Documentation — «Disaster Recovery using MirrorMaker 2», docs.redhat.com, Streams for Apache Kafka 3.1
14. WarpStream — «The Hitchhiker's Guide to Disaster Recovery and Multi-Region Kafka», warpstream.com/blog, June 2025
