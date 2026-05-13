# 07-операции.02 — Логирование в Apache Kafka

**Нижняя строка:** логирование в Kafka — это не просто «чтобы было». Каждый из шести специализированных логов (server, controller, state-change, kafka-request, authorizer, log-cleaner) решает конкретную операционную задачу, а перевод на структурированный JSON с агрегацией в ELK/Loki/OpenSearch — это минимальный порог для production-ready кластера, аудита и compliance.

---

## 1. Архитектура логирования Kafka

**Ключевая мысль:** Kafka логирует не «всё в один файл», а разделяет потоки по назначению — каждый лог-файл обслуживает конкретную аудиторию (операторы, разработчики, безопасники).

### 1.1 Фреймворк логирования

Kafka использует **SLF4J** как фасад логирования и **Apache Log4j2** в качестве реализации (начиная с версии 2.5, когда был выполнен переход с Log4j 1.x — см. KIP-653). Конфигурация по умолчанию лежит в `config/log4j2.yaml` и задаёт:

- **6 файловых appender'ов** (RollingFile) с ротацией по часам (`%d{yyyy-MM-dd-HH}`)
- **1 консольный appender** (STDOUT)
- **7 именованных logger'ов**, каждый со своим additivity=false (запрет всплытия в родительский логгер)

**Аналогия:** Представьте, что Kafka-брокер — это офисное здание. У каждого отдела своя картотека: бухгалтерия ведёт финансовый журнал (server.log), служба безопасности — журнал посетителей (authorizer.log), диспетчерская — журнал движения лифтов (state-change.log). Если свалить всё в одну папку — найти нужное будет невозможно.

### 1.2 Загрузка конфигурации

Kafka определяет путь к конфигурации Log4j2 через переменную окружения `KAFKA_LOG4J_OPTS` в стартовом скрипте `kafka-server-start.sh`:

```bash
# Указание внешнего конфига (начиная с Kafka 2.5+)
export KAFKA_LOG4J_OPTS="-Dlog4j.configurationFile=/path/to/custom-log4j2.yaml"

# Для компонентов Confluent Platform — свои переменные:
export KAFKA_CONNECT_LOG4J_OPTS="-Dlog4j.configurationFile=.../connect-log4j2.yaml"
export KAFKA_STREAMS_LOG4J_OPTS="-Dlog4j.configurationFile=.../streams-log4j2.yaml"
```

**Важно:** формат конфигурации Log4j2 поддерживает YAML, XML, JSON и Properties. Kafka по умолчанию поставляет YAML, но в production чаще используют XML из-за совместимости с инструментами управления конфигурацией (Ansible, Puppet) и лучшей поддержки валидации.

---

## 2. Форматы логов: 6 специализированных файлов

**Ключевая мысль:** каждый лог-файл Kafka закрывает конкретный сценарий, и оператор должен знать, куда смотреть при каждой типовой проблеме.

### 2.1 server.log — основной лог брокера

**Назначение:** «всё, что не попало в специализированные логи». Стартап/шатдаун брокера, общие ошибки, предупреждения о конфигурации.

**Формат записи (по умолчанию PatternLayout):**
```
[2026-05-13 10:15:23,456] INFO Kafka version: 4.1.0 (org.apache.kafka.common.utils.AppInfoParser)
[2026-05-13 10:15:23,789] INFO Kafka commitId: 77a89fcf8d7fa018 (org.apache.kafka.common.utils.AppInfoParser)
[2026-05-13 10:15:25,123] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)
```

**Когда смотреть:**
- Брокер не стартует → ищем `Fatal error during KafkaServer startup`
- Проблемы с сетью → `SocketServer` ошибки
- Предупреждения о неоптимальных настройках ОС (file descriptors, swappiness)

### 2.2 controller.log — логи контроллера кластера

**Назначение:** все действия активного контроллера — выборы лидеров партиций, управление ISR (In-Sync Replicas), перебалансировка реплик.

**Формат:**
```
[2026-05-13 10:16:01,234] INFO [Controller id=1] Processing broker 2 heartbeat (kafka.controller.KafkaController)
[2026-05-13 10:16:05,567] INFO [Controller id=1] New leader for partition orders-3 is 2 (kafka.controller.KafkaController)
[2026-05-13 10:16:10,890] INFO [Controller id=1] Shrinking ISR for partition payments-7 from [1,2,3] to [1,2] (kafka.controller.KafkaController)
```

**Когда смотреть:**
- Частые смены лидеров партиций (leader election storms)
- Проблемы с ISR (реплики выпадают/возвращаются)
- Нестабильность кластера при сетевых проблемах

**Ключевые паттерны для алертинга:**
- `Shrinking ISR` с частотой >10/мин — проблема с конкретным брокером или сетью
- `New leader` на одной партиции чаще чем раз в 5 минут — нестабильность

### 2.3 state-change.log — логи изменений состояния

**Назначение:** фиксация каждого изменения состояния партиций и реплик (онлайн/офлайн, лидер/фолловер).

**Формат:**
```
[2026-05-13 10:17:00,111] INFO [Partition orders-3 broker=1] Changed state for replica from OnlineReplica to OfflineReplica (state.change.logger)
[2026-05-13 10:17:00,222] INFO [Partition orders-3 broker=2] Changed leader from 1 to 2 (state.change.logger)
```

**Отличие от controller.log:** controller.log показывает процесс принятия решения (контроллер решил сменить лидера), state-change.log — сам факт изменения состояния на конкретном брокере. Как протокол собрания vs журнал дежурств — одна запись про обсуждение, другая про результат.

**Когда смотреть:**
- Расследование проблем доступности конкретной партиции
- Понимание хронологии отказов: «партиция была недоступна с 10:17 до 10:19, потому что реплика на брокере 1 ушла в офлайн»

### 2.4 kafka-request.log — логи обработки запросов

**Назначение:** детальная информация о каждом сетевом запросе к брокеру. По умолчанию уровень WARN — логируются только медленные запросы и ошибки.

**Формат (WARN-уровень, медленный запрос):**
```
[2026-05-13 10:18:05,456] WARN [RequestSendThread controllerId=1] Request of type=PRODUCE, correlationId=12345, clientId=app-producer-1 took 534 ms (kafka.request.logger)
```

**Уровни детализации:**
- **WARN** (умолчание) — только запросы дольше `request.timeout.ms` или с ошибками
- **INFO** — все запросы с указанием времени обработки
- **TRACE** — полный дамп запросов/ответов (включая тело сообщения). **Осторожно:** на production-кластере с 10K req/s генерирует гигабайты логов в час

**Когда смотреть:**
- Расследование latency проблем
- Поиск «тяжёлых» клиентов (корреляция по correlationId)
- Отладка проблем сериализации/десериализации на уровне протокола

### 2.5 kafka-authorizer.log — логи авторизации

**Назначение:** запись всех решений ACL (Access Control List) — кто, к какому ресурсу, с какой операцией и был ли разрешён доступ.

**Формат:**
```
[2026-02-03 14:23:15,678] INFO Principal = User:unauthorized-app is Denied Operation = Read from host = 10.0.1.45 on resource = Topic:LITERAL:payments (kafka.authorizer.logger)
[2026-02-03 14:23:16,001] DEBUG Principal = User:consumer-app is Allowed Operation = Read from host = 10.0.1.42 on resource = Topic:LITERAL:orders (kafka.authorizer.logger)
```

**Особенности:**
- Отказы (`Denied`) логируются на уровне **INFO**
- Разрешения (`Allowed`) — на уровне **DEBUG** (требует явного повышения уровня для аудита)
- Логгер пишет 2 строки на каждую операцию — при 10 000 запросов/с это ~20 тысяч строк/с

**Когда смотреть:**
- Аудит доступа: «кто читал топик payments за последние 90 дней?»
- Расследование инцидентов безопасности
- Compliance: SOC2, HIPAA, PCI-DSS, GDPR (см. раздел 7)

### 2.6 log-cleaner.log — логи очистки логов

**Назначение:** работа механизма Log Compaction — какие ключи были удалены, прогресс очистки, ошибки.

**Формат:**
```
[2026-05-13 10:20:00,345] INFO [LogCleaner-1] Starting log cleaning for partition orders-3 (org.apache.kafka.storage.internals.log.LogCleaner$CleanerThread)
[2026-05-13 10:20:02,567] INFO [LogCleaner-1] Log cleaning for partition orders-3 completed. 1,234 messages compacted (org.apache.kafka.storage.internals.log.LogCleaner$CleanerThread)
```

**Когда смотреть:**
- Log Compaction отстаёт (dirty ratio растёт)
- Ошибки при чтении/записи сегментов во время compaction

---

## 3. Структурированное логирование: JSON и Log4j2

**Ключевая мысль:** `[%d] %p %m (%c)%n` — это формат для человека, а не для машины. Для production-логов нужен JSON.

### 3.1 Почему JSON?

| Аспект | Pattern Layout (текст) | JSON-логи |
|--------|----------------------|-----------|
| Парсинг в агрегаторе | regex/grok (хрупко, медленно) | нативный JSON-парсер |
| Добавление полей | изменение pattern во всех файлах | новое поле в шаблоне |
| Типизация полей | нет (всё строки) | числа, boolean, вложенные объекты |
| Поиск по полям | full-text scan | индексированный поиск |
| Совместимость с SIEM | требует кастомных парсеров | «из коробки» |

**Аналогия:** Pattern-логи — это рукописный текст в блокноте, JSON-логи — заполненная форма с галочками, которую сканер обрабатывает мгновенно.

### 3.2 JsonTemplateLayout — современный подход

Log4j2 предоставляет **JsonTemplateLayout** (начиная с версии 2.14) — garbage-free, кастомизируемый JSON-генератор. В отличие от устаревшего JsonLayout, JsonTemplateLayout позволяет описать структуру JSON-документа через **шаблон** (JSON-файл с `$resolver`-ами).

**Подключение зависимости (Maven):**
```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-layout-template-json</artifactId>
    <version>2.25.3</version>
</dependency>
```

### 3.3 Предустановленные шаблоны ECS

Log4j2 поставляет готовые шаблоны (лежат в classpath библиотеки `log4j-layout-template-json`):

| Шаблон | Назначение |
|--------|-----------|
| `EcsLayout.json` | Elastic Common Schema (ECS) — стандарт Elastic |
| `LogstashJsonEventLayoutV1.json` | Совместимость с Logstash json_event |
| `GelfLayout.json` | Graylog Extended Log Format (GELF) |
| `GcpLayout.json` | Google Cloud structured logging |

### 3.4 Конфигурация Kafka на JSON-логирование

**Пример log4j2.yaml для Kafka с ECS-форматом:**

```yaml
Configuration:
  Properties:
    Property:
      - name: "kafka.logs.dir"
        value: "/var/log/kafka"
      - name: "logPattern"
        value: "[%d] %p %m (%c)%n"

  Appenders:
    # Консоль — для человека, pattern
    Console:
      name: STDOUT
      PatternLayout:
        pattern: "${logPattern}"

    # JSON-аппендер для агрегаторов
    RollingFile:
      - name: KafkaJsonAppender
        fileName: "${sys:kafka.logs.dir}/server.json"
        filePattern: "${sys:kafka.logs.dir}/server.json.%d{yyyy-MM-dd-HH}"
        JsonTemplateLayout:
          eventTemplateUri: "classpath:EcsLayout.json"
          # Дополнительные поля для каждого события
          EventTemplateAdditionalField:
            - key: "service.name"
              value: "kafka-broker"
            - key: "service.environment"
              value: "production"
            - key: "host.name"
              format: "${env:HOSTNAME}"
        TimeBasedTriggeringPolicy:
          modulate: true
          interval: 1

      # State Change — тоже JSON
      - name: StateChangeJsonAppender
        fileName: "${sys:kafka.logs.dir}/state-change.json"
        filePattern: "${sys:kafka.logs.dir}/state-change.json.%d{yyyy-MM-dd-HH}"
        JsonTemplateLayout:
          eventTemplateUri: "classpath:EcsLayout.json"
          EventTemplateAdditionalField:
            - key: "log.type"
              value: "state-change"
        TimeBasedTriggeringPolicy:
          modulate: true
          interval: 1

      # Controller — JSON
      - name: ControllerJsonAppender
        fileName: "${sys:kafka.logs.dir}/controller.json"
        filePattern: "${sys:kafka.logs.dir}/controller.json.%d{yyyy-MM-dd-HH}"
        JsonTemplateLayout:
          eventTemplateUri: "classpath:EcsLayout.json"
          EventTemplateAdditionalField:
            - key: "log.type"
              value: "controller"
        TimeBasedTriggeringPolicy:
          modulate: true
          interval: 1

      # Authorizer — JSON (важно для аудита!)
      - name: AuthorizerJsonAppender
        fileName: "${sys:kafka.logs.dir}/authorizer.json"
        filePattern: "${sys:kafka.logs.dir}/authorizer.json.%d{yyyy-MM-dd-HH}"
        JsonTemplateLayout:
          eventTemplateUri: "classpath:EcsLayout.json"
          EventTemplateAdditionalField:
            - key: "log.type"
              value: "authorizer"
        TimeBasedTriggeringPolicy:
          modulate: true
          interval: 1

  Loggers:
    Root:
      level: INFO
      AppenderRef:
        - ref: STDOUT        # человекочитаемый
        - ref: KafkaJsonAppender  # машиночитаемый

    Logger:
      - name: state.change.logger
        level: INFO
        additivity: false
        AppenderRef:
          ref: StateChangeJsonAppender

      - name: org.apache.kafka.controller
        level: INFO
        additivity: false
        AppenderRef:
          ref: ControllerJsonAppender

      - name: kafka.authorizer.logger
        level: INFO        # Минимум INFO для аудита всех действий
        additivity: false
        AppenderRef:
          ref: AuthorizerJsonAppender
```

**Ключевой момент:** `additivity: false` гарантирует, что события из специализированных логгеров не дублируются в `server.json`. Без этого — двойная запись и путаница в агрегаторах.

### 3.5 Кастомный JSON-шаблон для Kafka

Если ECS — избыточно или не подходит, можно создать свой шаблон:

```json
{
  "timestamp": {
    "$resolver": "timestamp",
    "pattern": {
      "format": "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'",
      "timeZone": "UTC"
    }
  },
  "level": {
    "$resolver": "level",
    "field": "name"
  },
  "logger": {
    "$resolver": "logger",
    "field": "name"
  },
  "thread": {
    "$resolver": "thread",
    "field": "name"
  },
  "message": {
    "$resolver": "message",
    "stringified": true
  },
  "cluster": "${env:KAFKA_CLUSTER_NAME:-default}",
  "broker_id": "${env:KAFKA_BROKER_ID:-unknown}"
}
```

**Совет:** при использовании Elasticsearch избегайте точек в именах custom-полей — точка интерпретируется как вложенный объект, что может вызвать mapping conflict (поле `foo.bar` и `foo: {bar: ...}` конфликтуют).

---

## 4. Управление уровнями логирования

**Ключевая мысль:** в production уровни должны быть минимальными (INFO для аудита, WARN/ERROR для остального), но возможность временно поднять уровень без рестарта — обязательна.

### 4.1 Динамическое изменение через kafka-configs.sh

Kafka поддерживает динамическое изменение уровней для брокеров, Connect и MirrorMaker 2 через Admin API (KIP-412):

```bash
# Установить DEBUG для брокера 1
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 1 \
  --alter --add-config "log4j.logger.kafka=DEBUG"

# Вернуть INFO
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 1 \
  --alter --add-config "log4j.logger.kafka=INFO"

# Просмотреть текущие настройки
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 1 --describe
```

**Предостережение:** DEBUG на busy-кластере (10K req/s) генерирует до 5-10 GB логов в час. Всегда ставьте автоматический откат через cron или таймер.

### 4.2 Рекомендуемые уровни для production

| Logger | Production | Troubleshooting | Аудит |
|--------|-----------|----------------|-------|
| Root (`kafka`) | INFO | DEBUG | INFO |
| `kafka.request.logger` | WARN | INFO/TRACE | WARN |
| `kafka.authorizer.logger` | INFO | DEBUG | **INFO** |
| `state.change.logger` | INFO | DEBUG | INFO |
| `org.apache.kafka.controller` | INFO | DEBUG | INFO |
| `kafka.network.Processor` | WARN | TRACE | WARN |

**Важно:** Для аудита `kafka.authorizer.logger` должен быть на **INFO** (не DEBUG, не WARN). На DEBUG — разрешённые операции тоже пишутся (нужно для расследований, но удваивает объём), на WARN — остаются только отказы (недостаточно для compliance).

---

## 5. Ротация и управление дисковым пространством

**Ключевая мысль:** логи Kafka — второй по величине потребитель диска после данных топиков. Без политики ротации и очистки диск заполнится в течение часов.

### 5.1 Стратегия ротации по умолчанию

Kafka использует `TimeBasedTriggeringPolicy` с интервалом в 1 час (`interval: 1`, `modulate: true`). Это значит:

- Ротация каждый час на границе часа (modulate=true)
- `modulate: true` = файл начинается ровно в 00:00, 01:00, а не через час после старта (предсказуемо)
- Старый файл переименовывается по маске: `server.log.2026-05-13-10`

### 5.2 Оценка объёмов

| Сценарий | Запросов/с | ~Объём логов/час | ~Объём/день |
|----------|-----------|-----------------|------------|
| Dev-кластер, INFO | 100 | 50 MB | 1.2 GB |
| Prod-кластер, INFO | 5 000 | 500 MB | 12 GB |
| Prod-кластер, INFO (с аудитом) | 10 000 | 2 GB | 48 GB |
| Prod-кластер, DEBUG | 10 000 | 10 GB | 240 GB |

**Формула для оценки:** `объём ≈ запросы/с × 200 байт × 3600 × уровень_логирования_множитель`, где `уровень_множитель` = 1× для WARN, 5× для INFO, 50× для DEBUG.

### 5.3 Политика очистки

В отличие от данных топиков, Log4j2 **не удаляет** старые логи автоматически. Необходим внешний механизм:

```bash
# Пример: cron-задача для удаления логов старше 7 дней
# (в production — через logrotate или systemd timer)
find /var/log/kafka -name "*.log.*" -mtime +7 -delete
```

**Рекомендация:** используйте `logrotate` (Linux) или эквивалент. Не полагайтесь на ручную очистку.

---

## 6. Агрегация логов: ELK, Loki, OpenSearch

**Ключевая мысль:** логи на диске брокера бесполезны для поиска и корреляции. Они должны попасть в централизованное хранилище с индексацией.

### 6.1 Общая архитектура агрегации

```
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌───────────┐
│ Kafka    │───▶│ Filebeat │───▶│ Logstash /    │───▶│ Хранилище │
│ Broker   │    │ (shipper)│    │ Fluentd       │    │ (ES/OS)   │
│ JSON-лог │    └──────────┘    │ (processor)   │    └─────┬─────┘
└──────────┘                    └──────────────┘          │
                                                    ┌─────▼─────┐
                                                    │ Визуализ- │
                                                    │ ация      │
                                                    │ (Kibana/  │
                                                    │  Grafana) │
                                                    └───────────┘
```

### 6.2 ELK Stack (Elasticsearch + Logstash + Kibana)

**Самый распространённый вариант.** Filebeat читает JSON-файлы Kafka и отправляет напрямую в Elasticsearch (минуя Logstash для снижения задержки).

**Filebeat конфигурация для Kafka-логов:**

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/kafka/server.json
      - /var/log/kafka/controller.json
      - /var/log/kafka/state-change.json
    json.keys_under_root: true
    json.overwrite_keys: true
    fields:
      log_source: kafka_broker
      cluster: production-us-east
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/kafka/authorizer.json
    json.keys_under_root: true
    json.overwrite_keys: true
    fields:
      log_source: kafka_authorizer
      cluster: production-us-east
    fields_under_root: true

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "${ES_USERNAME}"
  password: "${ES_PASSWORD}"
  index: "kafka-logs-%{+yyyy.MM.dd}"
  # ILM (Index Lifecycle Management) для автоматического удаления
  ilm:
    enabled: true
    policy_name: "kafka-logs-30d-retention"

# Шаблон индекса для правильного mapping-а
setup.template.name: "kafka-logs"
setup.template.pattern: "kafka-logs-*"
setup.template.settings:
  index.number_of_shards: 3
  index.number_of_replicas: 1
```

**Плюсы ELK:**
- Гибкие запросы (KQL, Lucene)
- Зрелая экосистема (Machine Learning для anomaly detection, Watcher для алертинга)
- ECS-совместимость из коробки

**Минусы:**
- Высокое потребление ресурсов (Java heap, IOPS)
- Сложность кластеризации
- Лицензионные ограничения (Elastic License 2.0 / SSPL)

### 6.3 Grafana Loki

**Лёгкая альтернатива** — индексирует только метаданные (labels), а сами логи хранит в object storage (S3, MinIO).

**Promtail конфигурация для Kafka:**

```yaml
server:
  http_listen_port: 9080

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kafka_broker_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: kafka
          cluster: production-us-east
          __path__: /var/log/kafka/server.json
    pipeline_stages:
      - json:
          expressions:
            level: level
            logger: logger
            message: message
            timestamp: "@timestamp"
      - labels:
          level:
          logger:
      - timestamp:
          source: timestamp
          format: RFC3339

  - job_name: kafka_authorizer_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: kafka_authorizer
          cluster: production-us-east
          __path__: /var/log/kafka/authorizer.json
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            timestamp: "@timestamp"
      - labels:
          level:
      - timestamp:
          source: timestamp
          format: RFC3339
```

**Плюсы Loki:**
- Низкое потребление ресурсов (нет индексации содержимого)
- Горизонтальное масштабирование через object storage
- Нативная интеграция с Grafana (единая панель с метриками Prometheus)

**Минусы:**
- Ограниченные поисковые возможности (только full-text scan по содержимому)
- Меньше enterprise-фич (нет ML, сложных алертов)
- LogQL проще KQL, но менее выразителен для сложных запросов

### 6.4 OpenSearch

**Open-source форк Elasticsearch 7.10** после смены лицензии Elastic. Полностью совместим с Filebeat (до версии 7.10+), но требует Data Prepper вместо Logstash для новых инсталляций.

**Конфигурация Data Prepper (pipeline.yaml):**

```yaml
log-pipeline:
  source:
    http:
      path: "/log/ingest"
  processor:
    - grok:
        match:
          message: ['%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}']
    - date:
        from_time_received: true
        destination: "@timestamp"
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        index: "kafka-logs-%{yyyy.MM.dd}"
        dlq:
          s3:
            bucket: "dlq-bucket"
            region: "us-east-1"
            key_path_prefix: "dlq/kafka-logs/"
```

**Когда выбирать OpenSearch:**
- Юридические ограничения на Elastic License / SSPL
- Необходимость полного open-source стека
- Интеграция с AWS (Amazon OpenSearch Service)

### 6.5 Матрица выбора решения

| Критерий | ELK | Loki | OpenSearch |
|----------|-----|------|------------|
| Сложность запросов | ★★★★★ | ★★☆ | ★★★★☆ |
| Потребление ресурсов | Высокое | Низкое | Высокое |
| Простота установки | ★★★ | ★★★★★ | ★★★☆ |
| Алертинг | ★★★★★ | ★★★☆ | ★★★☆☆ |
| Лицензия | Elastic/SSPL | AGPLv3 | Apache 2.0 |
| Интеграция с Grafana | Через плагин | Нативная | Через плагин |
| Retention > 30 дней | ILM + snapshots | Object storage (дёшево) | ISM + snapshots |

**Быстрое правило:**
- Уже есть ELK? Оставайтесь на ELK.
- Используете Grafana/Prometheus? Берите Loki — единая панель для метрик и логов.
- Нужен 100% open-source? OpenSearch.
- Стартап/MVP? Loki (проще и дешевле).

---

## 7. Аудиторский след (Audit Trail)

**Ключевая мысль:** аудит — это не «включить все логи на DEBUG». Это целенаправленная конфигурация, отвечающая на вопрос «кто, что, когда сделал с данными?».

### 7.1 Что хотят видеть аудиторы

| Фреймворк | Требование к Kafka-логам | Срок хранения |
|-----------|------------------------|---------------|
| **SOC2** | Кто получал доступ, какие операции выполнял | 90 дней (рекомендуется 1 год) |
| **HIPAA** | Все доступы к электронной защищённой медицинской информации (ePHI) | 6 лет |
| **PCI-DSS** | Все доступы к среде данных держателей карт (CDE) | 1 год (3 месяца немедленно доступны) |
| **GDPR** | Цель обработки, согласие субъекта, сроки хранения, запросы на удаление | Соответствует цели обработки |

### 7.2 Настройка audit trail в Apache Kafka

**Шаг 1: Включить authorizer-логирование на уровне INFO:**

```yaml
# log4j2.yaml
Logger:
  - name: kafka.authorizer.logger
    level: INFO          # Все операции, включая разрешённые
    additivity: false
    AppenderRef:
      ref: AuthorizerJsonAppender  # Обязательно JSON для индексации
```

**Шаг 2: Добавить контекстные поля:**

```yaml
AuthorizerJsonAppender:
  JsonTemplateLayout:
    eventTemplateUri: "classpath:EcsLayout.json"
    EventTemplateAdditionalField:
      - key: "event.category"
        value: "authentication"
      - key: "event.type"
        value: "access"
      - key: "event.action"
        format: "${json:message:$.operation}"  # Извлекаем операцию из сообщения
```

**Шаг 3: Маркировать чувствительные топики:**

Соглашение об именовании топиков:
```
phi-*         → Protected Health Information (HIPAA)
pci-*         → Payment Card Industry data
pii-*         → Personally Identifiable Information (GDPR)
finance-*     → Financial data (SOX)
```

Это позволяет фильтровать аудит-логи: `resource = Topic:LITERAL:phi-*`.

**Шаг 4: Настроить сбор и хранение:**

```yaml
# Filebeat: отдельный pipeline для аудит-логов
filebeat.inputs:
  - type: log
    paths: /var/log/kafka/authorizer.json
    fields:
      log_type: audit
      retention_period: 6_years  # для HIPAA
    fields_under_root: true

output.elasticsearch:
  index: "kafka-audit-%{+yyyy.MM.dd}"
  ilm:
    policy_name: "kafka-audit-6y-retention"  # ILM-политика на 6 лет
```

### 7.3 Расследование инцидентов — типовые запросы

**Кто обращался к топику payments за последние 24 часа?**
```bash
grep "resource = Topic:LITERAL:payments" /var/log/kafka/kafka-authorizer.log | \
  grep -oP "Principal = \K[^,]+" | sort | uniq -c | sort -rn
```

**Какие операции выполнял подозрительный сервисный аккаунт?**
```bash
grep "Principal = User:suspicious-app" /var/log/kafka/kafka-authorizer.log | \
  grep -oP "Operation = \K\w+" | sort | uniq -c
```

**Все ли отказы доступа произошли за последний час?**
```bash
grep "Denied Operation" /var/log/kafka/kafka-authorizer.log
```

Для JSON-логов — те же запросы в Kibana/OpenSearch Dashboards через KQL:
```
log.type: "authorizer" AND message: "*Denied*" AND message: "*payments*"
```

### 7.4 Ограничения open-source Kafka для аудита

Честно о том, чего Apache Kafka **не умеет** без дополнительных инструментов:

- **Нет record-level аудита** — видно, что кто-то читал топик `payments`, но не видно, какие конкретно записи
- **Нет классификации данных** — все топики равны, только naming convention спасает
- **Нет real-time алертинга** — только post-hoc анализ (нужен SIEM/SOAR поверх)
- **Нет защиты от подделки** — локальные логи можно модифицировать (решается хранением в append-only external storage)

**Для regulated-индустрий рекомендуется:**
- **Confluent Audit Logs** (Confluent Server Authorizer) — структурированные аудит-события в топик `_confluent-audit-log`, routing по категориям (MANAGEMENT, AUTHENTICATE, AUTHORIZE, PRODUCE, CONSUME), retention через ILM
- **Conduktor Gateway** — прокси-слой с записью audit trail на уровне отдельных запросов
- **Schema Registry** — метаданные полей с классификацией (какие поля содержат PII)

---

## 8. Интеграция логирования с другими подсистемами

**Ключевая мысль:** логи Kafka не живут в вакууме — они должны коррелироваться с метриками, трейсами и алертами.

### 8.1 Корреляция логов и метрик

| Проблема | Что в логах | Что в метриках | Корреляция |
|----------|-----------|----------------|-----------|
| Slow requests | `kafka-request.log`: запросы > N ms | `TotalTimeMs` (P95/P99) | По correlationId |
| Consumer lag | `controller.log`: ISR change | `kafka_consumergroup_lag` | По consumer group + partition |
| Недоступность партиции | `state-change.log`: OfflineReplica | `UnderReplicatedPartitions` | По partition |
| Проблемы диска | `server.log`: IOException | `kafka_log_LogFlushStats` | По broker + partition |

**Совет:** добавьте `correlationId` или `traceId` в JSON-шаблон для сквозной трассировки запроса через все логи.

### 8.2 Интеграция с SIEM

**Wazuh (open-source):**
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/kafka/authorizer.json</location>
</localfile>
```

**Splunk:**
```
[monitor:///var/log/kafka/authorizer.json]
sourcetype = kafka:authorizer:json
index = kafka_audit
```

### 8.3 Интеграция с алертингом

**ElastAlert 2 (над Elasticsearch):**
```yaml
name: Kafka Authorization Denials
type: frequency
index: kafka-audit-*
num_events: 10
timeframe:
  minutes: 5
filter:
  - query:
      query_string:
        query: "message: *Denied*"
alert: "slack"
slack_webhook_url: "https://hooks.slack.com/services/..."
```

**Grafana Alerting (над Loki):**
```logql
# Алерт: более 50 отказов в доступе за 5 минут
sum(count_over_time({job="kafka_authorizer"} |~ "Denied Operation" [5m])) > 50
```

---

## 9. Best Practices и антипаттерны

### 9.1 Чеклист production-готовности логирования

- [ ] Все логи пишутся в JSON (JsonTemplateLayout + ECS)
- [ ] Специализированные логгеры имеют `additivity: false`
- [ ] `kafka.authorizer.logger` на уровне INFO (не DEBUG, не WARN)
- [ ] Ротация настроена: `TimeBasedTriggeringPolicy` с `interval: 1` и `modulate: true`
- [ ] Настроена внешняя очистка старых логов (logrotate / cron / ILM)
- [ ] Логи отправляются в централизованное хранилище (ELK/Loki/OpenSearch)
- [ ] Аудит-логи хранятся отдельным индексом с длительным retention
- [ ] Настроен алертинг на всплески отказов доступа
- [ ] Логи не пишутся на тот же диск, что и данные топиков (IO contention)
- [ ] Для чувствительных топиков используется naming convention (phi-*, pci-*, pii-*)
- [ ] Настроен `KAFKA_LOG4J_OPTS` для указания внешнего конфига (не редактирование bundled)
- [ ] Динамическое изменение уровней через `kafka-configs.sh` работает (KIP-412)

### 9.2 Топ-5 ошибок

**Ошибка 1: `additivity` не выставлен в false**
События из `controller.logger` попадают и в `controller.log`, и в `server.log`. При агрегации — дубликаты, двойной подсчёт, путаница. Решение: всегда `additivity: false` для специализированных логгеров.

**Ошибка 2: Логи в текстовом формате на production**
Grok-парсинг `[%d] %p %m (%c)%n` ломается при любом изменении формата сообщения (новый тип события — новый парсер). JSON решает проблему один раз и навсегда.

**Ошибка 3: DEBUG-level на production для аудита**
Пишет ВСЁ — разрешённые и запрещённые операции. 2 строки на каждый запрос. При 10K req/s — ~20K строк/с. Объём логов превышает объём данных топиков. Правильно: INFO для аудита (все операции), DEBUG — только для active troubleshooting.

**Ошибка 4: Логи на том же диске, что и данные топиков**
Kafka-брокер уже интенсивно использует диск (запись данных, page cache). Добавление логов на тот же диск крадёт IOPS и пропускную способность. Решение: отдельный mount point или диск для логов (`kafka.logs.dir`).

**Ошибка 5: Нет внешней очистки логов**
Log4j2 RollingFile только переименовывает старые файлы, но не удаляет их. Без `logrotate` или `find -mtime +N -delete` диск заполнится за дни. Решение: cron-задача каждые 24 часа.

---

## 10. Что изменилось в Kafka 3.8–4.1

- **Kafka 3.8:** KIP-653 завершён — полный переход на Log4j2, удаление поддержки Log4j 1.x. Конфигурация по умолчанию переведена на YAML (`log4j2.yaml`)
- **Kafka 3.9:** динамическое изменение уровней логирования для брокеров и Connect через Admin API (расширение KIP-412)
- **Kafka 4.0:** KRaft — controller.log и state-change.log теперь пишутся в рамках одного процесса (metadata quorum), логи стали более детализированными для операторов KRaft
- **Kafka 4.1:** улучшенная интеграция с OpenTelemetry — `correlationId` и `traceId` доступны в Log4j2 контексте через `ThreadContext` (MDC)

---

## Ссылки по теме

- [07-операции.01 — Мониторинг Kafka](./01-monitoring.md) — ключевые метрики, Prometheus, Grafana, алерты
- [06-производительность.04 — Диагностика узких мест](../06-performance/04-bottlenecks.md) — использование логов для поиска проблем
- [08-безопасность.01 — Механизмы безопасности](../08-security/01-security-features.md) — ACL, авторизация, RBAC
- [08-безопасность.02 — Харденинг Kafka](../08-security/02-hardening.md) — аудит как часть hardening

---

## Источники

1. **Apache Kafka — log4j2.yaml (trunk)** — официальный конфиг логирования. [GitHub](https://github.com/apache/kafka/blob/trunk/config/log4j2.yaml)
2. **KIP-653: Upgrade log4j to log4j2** — план перехода Kafka на Log4j2. [Apache Wiki](https://cwiki.apache.org/confluence/display/KAFKA/KIP-653%3A+Upgrade+log4j+to+log4j2)
3. **Sling Academy — How to interpret Kafka logs (with 8 examples)** — разбор всех типов логов с примерами. [slingacademy.com](https://www.slingacademy.com/article/how-to-interpret-kafka-logs-with-examples/)
4. **Conduktor Blog — Audit Logging in Kafka: Who Did What and When** — практическое руководство по аудит-логированию. [conduktor.io](https://www.conduktor.io/blog/kafka-audit-logging-compliance-forensics)
5. **Confluent Docs — Kafka Connect Logging** — конфигурация логирования для Kafka Connect. [docs.confluent.io](https://docs.confluent.io/platform/current/connect/logging.html)
6. **Apache Log4j2 — JSON Template Layout** — официальная документация JsonTemplateLayout. [logging.apache.org](https://logging.apache.org/log4j/2.x/manual/json-template-layout.html)
7. **Elastic — Structured logging with log4j2 (ECS)** — интеграция Log4j2 с Elastic Common Schema. [elastic.co](https://www.elastic.co/docs/reference/ecs/logging/java/_structured_logging_with_log4j2)
8. **Red Hat Streams for Apache Kafka — Configuring logging** — настройка уровней и динамическое изменение. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.0/html/using_streams_for_apache_kafka_on_rhel/assembly-kafka-logging-str)
9. **KLogic — Kafka Audit Logging & Compliance** — обзор compliance-требований к аудиту Kafka. [klogic.io](https://klogic.io/guides/kafka-audit-logging-compliance/)
10. **AutoMQ Blog — Kafka Logs: Concept & How It Works & Format** — архитектура логирования Kafka. [automq.com](https://www.automq.com/blog/kafka-logs-concept-how-it-works-format)
11. **Confluent Docs — Audit Log Concepts** — концепции аудит-логов Confluent Platform. [docs.confluent.io](https://docs.confluent.io/platform/current/security/compliance/audit-logs/audit-logs-concepts.html)
12. **Confluent Docs — Best Practices for Production Deployments** — рекомендации по логам state-change для production. [docs.confluent.io](https://docs.confluent.io/platform/current/kafka/post-deployment.html)
