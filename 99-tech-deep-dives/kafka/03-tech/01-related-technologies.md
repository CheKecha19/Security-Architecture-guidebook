# Смежные технологии Apache Kafka: что стоит рядом в архитектуре

**Трек:** 03 — Tech  
**Статья:** 01-related-technologies.md  
**Статус:** done  
**Слов:** ~4600  
**Источников:** 12

---

## TL;DR

Kafka не живёт в вакууме — вокруг неё выстраивается целая экосистема технологий, решающих задачи, которые Kafka сама не берёт на себя. **Собственные расширения** (Kafka Connect, Schema Registry, Kafka Streams, ksqlDB) закрывают интеграцию, управление схемами и стримовую обработку. **Внешние технологии** делятся на четыре слоя: источники данных (PostgreSQL, MySQL, MongoDB через Debezium CDC), потребители (Elasticsearch, S3, Hadoop), обработчики (Flink, Spark, Kafka Streams) и инфраструктурные слои (Kubernetes, Prometheus, Redis). Выбор технологий-компаньонов определяет производительность, надёжность и сложность всей системы.

---

## 1. Внутренняя экосистема Kafka: четыре технологии, расширяющие ядро

Прежде чем смотреть на внешние технологии, разберём, что Kafka предлагает «из коробки» для решения смежных задач. Эти компоненты — не альтернативы, а расширения, тесно связанные с ядром.

### 1.1 Kafka Connect — интеграционный слой

**Назначение:** Стандартизированный фреймворк для подключения Kafka к внешним системам без написания кода.

Kafka Connect работает по модели коннекторов двух типов:
- **Source Connectors** — забирают данные из внешней системы и публикуют в Kafka (producer)
- **Sink Connectors** — читают данные из Kafka и отправляют во внешнюю систему (consumer)

**С какими технологиями работает (ключевые коннекторы):**

| Категория | Source (→Kafka) | Sink (Kafka→) |
|-----------|-----------------|---------------|
| **Реляционные БД** | JDBC Source, Debezium (PostgreSQL, MySQL, Oracle, SQL Server) | JDBC Sink |
| **NoSQL** | MongoDB Source, Cassandra Source | MongoDB Sink, Cassandra Sink |
| **Поисковые системы** | — | Elasticsearch Sink, OpenSearch Sink |
| **Облачные хранилища** | — | S3 Sink, GCS Sink, Azure Blob Sink |
| **Озёра данных** | — | HDFS Sink, Iceberg Sink, Delta Lake Sink |
| **Очереди сообщений** | JMS Source, MQTT Source | — |
| **Файловые системы** | FileStream Source, SpoolDir Source | FileStream Sink |

**Ключевое преимущество:** Kafka Connect превращает Kafka в универсальный хаб данных (data hub) — любые данные попадают в Kafka, обрабатываются там и уходят в любые потребители. По состоянию на 2025 год экосистема насчитывает 200+ коннекторов [1].

**Аналогия:** Kafka Connect — это адаптер питания в аэропорту. Разные страны (системы) имеют разные розетки (форматы данных, протоколы). Адаптер (Connect) стандартизирует подключение — Kafka работает как универсальный USB-порт, в который воткнуто всё остальное.

### 1.2 Schema Registry — управление форматами данных

**Назначение:** Централизованный реестр схем данных, обеспечивающий совместимость при эволюции форматов.

Kafka сама по себе не валидирует структуру сообщений — она хранит сырые байты. Schema Registry решает проблему «откуда потребитель знает, как десериализовать сообщение?»

**Поддерживаемые форматы сериализации [2][3]:**

| Формат | Тип схемы | Размер | Человекочитаемость | Эволюция схем | Применение с Kafka |
|--------|-----------|--------|-------------------|---------------|-------------------|
| **Avro** | Бинарный | Компактный | Нет (JSON-схема отдельно) | Forward/Backward/Full | Стандарт де-факто в Hadoop/Kafka-экосистеме |
| **Protobuf** | Бинарный | Очень компактный | Нет (.proto файлы) | Forward/Backward | gRPC-экосистема, высокопроизводительные микросервисы |
| **JSON Schema** | Текстовый | Большой | Да | Forward/Backward | Простота отладки, веб-API, когда читаемость важнее размера |

**Стратегии эволюции схем:**
- `BACKWARD` — новый consumer может читать старые данные (добавление полей с default)
- `FORWARD` — старый consumer может читать новые данные (удаление полей)
- `FULL` — совместимость в обе стороны
- `NONE` — без проверок (быстро, но опасно)

**Как работает:** При отправке сообщения producer отправляет schema ID (не schema body). Consumer получает schema ID и ищет схему в Registry. Схема кэшируется на клиенте. Сообщение без схемы весит меньше, а эволюция контролируется централизованно.

**Аналогия:** Schema Registry — это словарь, лежащий между писателем (producer) и читателем (consumer). Писатель пишет слово номер 42, читатель смотрит в словарь: «а, это означает "order_created" с полями A, B, C».

### 1.3 Kafka Streams — библиотека потоковой обработки

**Назначение:** Java-библиотека для построения приложений потоковой обработки, работающая как обычное Java-приложение (без отдельного кластера).

Kafka Streams реализует:
- **Stateless операции:** filter, map, flatMap, branch
- **Stateful операции:** join (KStream-KStream, KStream-KTable, KTable-KTable), aggregation (count, sum, reduce), windowing (hopping, tumbling, session)
- **Внутреннее хранилище состояний:** RocksDB (на диске, с backup в Kafka changelog-топики)
- **Interactive Queries:** REST-доступ к локальным state store

**Ключевое отличие от других обработчиков:** Kafka Streams — это библиотека, не кластер. Она запускается как часть вашего приложения, масштабируется горизонтально добавлением инстансов, автоматически перебалансирует состояние при сбоях. Это устраняет необходимость в отдельном вычислительном кластере (в отличие от Spark/Flink).

**Связка с Kafka:** Kafka Streams использует Kafka как:
- Источник данных (source topics)
- Приёмник результатов (sink topics)
- Механизм fault-tolerance (changelog topics для восстановления состояния)
- Коммуникационный канал между инстансами (repartition topics)

Подробнее — в разделе о Kafka Streams в статье [Ecosystem](../03-tech/02-ecosystem.md).

### 1.4 ksqlDB — SQL-интерфейс над Kafka

**Назначение:** Потоковая база данных, позволяющая писать потоковые запросы на SQL-подобном языке поверх Kafka-топиков.

ksqlDB строит поверх Kafka Streams уровень, где операции — это SQL-запросы, выполняемые в реальном времени. Это радикально снижает порог входа для аналитиков и data engineer'ов, которым не нужно писать на Java.

**Ключевые возможности:**
- **Persistent Queries** — бесконечно выполняющиеся запросы, результаты которых пишутся в выходной топик
- **Pull Queries** — точечные запросы к материализованным представлениям (аналог key-value lookup)
- **Push Queries** — подписка на поток изменений (аналог реактивного стрима)

**Пример ksqlDB запроса:**
```sql
-- Создаём стрим из Kafka-топика
CREATE STREAM orders (
  order_id VARCHAR KEY,
  user_id VARCHAR,
  amount DOUBLE,
  status VARCHAR
) WITH (
  KAFKA_TOPIC='orders',
  VALUE_FORMAT='AVRO'
);

-- Потоковый запрос: подсчёт заказов по статусам за 5-минутные окна
CREATE TABLE order_stats AS
  SELECT status,
         COUNT(*) AS order_count,
         SUM(amount) AS total_amount
  FROM orders
  WINDOW TUMBLING (SIZE 5 MINUTES)
  GROUP BY status;
```

**Связка с другими компонентами:** ksqlDB тесно интегрирован с Schema Registry (для Avro/Protobuf/JSON Schema), использует Kafka как storage layer и может читать/писать напрямую в топики.

---

## 2. Базы данных: CDC и прямая интеграция

### 2.1 Change Data Capture (CDC) через Debezium

**Debezium** — специализированный набор Kafka Connect Source коннекторов для захвата изменений на уровне строк базы данных [4][5].

**Поддерживаемые базы и механизмы захвата:**

| База данных | Механизм CDC | Типичная задержка |
|-------------|-------------|-------------------|
| **PostgreSQL** | Logical Decoding (WAL) | <1 сек |
| **MySQL** | Binary Log (binlog) | <1 сек |
| **MongoDB** | Change Streams / Oplog | <1 сек |
| **SQL Server** | Change Tracking / CDC Tables | <5 сек |
| **Oracle** | LogMiner / XStream | <5 сек |

**Типичный поток Debezium → Kafka:**

```
┌──────────┐    INSERT/UPDATE    ┌──────────┐    CDC Event     ┌─────────┐
│PostgreSQL│──────────────────►│ Debezium │───────────────►│  Kafka   │
│          │      /DELETE        │Connector │                 │  Topic   │
└──────────┘                     └──────────┘                 └────┬────┘
                                                                   │
                                          ┌────────────────────────┘
                                          ▼
                              ┌──────────────────────┐
                              │ Потребители CDC:      │
                              │ • Search index sync   │
                              │ • Cache invalidation  │
                              │ • Audit logging       │
                              │ • Data warehouse ETL  │
                              └──────────────────────┘
```

**Почему это важно:** CDC позволяет строить архитектуры, где Kafka — это центральная нервная система данных. Любое изменение в любой БД транслируется в Kafka и оттуда доставляется всем заинтересованным потребителям. Это устраняет проблемы точечных интеграций (N × M соединений) и превращает их в N + M (через единый хаб).

**Реальный сценарий:** Интернет-магазин использует PostgreSQL для заказов. Через Debezium каждое изменение заказа попадает в топик `orders.cdc`. Потребители: Elasticsearch (поиск заказов), Redis (кеш статуса), Data Warehouse (аналитика), сервис уведомлений (email при смене статуса). Ни один потребитель не подключается к PostgreSQL напрямую.

### 2.2 JDBC-интеграция (batch-oriented)

Для сценариев, не требующих real-time CDC, используется JDBC Connector — он работает в режиме опроса (polling): раз в N секунд проверяет таблицу на наличие новых/изменённых строк по столбцу-метке (timestamp или auto-increment). Подходит для:
- Пакетной синхронизации данных без требований к низкой задержке
- Обратной загрузки обработанных данных из Kafka в БД
- Простых ETL-пайплайнов без необходимости в CDC

---

## 3. Потоковая обработка: Flink и Spark как компаньоны Kafka

### 3.1 Apache Flink — stateful stream processor

**Роль:** Самая мощная платформа для сложной stateful-обработки потоков с exactly-once семантикой.

**Связка Flink ↔ Kafka [6]:**
- Flink читает из Kafka как из unbounded источника
- Flink пишет результаты обратно в Kafka
- Flink использует Kafka для checkpoint-ов (сохранение состояния)
- Flink и Kafka совместно обеспечивают end-to-end exactly-once

**Когда Flink + Kafka лучше, чем Kafka Streams:**
- Нужна обработка с миллисекундной задержкой и сложная event-time логика
- Требуется интеграция с не-Kafka источниками (файлы, другие шины данных)
- Нужен отдельный вычислительный кластер с продвинутым мониторингом и управлением
- SQL-интерфейс (FlinkSQL) как альтернатива ksqlDB для SQL-ориентированных команд

### 3.2 Apache Spark — batch + micro-batch

**Роль:** Универсальный движок обработки данных — от batch ETL до near-realtime streaming через Structured Streaming.

**Связка Spark ↔ Kafka [7]:**
- Spark читает из Kafka через `spark.readStream.format("kafka")`
- Spark пишет в Kafka через `df.writeStream.format("kafka")`
- Spark Structured Streaming использует micro-batching по умолчанию (latency ~100ms+) либо Continuous Processing (experimental, ~1ms)

**Когда Spark + Kafka:**
- Организация уже использует Spark для batch-обработки и хочет унифицировать стек
- Нужны сложные ML-трансформации (Spark MLlib) поверх потоковых данных
- Требуется одновременно batch и streaming обработка из одних и тех же Kafka-топиков

### 3.3 Kafka Streams vs Flink vs Spark — выбор процессора

| Характеристика | Kafka Streams | Apache Flink | Spark Structured Streaming |
|----------------|---------------|--------------|---------------------------|
| **Развёртывание** | Библиотека в JVM-приложении | Отдельный кластер | Отдельный кластер |
| **Модель** | Event-at-a-time | Event-at-a-time | Micro-batch (основная) |
| **Гарантии** | Exactly-once (Kafka→Kafka) | Exactly-once (end-to-end) | At-least-once / Exactly-once |
| **Состояние** | RocksDB (локально) | RocksDB (локально) | State store (HDFS-бэкенд) |
| **Среда** | Только JVM | JVM + Python, SQL | JVM, Python, R, SQL |
| **Порог входа** | Низкий (Java-разработчикам) | Средний | Низкий (PySpark знаком многим) |

---

## 4. Хранилища и аналитика: куда уходят данные из Kafka

### 4.1 Elasticsearch / OpenSearch — поисковые и аналитические индексы

**Типичный сценарий:** Kafka собирает логи, события, изменения данных → Sink Connector пишет в Elasticsearch → Kibana/OpenSearch Dashboards для визуализации и поиска [8].

**Ключевая ценность:** Kafka буферизует поток и гарантирует доставку даже при временной недоступности Elasticsearch. Без Kafka пиковый всплеск логов «положит» Elasticsearch. С Kafka — Elasticsearch может потреблять в своём темпе.

### 4.2 Объектные хранилища — Data Lake

**S3 (AWS), GCS (Google), Azure Blob, MinIO** — типовые конечные точки для долговременного хранения событий.

**Паттерн:** Kafka — горячий слой (задержка ms, retention дни), S3 — холодный слой (задержка минуты, retention годы). Sink-коннекторы пишут данные в Avro/Parquet, партицируя по времени.

**Современный тренд:** Tiered Storage (KIP-405) в Kafka 3.6+ позволяет прозрачно перемещать холодные сегменты из локального диска в S3, делая Kafka практически бесконечным хранилищем.

### 4.3 Hadoop HDFS / Hive — классический Data Warehouse

Kafka → HDFS Sink Connector или Spark streaming job → Hive-таблицы. Традиционная связка в Hadoop-ориентированных data-платформах. Постепенно вытесняется облачными решениями (BigQuery, Snowflake, Databricks Delta Lake).

---

## 5. Инфраструктурный слой: оркестрация, мониторинг, кеширование

### 5.1 Kubernetes — оркестрация и жизненный цикл

**Роль:** Платформа для запуска Kafka в контейнеризированной среде.

**Два основных оператора [9][10]:**

| Характеристика | Strimzi | Confluent for Kubernetes (CFK) |
|----------------|---------|-------------------------------|
| **Лицензия** | Apache 2.0 (open-source) | Commercial (Confluent) |
| **Управление** | CRD (Kafka, KafkaConnect, KafkaMirrorMaker) | CRD + Confluent-специфичные ресурсы |
| **TLS/сертификаты** | Встроенное управление | Встроенное + внешнее |
| **Мониторинг** | Prometheus exporter из коробки | Confluent Control Center |
| **Обновления** | Rolling update с проверкой | Автоматические + canary |

**Почему Kubernetes + Kafka вместе:** Kubernetes берёт на себя оркестрацию, масштабирование, мониторинг пода (контейнера) и автоматический перезапуск при сбоях. Strimzi добавляет Kafka-специфичную логику: знает, как добавлять брокер в кластер, как реплицировать топики после расширения, как обновлять без потери данных.

### 5.2 Prometheus + Grafana — мониторинг и наблюдаемость

**Связка [11]:**
- Kafka экспортирует метрики через JMX → Prometheus JMX Exporter → Prometheus
- Альтернативно: Kafka Exporter (специализированный экспортер для consumer lag, offset и т.д.)
- Grafana визуализирует дашборды: consumer lag, throughput per partition, broker disk usage, network I/O

**Ключевые метрики для мониторинга:**
- `kafka_consumergroup_group_lag` — отставание потребителя (критичный индикатор)
- `kafka_server_BrokerTopicMetrics_BytesInPerSec` — скорость записи
- `kafka_server_BrokerTopicMetrics_BytesOutPerSec` — скорость чтения
- `kafka_controller_ActiveControllerCount` — активен ли контроллер (должен быть ровно 1)

### 5.3 Redis — кеширующий слой

**Роль:** Кеш для быстрого доступа к данным, проходящим через Kafka.

**Паттерн:** Kafka получает CDC-события из PostgreSQL → Kafka Streams/ksqlDB материализует актуальное состояние → Результат пишется в Redis для быстрого key-value доступа с задержкой <1ms. Без Kafka синхронизация Redis и PostgreSQL потребовала бы кастомного кода и рисковала рассинхронизацией.

### 5.4 ZooKeeper / KRaft — управление метаданными

**Исторически:** Kafka до версии 4.0 использовала Apache ZooKeeper для хранения метаданных кластера (список брокеров, лидеры партиций, конфигурация топиков) [12].

**Современность (Kafka 4.0+, 2024):** ZooKeeper полностью заменён встроенным протоколом консенсуса KRaft. Это устраняет необходимость в отдельном ZooKeeper-ансамбле (обычно 3-5 серверов), упрощает развёртывание и снижает операционную сложность. KRaft использует тот же механизм лога (Raft), но встроенный в сам Kafka-брокер.

**Почему это важно для смежных технологий:** Уход от ZooKeeper упрощает развёртывание Kafka в Kubernetes (меньше компонентов) и уменьшает поверхность атаки (меньше портов, меньше сервисов для аудита безопасности).

---

## 6. Карта взаимодействия: что с чем связано

Интеграционная паутина Kafka выглядит так:

```
                          ┌─────────────────────┐
                          │  Schema Registry     │◄─── Avro/Protobuf схемы
                          │  (Управление схемами) │
                          └──────────┬──────────┘
                                     │
┌─────────────────┐         ┌────────▼──────────┐         ┌──────────────────┐
│ Базы данных      │  CDC    │                    │  Sink   │ Потребители       │
│ • PostgreSQL    ├────────►│    Apache Kafka    ├────────►│ • Elasticsearch  │
│ • MySQL         │ Debezium│                    │ Connect │ • S3/Data Lake   │
│ • MongoDB       │         │  (Центральная шина) │         │ • Redis (кеш)    │
│ • Oracle        │         │                    │         │ • HDFS/Hive      │
│ • SQL Server    │         └──────┬──────┬──────┘         └──────────────────┘
└─────────────────┘                │      │
                                   │      │
                       ┌───────────▼┐    ┌▼──────────────┐
                       │ Обработка   │    │ Инфраструктура │
                       │ • Flink    │    │ • Kubernetes   │
                       │ • Spark    │    │ • Prometheus   │
                       │ • ksqlDB   │    │ • Grafana      │
                       │ • Streams  │    │ • KRaft        │
                       └────────────┘    └────────────────┘
```

**Ключевое наблюдение:** Kafka в центре этой схемы — не потому что она «самая умная», а потому что она самая надёжная как буфер. Все стрелки ведут к Kafka в первую очередь ради гарантии доставки и декаплинга producer/consumer.

---

## 7. Практические правила выбора технологий-компаньонов

1. **Если данные нужно достать из БД в реальном времени** → Debezium + Kafka Connect
2. **Если нужна потоковая обработка и команда знает Java** → Kafka Streams
3. **Если нужна обработка и команда знает SQL** → ksqlDB
4. **Если нужна сложная stateful-аналитика на нескольких источниках** → Flink + Kafka
5. **Если организация уже использует Spark и хочет унифицировать стек** → Spark + Kafka
6. **Если данные нужно искать полнотекстовым поиском** → Kafka → Elasticsearch Sink
7. **Если данные нужно хранить долго и дёшево** → Kafka → S3 Sink (с tiered storage или без)
8. **Если Kafka разворачивается в Kubernetes** → Strimzi (open-source) или CFK (enterprise)
9. **Если нужна валидация форматов и эволюция схем** → Schema Registry обязательно
10. **Если нужен мониторинг** → Prometheus + Grafana (де-факто стандарт)

Каждая из этих технологий будет подробно разобрана в соответствующих треках: Debezium и коннекторы — в [Ecosystem](02-ecosystem.md), Flink/Spark vs Kafka — в [VS Alternatives](03-vs-alternatives.md), мониторинг и Prometheus — в [треке Operations](../../07-operations/01-monitoring.md).

---

## Источники

1. Apache Kafka Ecosystem page — https://kafka.apache.org/37/getting-started/ecosystem/ (Tier 1, официальная документация)
2. Cloudurable — Kafka Ecosystem 2025 Edition — https://cloudurable.com/blog/kafka-ecosystem-2025/ (Tier 2, специализированный блог)
3. Confluent Docs — Formats, Serializers, and Deserializers — https://docs.confluent.io/platform/current/schema-registry/fundamentals/serdes-develop/ (Tier 1, официальная документация)
4. Debezium Documentation — https://debezium.io/documentation/reference/stable/tutorial.html (Tier 1, официальная документация)
5. Debezium + Kafka Connect CDC guide, codestudy.net — https://www.codestudy.net/blog/debezium-kafka-connect/ (Tier 3, технический блог)
6. Real-Time Data Pipelines: Kafka, Flink, Spark — https://calmops.com/database/real-time-data-pipelines-kafka-flink-spark/ (Tier 3, технический ресурс)
7. End-to-End Data Pipeline (Kafka + Spark + Airflow) — https://github.com/hoangsonww/end-to-end-data-pipeline (Tier 2, GitHub reference architecture)
8. Elastic + Kafka integration — https://www.elastic.co/search-labs/blog/elasticsearch-apache-kafka-ingest-data (Tier 1, официальный блог Elastic)
9. Strimzi — Apache Kafka on Kubernetes — https://strimzi.io/ (Tier 1, официальный сайт проекта)
10. Confluent for Kubernetes — https://docs.confluent.io/operator/current/overview.html (Tier 1, официальная документация Confluent)
11. Microsoft Learn — Prometheus metrics for Kafka — https://learn.microsoft.com/en-us/azure/azure-monitor/containers/prometheus-kafka-integration (Tier 2, облачный вендор)
12. Apache Kafka 4.0 Release Blog — https://confluent.io/blog/introducing-apache-kafka-4-0 (Tier 2, официальный блог Confluent)

---

*Далее: [Ecosystem](02-ecosystem.md) — коннекторы, плагины, SDK и языковые привязки Kafka*
