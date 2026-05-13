# Open-Source проекты в экосистеме Apache Kafka: community tooling, self-hosted options и лицензионная карта

**Трек 04 — Software** | Статья 3 из 4  
**Проект:** kafka | **Профиль:** A (Infrastructure Platform)  
**Дата:** 2026-05-13

---

## Содержание

1. [Обзор: границы open-source в мире Kafka](#обзор-границы-open-source-в-мире-kafka)
2. [Ядро Apache Kafka: что под Apache 2.0](#ядро-apache-kafka-что-под-apache-20)
3. [Уровень операций: Strimzi и другие CNCF-проекты](#уровень-операций-strimzi-и-другие-cncf-проекты)
4. [Уровень данных: Debezium и CDC-экосистема](#уровень-данных-debezium-и-cdc-экосистема)
5. [Уровень управления: GUI, CLI и governance](#уровень-управления-gui-cli-и-governance)
6. [Альтернативы Kafka: open-source или source-available?](#альтернативы-kafka-open-source-или-source-available)
7. [Community tooling: карта инструментов](#community-tooling-карта-инструментов)
8. [Self-hosted vs managed: что вы теряете и приобретаете](#self-hosted-vs-managed-что-вы-теряете-и-приобретаете)
9. [Лицензионная карта: Apache 2.0 vs SSPL vs BSL](#лицензионная-карта-apache-20-vs-sspl-vs-bsl)
10. [Практический чек-лист: выбор open-source стека](#практический-чек-лист-выбор-open-source-стека)
11. [Источники](#источники)

---

## Обзор: границы open-source в мире Kafka

Apache Kafka — один из самых успешных open-source проектов Apache Software Foundation. Его ядро (брокер, протокол, базовые клиенты) лицензировано под Apache 2.0 — лицензией, которая разрешает использование, модификацию, распространение и коммерческое использование без каких-либо ограничений.

**Ключевая мысль:** экосистема Kafka — это трёхслойный пирог. Ядро (сердцевина) — полностью открытое и бесплатное. Инструментальный слой (Strimzi, AKHQ, Cruise Control, Debezium) — в основном тоже открытый, но уже с нюансами. Периферия (Kafka-совместимые альтернативы вроде Redpanda и WarpStream) — всё чаще уходит в source-available и BSL-лицензии, создавая ловушки для тех, кто не читает мелкий шрифт.

**Почему это важно именно сейчас (2026):** за последние три года произошло два тектонических сдвига. Во-первых, Kafka 4.0 (март 2025) окончательно перешёл на KRaft, похоронив ZooKeeper — и вместе с ним снял крупнейший операционный барьер для self-hosted инсталляций. Во-вторых, война лицензий (Confluent Community License 2018 → SSPL-переходы в 2024–2025 у смежных проектов) радикально изменила правила игры для тех, кто планирует строить бизнес на перепродаже managed Kafka.

**Структура статьи:** мы пройдём от ядра к периферии — что действительно открыто и бесплатно, какие community-инструменты решают реальные боли, где заканчивается Apache 2.0 и начинаются подводные камни source-available лицензий.

---

## Ядро Apache Kafka: что под Apache 2.0

### Что входит в ядро

Apache Kafka как проект Apache Software Foundation включает следующие компоненты, все под Apache License 2.0:

| Компонент | Описание | Статус |
|-----------|----------|--------|
| **Kafka Broker** | Основной брокер сообщений | ✅ Apache 2.0 |
| **Kafka Clients** | Java-клиент (Producer + Consumer API) | ✅ Apache 2.0 |
| **Kafka Streams** | Библиотека потоковой обработки (встроена в ядро) | ✅ Apache 2.0 |
| **Kafka Connect** | Фреймворк для коннекторов (runtime встроен в ядро) | ✅ Apache 2.0 |
| **MirrorMaker 2** | Инструмент кросс-кластерной репликации (KIP-382) | ✅ Apache 2.0 |
| **KRaft (Kafka Raft)** | Встроенный консенсус-движок (замена ZooKeeper) | ✅ Apache 2.0 |
| **Kafka Admin Client** | Клиент для административных операций | ✅ Apache 2.0 |

**Исходный код:** [github.com/apache/kafka](https://github.com/apache/kafka) — 32K+ звёзд, 15K+ форков, ~1200 контрибьюторов.

### Что НЕ входит в ядро (распространённое заблуждение)

Часто думают, что «Kafka из коробки» включает всё перечисленное ниже — но это отдельные проекты с собственными лицензиями:

- **Schema Registry** — разрабатывается Confluent, но есть открытые реализации
- **ksqlDB** — Confluent Community License (не Apache 2.0)
- **REST Proxy** — Confluent Community License
- **Control Center** — Confluent Enterprise (коммерческая)
- **Connector Hub** (120+ коннекторов) — смешанные лицензии, многие под Confluent Community License

**Практический вывод:** ядро Kafka — полностью открытое. Вы можете скачать, собрать, модифицировать, запустить в production и даже перепродавать как managed service. Именно так делают AWS (MSK), Aiven, Instaclustr и десятки других.

### KIP-процесс: как сообщество развивает ядро

Kafka Improvement Proposal (KIP) — формальный процесс внесения изменений в ядро. Аналог Python PEP или Kubernetes KEP. Процесс полностью открытый:

1. **Любой** может предложить KIP через Apache Confluence Wiki
2. Дискуссия проходит в публичном mailing list
3. Голосование — коммиттеры проекта (около 40 человек на 2026 год)
4. Принятые KIP реализуются через GitHub Pull Requests

По состоянию на 2026 год принято **800+ KIP** (с KIP-1 в 2013 году до KIP-1000+ в 2024). Ключевые недавние KIP:

- **KIP-500** (2019) — миграция с ZooKeeper на KRaft
- **KIP-848** (2023) — новый протокол consumer group rebalancing
- **KIP-966** (2024) — поддержка ARM64 для KRaft
- **KIP-1057** (2025) — Tiered Storage GA в Kafka 4.0

---

## Уровень операций: Strimzi и другие CNCF-проекты

### Strimzi: Kafka на Kubernetes как код

**Strimzi** — CNCF Incubating проект (принят 2019, Incubating с февраля 2024), оператор Kubernetes для управления Kafka-кластерами. Лицензия: **Apache 2.0**.

**Что делает:**
- Декларативное управление Kafka-кластером через Kubernetes Custom Resources
- Управляет не только брокерами, но и Kafka Connect, MirrorMaker 2, Kafka Bridge, Cruise Control
- Автоматическое управление TLS-сертификатами
- Поддержка аутентификации: mTLS, SCRAM-SHA-512, OAuth 2.0
- Rack awareness (разнесение брокеров по availability zones)
- Поддержка GitOps (все ресурсы — YAML, совместимы с ArgoCD/Flux)

**Быстрый старт:**
```yaml
# Установка оператора (последняя стабильная версия 0.51.0)
kubectl create namespace kafka
kubectl apply -f 'https://strimzi.io/install/latest' -n kafka

# Создание кластера из трёх брокеров (KRaft mode)
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles: [controller]
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles: [broker]
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 4.2.0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

**CNCF статус:**
- Принят в CNCF: 28 августа 2019
- Уровень Incubating: 8 февраля 2024
- ~5.8K звёзд на GitHub, активное сообщество
- StrimziCon — собственная виртуальная конференция

**Почему Strimzi важен для open-source стека:** это де-факто стандарт для запуска self-hosted Kafka на Kubernetes. Без него вам пришлось бы вручную настраивать StatefulSets, Service-объекты, сертификаты, обновления брокеров с rolling restart — сотни строк YAML и shell-скриптов.

### Cruise Control: LinkedIn под капотом

**Cruise Control** — open-source проект LinkedIn (создатели Kafka), Apache 2.0. 3K+ звёзд на GitHub.

**Что делает:**
- Автоматическая ребалансировка партиций между брокерами
- Мониторинг нагрузки на брокеры (CPU, сеть, диск)
- Выявление аномалий (перегруженные/недогруженные брокеры)
- Предложение плана ребалансировки (можно применить вручную или автоматически)
- Самоисцеление (self-healing) — автоматическая замена упавших брокеров

**Интеграция со Strimzi:** Strimzi умеет разворачивать Cruise Control как часть Kafka-кластера.

```yaml
# Включение Cruise Control через Strimzi
spec:
  cruiseControl:
    image: quay.io/strimzi/kafka:latest-kafka-4.2.0
    config:
      goal.violation.detection.enabled: "true"
      self.healing.enabled: "true"
```

**Альтернативы:** Confluent Auto-Balancer (Confluent Platform, коммерческая), KafScale Auto-Rebalancer (новый проект, 2026).

### Kafka Bridge (Strimzi)

HTTP-мост для доступа к Kafka-топикам без нативного клиента. Позволяет читать/писать сообщения через REST API.

- Лицензия: Apache 2.0
- Продакшн-состояние: v1.0.0 (2025)
- Использование: IoT-устройства, браузерные приложения, legacy-системы без Java

---

## Уровень данных: Debezium и CDC-экосистема

### Debezium: Change Data Capture как сердцевина архитектуры

**Debezium** — проект с открытым исходным кодом для CDC (Change Data Capture), 12.6K+ звёзд на GitHub. Лицензия: **Apache 2.0**.

**Что делает:**
- Захватывает изменения строк (INSERT, UPDATE, DELETE) из базы данных в реальном времени
- Транслирует их в Kafka-топики как события
- Поддерживает 11+ баз данных: PostgreSQL, MySQL, MongoDB, SQL Server, Oracle, Db2, Cassandra, Vitess, Spanner, YugabyteDB, MariaDB

**Архитектура Debezium:**
- Подключается к WAL (Write-Ahead Log) реляционных БД или oplog MongoDB
- Каждое изменение → событие в Kafka с before/after state
- Формат события — унифицированный (структура Debezium event)
- Работает как Kafka Connect Source Connector

**Поддерживаемые коннекторы (все Apache 2.0):**

| Коннектор | База данных | Механизм захвата | Статус |
|-----------|------------|------------------|--------|
| PostgreSQL | PostgreSQL 10–17 | Логическая репликация (pgoutput/decoderbufs) | GA |
| MySQL | MySQL 5.7–8.4 | Binlog reader | GA |
| MongoDB | MongoDB 4.2–7.0 | Change Streams | GA |
| SQL Server | SQL Server 2017–2022 | CDC tables | GA |
| Oracle | Oracle 12c–21c | LogMiner / XStream | GA |
| Db2 | IBM Db2 11.5+ | CDC tables | Incubating |
| Cassandra | Apache Cassandra 3.11+ | Commit log reader | Incubating |

**Пример конфигурации Debezium PostgreSQL:**
```json
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.dbname": "inventory",
    "topic.prefix": "dbserver1",
    "plugin.name": "pgoutput",
    "slot.name": "debezium",
    "publication.autocreate.mode": "filtered"
  }
}
```

**Debezium Server:** отдельный проект (Apache 2.0) позволяет запускать Debezium-коннекторы без Kafka — напрямую в Google Pub/Sub, Amazon Kinesis, Redis Streams, HTTP-клиенты.

### Альтернативы CDC в open-source

| Инструмент | Модель | Лицензия | Сравнение с Debezium |
|-----------|--------|----------|---------------------|
| **Maxwell's Daemon** | MySQL → Kafka | MIT | Только MySQL, проще, меньше фич |
| **pgcapture** | PostgreSQL → Kafka | Apache 2.0 | Молодой проект, меньше production-проверок |
| **Airbyte** | ETL/ELT платформа | MIT (Core) / ELv2 (Connectors) | Шире CDC, но больше overhead |
| **PeerDB** | PostgreSQL CDC | ELv2 | Специализация на Postgres, выше скорость |

---

## Уровень управления: GUI, CLI и governance

### Kafka UI: provectus/kafka-ui и kafbat/kafka-ui

История раздвоения, важная для понимания open-source динамики.

**provectus/kafka-ui** (оригинальный проект):
- 12K+ звёзд, Apache 2.0
- Web UI для управления Kafka-кластерами
- Функции: просмотр/создание/удаление топиков, просмотр сообщений, consumer groups, Schema Registry, Kafka Connect
- Поддерживает несколько кластеров одновременно
- Встроенная аутентификация (Basic, OAuth, LDAP)

**kafbat/kafka-ui** (форк от сообщества):
- 2.2K+ звёзд, Apache 2.0
- Создан в 2024 году после замедления развития оригинального проекта
- TypeScript-first архитектура (59% TS vs 38% Java)
- Активнее обновляется в 2025–2026

**Docker-запуск за одну минуту:**
```yaml
version: '3'
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

### AKHQ — альтернативный взгляд на GUI

**AKHQ** (akhq.io, ранее tchiotludo/akhq):
- 3.8K+ звёзд, Apache 2.0
- Java + JavaScript (React)
- Делает акцент на визуализации данных внутри топиков — удобный просмотр сообщений с фильтрацией и поиском
- Поддерживает Avro, Protobuf десериализацию «из коробки»
- Live tail — просмотр сообщений в реальном времени (как `tail -f` для топика)

**Сравнение с Kafka UI:**

| Критерий | Kafka UI (provectus) | AKHQ | Kafdrop |
|----------|---------------------|------|---------|
| Звёзды GitHub | 12K | 3.8K | 6.1K |
| Просмотр сообщений | ✔️ базовый | ✔️ продвинутый (фильтры, поиск, tail) | ✔️ базовый |
| Schema Registry | ✔️ | ✔️ Avro/Protobuf | ✔️ Avro |
| Kafka Connect | ✔️ | ✔️ | ❌ |
| Multi-cluster | ✔️ | ✔️ | ❌ |
| Аутентификация | Basic/OAuth/LDAP | LDAP/OIDC/OAuth | Нет |
| Лицензия | Apache 2.0 | Apache 2.0 | Apache 2.0 |

### Kafdrop и другие легковесные UI

**Kafdrop** (obsidiandynamics/kafdrop):
- 6.1K звёзд, Apache 2.0
- Минималистичный UI: топики, партиции, consumer groups, просмотр сообщений
- Хорош для разработки и небольших кластеров
- Нет поддержки Schema Registry и Kafka Connect

**Kouncil** (Consdata/kouncil):
- 290+ звёзд, Apache 2.0
- Web-дэшборд с мониторингом, управлением группами, диагностикой
- Специализация на визуальном трейсинге сообщений (event flow visualisation)

### Klaw: governance для Kafka

**Klaw** — open-source governance toolkit от Aiven. Лицензия: **Apache 2.0**.

**Проблема, которую решает:** в больших организациях Kafka используют десятки команд. Без governance любой разработчик может создать топик, забить кластер, перехватить чужие данные. Klaw добавляет слой RBAC (Role-Based Access Control) поверх Kafka:

- **Запросы на создание топиков** с approval workflow
- **RBAC назначения:** кто может создавать/читать/писать в топики
- **Управление схемами:** кто может регистрировать/обновлять Avro-схемы
- **Управление коннекторами:** approval на развёртывание Connect-коннекторов
- **Аудит:** кто и когда создал/изменил/удалил ресурс
- **Self-service портал:** разработчики запрашивают доступ через UI, админы одобряют

**Архитектура:**
- Klaw Core — основной сервис governance и метаданных
- Klaw Cluster API — соединение с Kafka-кластерами, Schema Registry, Connect
- UI — веб-интерфейс для self-service

**Быстрый старт:**
```bash
docker run -d -t -i \
  -e KLAW_CLUSTERAPI_ACCESS_BASE64_SECRET="dGhpcyBpcyBhIHNlY3JldCB0byBhY2Nlc3MgY2x1c3RlcmFwaQ==" \
  -p 9343:9343 \
  --name klaw-cluster-api aivenoy/klaw-cluster-api:latest
```

### CLI-инструменты поверх стандартных

**kcctl** (kcctl/kcctl):
- 420+ звёзд, Apache 2.0
- Современный CLI для Kafka Connect (современнее стандартного `connect-distributed.sh`)
- Функции: `kcctl get connectors`, `kcctl restart connector`, `kcctl logs connector`

**kcat** (ранее kafkacat):
- Утилита для работы с Kafka в стиле netcat, Apache 2.0
- Не-Java клиент — работает на C/librdkafka
- Идеален для отладки и скриптов

---

## Альтернативы Kafka: open-source или source-available?

Это самый горячий участок экосистемы. За последние 2 года появилось несколько Kafka-совместимых платформ, которые обещают «Kafka без боли». Но их лицензии — минное поле.

### Redpanda: Apache 2.0 с оговорками

**Redpanda** — переписанный на C++ Kafka-брокер, совместимый с Kafka API.

| Аспект | Детали |
|--------|--------|
| **Лицензия** | BSL (Business Source License) → Apache 2.0 через 4 года |
| **Совместимость** | Полная совместимость с Kafka Protocol API |
| **Архитектура** | Thread-per-core, Direct I/O, без page cache |
| **Замена ZooKeeper** | Встроенный Raft (свой, не KRaft) |
| **Self-hosted** | ✅ бесплатно для production |
| **SaaS/перепродажа** | ❌ запрещено BSL |

**BSL-ограничения Redpanda:**
- Можно использовать **внутри компании** для любых целей
- Нельзя **продавать как managed service** (это прямая конкуренция с Redpanda Cloud)
- Через 4 года после релиза каждая версия переходит на Apache 2.0

**Технические отличия от Apache Kafka:**
- Thread-per-core вместо thread-per-connection: ниже latency, выше throughput на ядро
- Direct I/O в обход page cache: предсказуемая производительность, нет double-buffering
- Нет JVM → нет GC-пауз, меньше памяти на брокер
- Встроенный Schema Registry (в отличии от Kafka, где SR — отдельный сервис)

### WarpStream: дисклесс-архитектура → поглощён Confluent

**WarpStream** (2023–2024):
- Архитектура: zero local disk, всё в S3
- Лицензия: BSL → в сентябре 2024 поглощён Confluent за $600M
- После поглощения: WarpStream стал «Confluent WarpStream», лицензия не менялась
- **Практический вывод:** теперь это фактически проприетарный продукт Confluent

### AutoMQ: Cloud-native Kafka

**AutoMQ:**
- 100% совместимость с Kafka API
- Архитектура: stateless брокеры, все данные в S3
- Лицензия: Apache 2.0 для ядра, но некоторые enterprise-фичи — ELv2
- Поддерживает shared-nothing scaling (брокеры без жёсткой привязки к данным)

**Сравнение альтернатив по self-hosted применимости:**

| Платформа | Self-hosted | Архитектура | Лицензия | Риск vendor lock-in |
|-----------|-------------|-------------|----------|-------------------|
| Apache Kafka | ✅ Полная | Брокер + локальный диск | Apache 2.0 | Нулевой |
| Redpanda | ✅ Бесплатно для своих | C++ брокер + локальный диск | BSL → Apache 2.0 | Средний (BSL-ограничения) |
| AutoMQ | ✅ Бесплатно для своих | Stateless + S3 | Apache 2.0 + ELv2 | Низкий (совместимость с Kafka API) |
| WarpStream | ⚠️ Только через Confluent | Zero-disk, S3-only | BSL (Confluent) | Высокий |
| KafScale | ✅ Бесплатно | Stateless Kafka на S3 | Apache 2.0 | Низкий |

### Почему это важно для self-hosted стратегии

Допустим, вы — стартап, который хочет запустить streaming-платформу и потом продавать её как SaaS. Если вы возьмёте Redpanda — через год получите cease-and-desist от их юристов (потому что SaaS-перепродажа запрещена BSL). Если возьмёте WarpStream — вы теперь клиент Confluent. Apache Kafka под Apache 2.0 таких проблем не создаёт — именно на нём построены AWS MSK, Aiven Kafka, Instaclustr.

**Правило большого пальца:** если ваша бизнес-модель не подразумевает перепродажу managed Kafka — берите что угодно. Если подразумевает — только Apache 2.0, никаких BSL.

---

## Community tooling: карта инструментов

Open-source экосистема Kafka — это не только официальные проекты, но и сотни community-инструментов. Вот карта по категориям.

### Мониторинг и наблюдаемость

| Инструмент | Назначение | Лицензия | Звёзды |
|-----------|------------|----------|--------|
| **LinkedIn Burrow** | Мониторинг consumer lag | Apache 2.0 | 3.5K |
| **Kafka Lag Exporter** | Prometheus exporter для lag | Apache 2.0 | 800+ |
| **Kafka JMX Exporter** | Экспорт JMX-метрик в Prometheus | Apache 2.0 | Официальный Prometheus |
| **kminion** | Prometheus exporter + health checks | MIT | 600+ |
| **AxonOps** | Cassandra + Kafka observability | Apache 2.0 | — |

### Schema Registry (альтернативы Confluent)

**Apicurio Registry** (Red Hat):
- Полная реализация Schema Registry API, совместим с Confluent
- Поддерживает Avro, Protobuf, JSON Schema, OpenAPI, AsyncAPI, GraphQL
- Лицензия: Apache 2.0
- Может работать как с Kafka, так и с другими хранилищами (PostgreSQL, Infinispan)

**Karapace** (Aiven):
- Schema Registry + REST Proxy в одном
- Лицензия: Apache 2.0
- Совместим с Confluent Schema Registry API

### Kafka Connect коннекторы (независимые)

Большинство коннекторов на Confluent Hub — под Confluent Community License. Но есть значительное количество Apache 2.0 коннекторов от сообщества:

- **Debezium-коннекторы** (все под Apache 2.0) — для баз данных
- **JDBC Sink Connector** (Aiven) — Apache 2.0 альтернатива Confluent JDBC
- **S3 Sink Connector** (Aiven) — Apache 2.0, запись в S3
- **Elasticsearch Sink Connector** (Aiven) — Apache 2.0, запись в Elasticsearch
- **BigQuery Sink Connector** (встроен в Kafka 4.0+) — Apache 2.0

### MirrorMaker 2 расширения

**Strimzi MirrorMaker 2 Extensions:**
- Apache 2.0
- Дополнительные Identity Replication Policies для сохранения смещений и групп при репликации
- Улучшенная интеграция со Schema Registry

### JulieOps (GitOps для Kafka)

**JulieOps** (бывший kafka-gitops):
- Управление топиками, ACL, принципалами через YAML в Git
- Принцип: «ваш Git-репозиторий — source of truth для Kafka-конфигурации»
- Лицензия: Apache 2.0

```yaml
# Пример конфигурации JulieOps
topics:
  orders-topic:
    replication_factor: 3
    partitions: 12
    configs:
      retention.ms: "604800000"
      cleanup.policy: "delete"
  dlq-topic:
    replication_factor: 3
    partitions: 3
    configs:
      retention.ms: "2592000000"

acls:
  - principal: "User:orders-service"
    host: "*"
    operation: WRITE
    resource_type: TOPIC
    resource_name: "orders-topic"
```

### Kafka REST Proxy (альтернативы)

- **Karapace REST** (Aiven) — Apache 2.0 REST Proxy
- **Strimzi Kafka Bridge** — Apache 2.0 HTTP-мост (проще REST Proxy, меньше фич)

---

## Self-hosted vs managed: что вы теряете и приобретаете

### Self-hosted Apache Kafka: полная картина

**Что вы получаете, запуская open-source Kafka самостоятельно:**

| Аспект | Self-hosted (Apache Kafka) |
|--------|---------------------------|
| **Стоимость лицензий** | ₽0 — полностью бесплатно |
| **Контроль над данными** | Полный — данные на вашем железе |
| **Контроль над конфигурацией** | Полный — правите server.properties напрямую |
| **Кастомизация** | Безграничная — можете патчить ядро |
| **Сложность эксплуатации** | Высокая — нужен DevOps-опыт |
| **Апдейты и патчи** | Ваша ответственность |
| **Мониторинг** | Сами настраиваете Prometheus/Grafana |
| **Балансировка нагрузки** | Сами через Cruise Control |
| **DR (Disaster Recovery)** | Сами через MirrorMaker 2 |

**Что вы теряете по сравнению с managed-решениями:**
- Автоматическое масштабирование партиций
- Self-service порталы для разработчиков (можно добавить Klaw)
- SLA гарантии (99.95% uptime вам придётся обеспечивать самим)
- Интегрированный мониторинг из коробки
- Поддержку и экспертизу вендора

### Когда self-hosted оправдан

**Self-hosted имеет смысл, если:**
1. **Regulatory compliance** — данные не могут покидать ваш ЦОД (банки, госсектор)
2. **Cost at scale** — при >100 партиций managed-решения становятся дороже self-hosted
3. **Custom security** — нужна интеграция с кастомными KMS/HSM
4. **Low latency** — брокеры должны быть в том же ЦОД, что и producer/consumer
5. **Unlimited retention** — нужны месяцы/годы хранения (managed обычно лимитируют)

**Когда managed лучше:**
1. Команда <5 человек, нет выделенного Kafka-admin
2. Трафик непредсказуемый (нужен auto-scaling, который managed дают из коробки)
3. Time-to-market критичен (развернуть managed-кластер = 5 минут против дней на self-hosted)
4. Нет компетенций в Linux/Kubernetes/JVM-тюнинге

### Стоимость self-hosted vs managed (2026, примерные цифры)

Сравнение для кластера из 3 брокеров, 100 партиций, throughput 50 MB/s:

| Статья расходов | Self-hosted | Confluent Cloud | AWS MSK |
|----------------|-------------|-----------------|---------|
| Инфраструктура (compute) | $500–800/мес (3× EC2 m5.2xlarge) | Включено | $600/мес |
| Storage (EBS 500 GB) | $150/мес | Включено | $150/мес |
| DevOps (FTE, часть времени) | $2,000–4,000/мес | $0 | $500/мес (проще) |
| Лицензия / платформа | $0 | $1,500–3,000/мес | $0 (плата за брокер) |
| **Итого/мес** | **$2,650–4,950** | **$1,500–3,000** | **$1,250** |

**Ключевой инсайт:** главная статья расходов self-hosted — не железо, а люди. Если у вас уже есть команда, которая обслуживает инфраструктуру, добавление Kafka может быть почти бесплатным. Если команды нет — managed почти всегда дешевле.

---

## Лицензионная карта: Apache 2.0 vs SSPL vs BSL

Это критический раздел для тех, кто принимает архитектурные решения о том, на каких open-source компонентах строить платформу. Лицензия определяет не только то, что вы можете делать сегодня, но и то, сможете ли вы делать это завтра.

### Apache License 2.0 — золотой стандарт

**Что разрешает (без ограничений):**

| Действие | Apache 2.0 | SSPL | BSL | ELv2 |
|----------|-----------|------|-----|------|
| Использование в production | ✅ | ✅ | ✅ | ✅ |
| Модификация кода | ✅ | ✅ | ✅ | ✅ |
| Распространение | ✅ | ✅ | ✅ | ✅ |
| SaaS / Managed Service | ✅ | ❌ | ❌ | ❌ |
| Коммерческое использование | ✅ | ✅ (кроме SaaS) | ✅ (кроме SaaS) | ✅ (кроме SaaS) |
| Сублицензирование | ✅ | ❌ | ✅ | ❌ |
| Патентная защита | ✅ | ❌ | ❌ | ❌ |

**Ключевое преимущество Apache 2.0 для бизнеса:** полная свобода действий. Вы можете взять Apache Kafka, доработать, упаковать в Docker, продавать как managed service — и никто не придёт к вам с юридическими претензиями.

### SSPL (Server Side Public License)

**Происхождение:** создана MongoDB Inc. в 2018 году в ответ на то, что AWS продавала MongoDB как managed service, не платя MongoDB Inc. ничего.

**Суть ограничения:** если вы предлагаете софт под SSPL как сервис третьим лицам, вы обязаны открыть исходный код **всего сопутствующего софта**, который используете для предоставления этого сервиса (мониторинг, оркестрация, UI, конфигурация).

**Почему SSPL не одобрена OSI:** Open Source Initiative не признаёт SSPL открытой лицензией, потому что требование открывать код «всего сопутствующего софта» слишком широкое и непрактичное.

**Примеры SSPL в Kafka-экосистеме:** прямых SSPL-проектов в Kafka-мире мало, но тренд движется в эту сторону у смежных проектов (Elasticsearch перешёл на SSPL → форк OpenSearch под Apache 2.0).

### BSL (Business Source License)

**Автор:** компания MariaDB (позже принята Cockroach Labs, Redpanda, WarpStream/Confluent).

**Суть:** исходный код доступен для чтения и модификации, бесплатен для некоммерческого использования и использования внутри компании, но запрещает продажу как сервис. Через N лет (обычно 3–4) каждая версия переходит под Apache 2.0.

**BSL в Kafka-экосистеме:**

| Проект | BSL-период | Примечание |
|--------|-----------|------------|
| Redpanda | 4 года | Каждая версия → Apache 2.0 через 4 года после релиза |
| WarpStream | Неопределён | Поглощён Confluent в 2024 |

**Ловушка BSL для стартапов:** вы можете спокойно использовать Redpanda внутри компании 3 года, построить на нём продукт, а потом решить продавать его как SaaS. В этот момент BSL-ограничение срабатывает — и вы должны либо платить Redpanda за коммерческую лицензию, либо мигрировать на Apache Kafka. Миграция потоковой платформы «на живую» — одна из самых болезненных операций в инженерии данных.

### ELv2 (Elastic License v2)

**Автор:** Elastic (создатели Elasticsearch).

**Суть:** похожа на BSL, но мягче — разрешает использование в production и модификацию, запрещает SaaS-перепродажу, но не имеет автоматического перехода на Apache 2.0.

**Примеры в Kafka-экосистеме:**
- AutoMQ enterprise features
- Confluent connectors (частично)

### Confluent Community License — история, которая всё изменила

В декабре 2018 года Confluent изменил лицензию части компонентов Confluent Platform с Apache 2.0 на Confluent Community License. Это была реакция на запуск Amazon MSK (Managed Streaming for Kafka).

**Компоненты, переведённые с Apache 2.0 на CCL:**
- Confluent REST Proxy
- Confluent Schema Registry
- KSQL (позже ksqlDB)
- Confluent connectors (JDBC, S3, Elasticsearch, HDFS)

**Суть CCL:** запрещает продавать софт как managed service конкурирующим облачным провайдерам. Вы можете использовать его внутри компании бесплатно, но AWS/Azure/GCP не могут взять Confluent Schema Registry и продавать как часть своего managed Kafka.

**Последствия:**
1. **AWS MSK** не включает Schema Registry «из коробки» — клиенты должны ставить свой
2. **Aiven** создал открытые альтернативы (Karapace = Schema Registry + REST Proxy под Apache 2.0)
3. **Apache Apicurio** (Red Hat) стал основной открытой альтернативой Schema Registry
4. **Кафка-экосистема разделилась** на «confluent-track» и «open-track»

### Практическая матрица выбора лицензии

**Задача:** выбрать компоненты для постройки self-hosted Kafka-платформы, которую вы потенциально будете продавать как SaaS.

| Компонент | Open-source решение | Лицензия | SaaS-safe |
|-----------|-------------------|----------|-----------|
| Брокер Kafka | Apache Kafka | Apache 2.0 | ✅ |
| Schema Registry | Apicurio Registry | Apache 2.0 | ✅ |
| REST Proxy | Karapace REST | Apache 2.0 | ✅ |
| CDC | Debezium | Apache 2.0 | ✅ |
| Оркестрация | Strimzi | Apache 2.0 | ✅ |
| GUI | AKHQ / Kafka UI | Apache 2.0 | ✅ |
| Мониторинг | Prometheus + Burrow | Apache 2.0 | ✅ |
| Governance | Klaw | Apache 2.0 | ✅ |
| Stream Processing | Kafka Streams (ядро Kafka) | Apache 2.0 | ✅ |
| SQL Streaming | ❌ ksqlDB — CCL | ⚠️ Нет полного аналога под Apache 2.0 | ❌ |

**Главный пробел в open-source стеке (2026):** ksqlDB остаётся под Confluent Community License, и полноценного SQL-движка для потоковой обработки под Apache 2.0 нет. Apache Flink SQL — ближайшая альтернатива, но это отдельный тяжёлый проект, а не лёгкая надстройка над Kafka.

---

## Практический чек-лист: выбор open-source стека

### Сценарий 1: «Разработка и тестирование»

**Цель:** локально запустить Kafka для разработки микросервисов.

**Рекомендованный стек (полностью бесплатный, Apache 2.0):**
1. **Apache Kafka** через Docker Compose (KRaft mode, без ZooKeeper)
2. **Kafka UI** (provectus) — просмотр топиков и сообщений
3. **kcat** — отладка из командной строки

```bash
# docker-compose.yml для разработки
version: '3'
services:
  kafka:
    image: apache/kafka:4.2.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_LISTENERS: 'PLAINTEXT://:9092,CONTROLLER://:9093'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:9093'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

Время развёртывания: 2 минуты.

### Сценарий 2: «Production, on-premise, Kubernetes»

**Цель:** production-кластер на своём Kubernetes, без вендорских лицензий.

**Рекомендованный стек (все Apache 2.0):**
1. **Strimzi Operator** — оркестрация Kafka на K8s
2. **Apache Kafka 4.2** (KRaft mode) — брокеры
3. **Cruise Control** — автоматическая ребалансировка
4. **Prometheus + Grafana** — мониторинг
5. **AKHQ** — GUI для операторов
6. **Klaw** — governance (если много команд)
7. **Debezium** — CDC (если нужна интеграция с БД)

### Сценарий 3: «Managed Service Provider»

**Цель:** построить Kafka-as-a-Service и продавать клиентам.

**Рекомендованный стек (SaaS-safe, все Apache 2.0):**
1. **Apache Kafka** — ядро (можно модифицировать)
2. **Apicurio Registry** — Schema Registry
3. **Karapace REST** — REST Proxy
4. **Strimzi** — оркестрация (или своя обёртка)
5. **Debezium** — CDC-коннекторы для клиентов
6. **Prometheus/Grafana** — мониторинг
7. **Ключевое правило** — не брать компоненты под CCL, BSL, SSPL, ELv2

**Антипример:** AWS MSK использует Apache Kafka (можно), но не включает Schema Registry (потому что официальный Confluent SR под CCL — нельзя для AWS). Клиенты вынуждены либо платить Confluent отдельно, либо ставить Apicurio.

### Сценарий 4: «Крупный enterprise с существующим ZooKeeper»

**Цель:** мигрировать с устаревающей ZK-based архитектуры на современный open-source стек.

**Стратегия:**
1. **Kafka 3.9** — последняя версия с dual-mode (ZK + KRaft bridge)
2. Миграция на **Kafka 4.0+** с KRaft-only
3. Замена Confluent Schema Registry на **Apicurio Registry**
4. Замена Confluent REST Proxy на **Karapace REST**
5. Установка **Strimzi** для унификации управления

---

## Заключение: дух open-source жив, но требует бдительности

Apache Kafka — один из немногих крупных инфраструктурных проектов, который сохранил лицензию Apache 2.0 на ядро, несмотря на давление коммерциализации. Это заслуга Apache Software Foundation как нейтрального «дома» и сообщества, которое не дало Confluent увести проект в сторону закрытых лицензий.

**Три главных вывода:**

1. **Open-source Kafka-стек существует и он полный** — от брокера до CDC до governance. Нет ни одной критической функции, для которой нужна коммерческая лицензия. Единственный заметный пробел — SQL-стриминг (ksqlDB), но он закрывается Flink SQL.

2. **Главный риск — не технологии, а лицензии.** BSL и SSPL выглядят как «почти open-source», но блокируют целые бизнес-модели. Если вы планируете продавать managed Kafka — проверяйте лицензию каждого компонента.

3. **KRaft убрал последнее «но» для self-hosted Kafka.** Раньше администрирование ZooKeeper было основным аргументом в пользу managed-решений. С Kafka 4.0 этого аргумента больше нет — self-hosted кластер на 3 ноды админится через те же YAML-манифесты Strimzi, что и любой другой Kubernetes-сервис.

**Финальный чек-лист для принимающих решение:**
- [ ] Ядро системы на Apache Kafka (Apache 2.0) ✅
- [ ] Schema Registry на Apicurio (Apache 2.0), не на Confluent SR (CCL) ✅
- [ ] Оркестрация на Strimzi (Apache 2.0, CNCF Incubating) ✅
- [ ] GUI управления на AKHQ/Kafka UI (Apache 2.0) ✅
- [ ] CDC на Debezium (Apache 2.0) ✅
- [ ] Мониторинг на Prometheus + Grafana + Burrow (Apache 2.0) ✅
- [ ] Не попался на BSL-ловушку Redpanda/WarpStream для SaaS ❌

---

## Источники

1. **Apache Software Foundation** — Apache Kafka official documentation, ecosystem page. https://kafka.apache.org/
2. **GitHub: apache/kafka** — основной репозиторий, лицензия Apache 2.0. https://github.com/apache/kafka
3. **Apache Kafka 4.0 Release Announcement** (2025-03-18) — David Jacot, описание KIP-966 Tiered Storage, KRaft GA. https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/
4. **GitHub: strimzi/strimzi-kafka-operator** — CNCF Incubating проект, Apache 2.0. https://github.com/strimzi/strimzi-kafka-operator
5. **CNCF: Strimzi Project** — статус Incubating с 2024-02-08. https://www.cncf.io/projects/strimzi/
6. **Strimzi Official Site** — документация, quick starts, архитектура. https://strimzi.io/
7. **GitHub: debezium/debezium** — CDC-проект, 12.6K звёзд, Apache 2.0. https://github.com/debezium/debezium
8. **Debezium License Page** — подтверждение Apache 2.0. https://debezium.io/license
9. **GitHub: linkedin/cruise-control** — LinkedIn, автоматическая ребалансировка, Apache 2.0. https://github.com/linkedin/cruise-control
10. **GitHub: provectus/kafka-ui** — Web UI, 12K звёзд, Apache 2.0. https://github.com/provectus/kafka-ui
11. **GitHub: kafbat/kafka-ui** — форк, TypeScript-first, Apache 2.0. https://github.com/kafbat/kafka-ui
12. **GitHub: tchiotludo/akhq** — AKHQ GUI, Apache 2.0. https://github.com/tchiotludo/kafkahq
13. **GitHub: Aiven-Open/klaw** — Kafka governance toolkit, Apache 2.0. https://github.com/Aiven-Open/klaw
14. **Aiven Blog** — Introducing Klaw for Apache Kafka Governance (2022-09-29). https://aiven.io/blog/introducing-klaw-for-apache-kafka-governance
15. **Confluent** — Confluent Community License FAQ. https://www.confluent.io/confluent-community-license-faq/
16. **Business Insider** — «Confluent Community License Created After Amazon Web Services Starts Selling Kafka» (2018-12-15). https://www.businessinsider.com/confluent-community-license-created-after-amazon-web-services-starts-selling-kafka-2018-12
17. **GitHub: obsidiandynamics/kafdrop** — Kafdrop UI, Apache 2.0. https://github.com/obsidiandynamics/kafdrop
18. **GitHub: kcctl/kcctl** — современный CLI для Kafka Connect, Apache 2.0. https://github.com/kcctl/kcctl
19. **AutoMQ Blog** — Self-Hosted Kafka vs. Fully Managed Kafka: Pros & Cons (2025-04-21). https://automq.com/blog/self-hosted-kafka-vs-fully-managed-kafka-pros-amp-cons
20. **AutoMQ Blog** — Top Open-Source Diskless Kafka Alternatives in 2026 (2026-04-28). https://www.automq.com/blog/top-open-source-diskless-kafka-alternatives
21. **AutoMQ Wiki** — AutoMQ vs Other Streaming Platforms (2025-01-17). https://github.com/AutoMQ/automq/wiki/AutoMQ-vs-Other-Streaming-Platforms
22. **KafScale** — Stateless Kafka on S3, Apache 2.0, сравнение альтернатив. https://kafscale.io/comparison/
23. **Apache Software Foundation** — Kafka Improvement Proposals (KIP). https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals
24. **Apache Software Foundation** — Kafka Project Bylaws. https://cwiki.apache.org/confluence/display/KAFKA/Bylaws
25. **Apache Kafka** — Geo-Replication (MirrorMaker 2), официальная документация. https://kafka.apache.org/28/operations/geo-replication-cross-cluster-data-mirroring/
26. **GitHub: strimzi/mirror-maker-2-extensions** — расширения MirrorMaker 2, Apache 2.0. https://github.com/strimzi/mirror-maker-2-extensions
