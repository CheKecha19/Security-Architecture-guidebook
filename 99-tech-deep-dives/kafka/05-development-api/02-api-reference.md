# Справочник по API Apache Kafka: wire protocol, операции, аутентификация и коды ошибок

> **Нижняя строка:** API Kafka — это бинарный TCP-протокол с версионированием, где все операции (Produce, Fetch, ListOffsets, Metadata, Admin) упаковываются в request/response сообщения с фиксированной структурой заголовков. Аутентификация выполняется через SASL поверх TLS, а ошибки возвращаются в виде 16-битных кодов (более 120 стандартизированных значений). Понимание wire-уровня критично для написания кастомных клиентов, отладки сетевых проблем и performance-оптимизации.

---

## 1. Архитектура Kafka Wire Protocol

### 1.1 Транспортный уровень: TCP, бинарный, запрос-ответ

Kafka использует **собственный бинарный протокол поверх TCP**. В отличие от большинства современных распределённых систем, которые используют HTTP/REST (Elasticsearch) или gRPC/Protobuf (etcd), команда Kafka приняла осознанное решение реализовать свой протокол с нуля. Причины этого решения:

- **Максимальная эффективность:** бинарный формат без HTTP-overhead позволяет достичь пропускной способности, близкой к пропускной способности сети и диска.
- **Полный контроль над семантикой:** брокер может гарантировать ordering запросов на одном TCP-соединении, что критично для идемпотентного продюсера и транзакций.
- **Batching на уровне протокола:** один ProduceRequest может содержать сообщения для множества партиций разных топиков — HTTP/REST потребовал бы многократных overhead-ов.

**Аналогия:** если HTTP/REST — это почтовая служба (письмо с конвертом, адресом, обратным адресом, марками), то Kafka protocol — это грузовой конвейер (паллеты, packed в контейнеры, packed в фуры). Огромный выигрыш в throughput, но сложнее в отладке и требует специальных инструментов.

**Характеристики транспортного уровня:**

| Свойство | Значение |
|----------|----------|
| Протокол | TCP |
| Формат сообщений | Бинарный, собственный |
| Модель взаимодействия | Request-Response (клиент инициирует) |
| Порядок обработки | FIFO в пределах одного TCP-соединения |
| In-flight запросов | 1 на соединение (с точки зрения брокера) |
| Pipelining | Клиент может отправлять запросы через non-blocking I/O |
| Размер запроса | Ограничен `socket.request.max.bytes` (по умолчанию 100 МБ) |
| Постоянные соединения | Рекомендуются для амортизации TCP handshake |

**Важное замечание про ordering:** брокер гарантирует, что на одном TCP-соединении запросы обрабатываются **строго в порядке отправки**, и ответы возвращаются в том же порядке. Это достигается тем, что брокер обрабатывает только один in-flight запрос на соединение. Однако клиенты могут использовать **non-blocking I/O** для отправки следующего запроса до получения ответа на предыдущий — запросы будут буферизованы в kernel socket buffer и обработаны последовательно. Это и есть pipelining.

### 1.2 Схема сообщения: Size + Header + Body

Каждое сообщение Kafka на wire-уровне имеет следующую структуру:

```
┌──────────────────────────────────────────────────────────┐
│                    TCP Payload                            │
├──────────────┬───────────────────────────────────────────┤
│  message_size│         RequestMessage / ResponseMessage   │
│   (INT32)    │                                           │
├──────────────┼──────────────┬──────────────┬─────────────┤
│   4 байта    │   Header     │    Body      │             │
│              │  (перемен.)  │  (перемен.)  │             │
└──────────────┴──────────────┴──────────────┴─────────────┘
```

**BNF-грамматика верхнего уровня (в нотации, обратной порядку wire-сериализации):**

```
RequestOrResponse => Size (RequestMessage | ResponseMessage)
  Size => INT32
```

**`message_size` (INT32, 4 байта):** размер последующего сообщения (header + body) в байтах. Клиент читает: сначала 4 байта как INT32 → N, затем читает и парсит N байт сообщения. Если пришедший размер превышает `socket.request.max.bytes`, брокер разрывает соединение.

### 1.3 Структура заголовка запроса (Request Header, v2)

Request Header содержит метаданные, необходимые брокеру для маршрутизации и обработки запроса:

```
Request Header v2 → api_key api_version correlation_id client_id TAG_BUFFER
  api_key => INT16
  api_version => INT16
  correlation_id => INT32
  client_id => NULLABLE_STRING
```

**Поля заголовка запроса:**

| Поле | Тип | Размер | Описание |
|------|-----|--------|----------|
| `api_key` | INT16 | 2 байта | Числовой идентификатор типа запроса (0 = Produce, 1 = Fetch, 3 = Metadata, и т.д.) |
| `api_version` | INT16 | 2 байта | Версия схемы сообщения. `api_key` + `api_version` однозначно определяют wire-формат |
| `correlation_id` | INT32 | 4 байта | Уникальный идентификатор запроса, генерируемый клиентом. Брокер копирует его в ответ — клиент сопоставляет ответы с запросами |
| `client_id` | NULLABLE_STRING | перемен. | Идентификатор клиентского приложения для логирования и метрик (например, `"myapp-producer-1"`) |

**Аналогия:** Request Header — это как накладная на груз: что за груз (api_key), какой формат упаковки (api_version), номер заказа (correlation_id) и от кого (client_id).

### 1.4 Структура заголовка ответа (Response Header, v1)

```
Response Header v1 → correlation_id TAG_BUFFER
  correlation_id => INT32
```

| Поле | Тип | Описание |
|------|-----|----------|
| `correlation_id` | INT32 | Копируется из запроса — клиент использует для сопоставления |

Ответ не содержит `api_key` — клиент знает, какой ответ ожидать, так как он сам инициировал запрос. `correlation_id` — единственный механизм сопоставления.

### 1.5 Версионирование протокола: как Kafka обеспечивает совместимость

Kafka имеет **двунаправленную совместимость** клиентов и брокеров:

- **Новые клиенты могут работать со старыми брокерами**
- **Старые клиенты могут работать с новыми брокерами**

Это достигается через механизм API version negotiation:

1. Клиент при подключении отправляет `ApiVersionsRequest` (api_key = 18)
2. Брокер возвращает `ApiVersionsResponse`, содержащий список всех поддерживаемых api_key и диапазоны их версий
3. Для каждого запроса клиент выбирает **максимальную версию, поддерживаемую и клиентом, и брокером**
4. Брокер обрабатывает запрос строго по схеме той версии, которую клиент указал в `api_version`

**Пример:** клиент поддерживает ProduceRequest v0-v9, брокер v0-v8 → клиент использует v8.

**Tagged fields (KIP-482):** начиная с версий протокола, помеченных как "flexible", в конец каждого сообщения добавляется TAG_BUFFER. Это позволяет добавлять опциональные поля **без инкремента версии**. Поля, которые получатель не знает, игнорируются. Tagged fields эффективнее для редко используемых полей — они не занимают места, когда не установлены.

**Request Header Version Mapping:** версия Request Header зависит от версии конкретного API-запроса:

```
api_key=X, api_version=Y → request_header_version = f(X, Y)
                                                        ↓
                                              Request Header v0, v1, или v2
```

Версия Response Header аналогично привязана к версии API-ответа.

---

## 2. Примитивные типы данных протокола

Kafka определяет собственный набор примитивных типов. Все многобайтовые целые используют **big-endian (network byte order)**.

### 2.1 Целочисленные типы

| Тип | Диапазон | Размер | Назначение |
|-----|----------|--------|------------|
| `BOOLEAN` | 0 (false) или 1 (true) | 1 байт | По сути INT8: любое ненулевое значение = true |
| `INT8` | -128 … 127 | 1 байт | Компактные значения, коды ошибок в старых версиях |
| `INT16` | -32768 … 32767 | 2 байта | `api_key`, `api_version` |
| `INT32` | -2³¹ … 2³¹-1 | 4 байта | `correlation_id`, `message_size`, длины массивов |
| `INT64` | -2⁶³ … 2⁶³-1 | 8 байт | Offset-ы, timestamp-ы |
| `UINT16` | 0 … 65535 | 2 байта | Беззнаковые короткие значения |
| `UINT32` | 0 … 2³²-1 | 4 байта | Беззнаковые 32-битные значения |
| `VARINT` | -2³¹ … 2³¹-1 | 1-5 байт | Zigzag-кодирование из Protobuf. Используется в компактных типах |
| `VARLONG` | -2⁶³ … 2⁶³-1 | 1-10 байт | Zigzag-кодирование для 64-битных. Используется в компактных типах |
| `FLOAT64` | IEEE 754 double | 8 байт | Числа с плавающей точкой |

### 2.2 Строковые и байтовые типы

| Тип | Формат | Макс. длина |
|-----|--------|-------------|
| `STRING` | INT16 (длина N) + N байт UTF-8 | 32767 байт |
| `NULLABLE_STRING` | INT16 (длина N или -1 для null) + N байт UTF-8 | 32767 байт |
| `COMPACT_STRING` | UNSIGNED_VARINT (N+1) + N байт UTF-8 | Зависит от контекста |
| `COMPACT_NULLABLE_STRING` | UNSIGNED_VARINT (N+1 или 0 для null) + N байт | Зависит от контекста |
| `BYTES` | INT32 (длина N) + N байт | 2³¹-1 байт |
| `NULLABLE_BYTES` | INT32 (длина N или -1 для null) + N байт | 2³¹-1 байт |
| `COMPACT_BYTES` | UNSIGNED_VARINT (N+1) + N байт | Зависит от контекста |
| `COMPACT_NULLABLE_BYTES` | UNSIGNED_VARINT (N+1 или 0) + N байт | Зависит от контекста |

### 2.3 Составные типы

| Тип | Формат |
|-----|--------|
| `[T]` (ARRAY) | INT32 (длина N или -1 для null) + N элементов типа T |
| `[T]` (COMPACT_ARRAY) | UNSIGNED_VARINT (N+1 или 0 для null) + N элементов типа T |
| `RECORDS` | NULLABLE_BYTES, содержащий Kafka record batch |
| `COMPACT_RECORDS` | COMPACT_NULLABLE_BYTES, содержащий Kafka record batch |
| `UUID` | 16 байт в big-endian |
| Структуры | Последовательность полей примитивных/составных типов |

**Главное правило компактных типов:** "COMPACT" и "не-COMPACT" варианты **не смешиваются** в одном запросе. Если версия сообщения помечена как "flexible", все строки/байты/массивы используют компактные варианты и в конце сообщения присутствует TAG_BUFFER.

### 2.4 Пример wire-сериализации

Отправка простого `ApiVersionsRequest` (api_key=18, api_version=0) вручную:

```
TCP-соединение на broker:9092 → отправляем следующие байты:

# message_size = 0x00 0x00 0x00 0x15  (INT32: 21 байт)
00 00 00 15
# Request Header v0
#   api_key = 0x00 0x12           (INT16: 18 = ApiVersions)
00 12
#   api_version = 0x00 0x00       (INT16: 0)
00 00
#   correlation_id = 0x00 0x00 0x00 0x00  (INT32: 0)
00 00 00 00
#   client_id = 0x00 0x05         (INT16: длина 5 → 5 байт)
00 05
#   "mycli"                        (5 байт UTF-8)
6D 79 63 6C 69
# ApiVersionsRequest body (v0):
#   client_software_name = 0x00 0x05    (STRING length)
00 05
#   "mycli"                              (5 байт UTF-8)
6D 79 63 6C 69
#   client_software_version = 0x00 0x05 (STRING length)
00 05
#   "1.0.0"                              (5 байт UTF-8)
31 2E 30 2E 30
```

Ответ брокера (ApiVersionsResponse):

```
# message_size = ... (INT32)
# Response Header v0:
#   correlation_id = 0x00 0x00 0x00 0x00  (скопирован из запроса)
# ApiVersionsResponse body v0:
#   error_code = 0x00 0x00               (INT16: 0 = NONE)
#   api_keys = [...]                     (ARRAY of api_key + min_version + max_version)
#   throttle_time_ms = 0x00 0x00 0x00 0x00  (INT32)
```

---

## 3. Ключевые операции (RPC)

На момент Kafka 4.0 протокол определяет около **90 типов API-сообщений** (часть из них — межброкерные). Ниже разобраны ключевые клиент-брокерные операции.

### 3.1 Обзор ключевых API keys

| API Key | Название | Назначение |
|---------|----------|------------|
| 0 | **Produce** | Отправка сообщений в топик |
| 1 | **Fetch** | Чтение сообщений из топика |
| 2 | **ListOffsets** | Получение доступных offset-ов для партиции |
| 3 | **Metadata** | Получение метаданных кластера (топики, партиции, брокеры) |
| 4 | LeaderAndIsr | Межброкерный: синхронизация лидерства |
| 8 | **OffsetCommit** | Коммит offset-ов consumer group |
| 9 | **OffsetFetch** | Получение закоммиченных offset-ов |
| 10 | **FindCoordinator** | Поиск координатора группы или транзакций |
| 11 | **JoinGroup** | Вход consumer-а в группу |
| 12 | **Heartbeat** | Поддержание членства в группе |
| 14 | **SyncGroup** | Синхронизация назначений партиций в группе |
| 15 | **DescribeGroups** | Получение состояния consumer groups |
| 16 | **ListGroups** | Список всех consumer groups |
| 17 | SaslHandshake | Согласование SASL-механизма |
| 18 | **ApiVersions** | Получение поддерживаемых версий API |
| 19 | **CreateTopics** | Создание топиков (Admin API) |
| 20 | **DeleteTopics** | Удаление топиков (Admin API) |
| 21 | **DeleteRecords** | Удаление записей до указанного offset |
| 22 | InitProducerId | Инициализация идемпотентного/транзакционного продюсера |
| 24 | **DescribeConfigs** | Получение конфигурации ресурсов |
| 25 | **AlterConfigs** | Изменение конфигурации ресурсов |
| 30 | **DescribeCluster** | Получение информации о кластере |
| 36 | **SaslAuthenticate** | Выполнение SASL-аутентификации |
| 37 | **CreatePartitions** | Увеличение количества партиций |
| 42 | **DeleteGroups** | Удаление consumer groups |
| 46 | **DescribeClientQuotas** | Получение клиентских квот |
| 47 | **AlterClientQuotas** | Изменение клиентских квот |
| 49 | **AlterPartitionReassignments** | Изменение назначения реплик |
| 50 | **ListPartitionReassignments** | Список операций reassignment |
| 56 | **DescribeUserScramCredentials** | Получение SCRAM credentials |
| 57 | **AlterUserScramCredentials** | Изменение SCRAM credentials |
| 60 | **DescribeProducers** | Получение информации об активных продюсерах |
| 65 | **DescribeTransactions** | Получение информации о транзакциях |
| 67 | **WriteTxnMarkers** | Межброкерный: запись transaction markers (commit/abort) |

### 3.2 Produce (API Key = 0) — Отправка сообщений

**Назначение:** отправить набор записей (record batches) в одну или несколько партиций топиков.

**Ключевые поля запроса (ProduceRequest, поздние версии):**

| Поле | Тип | Описание |
|------|-----|----------|
| `transactional_id` | COMPACT_NULLABLE_STRING | ID транзакции (null для обычного продюсера) |
| `acks` | INT16 | Уровень подтверждения: 0, 1, -1 (all) |
| `timeout_ms` | INT32 | Таймаут ожидания подтверждения от реплик |
| `topic_data` | COMPACT_ARRAY | Данные по топикам |
| `topic_data[].name` | COMPACT_STRING | Имя топика |
| `topic_data[].partition_data` | COMPACT_ARRAY | Данные по партициям |
| `partition_data[].index` | INT32 | Номер партиции |
| `partition_data[].records` | COMPACT_RECORDS | Батч записей (null = транзакционный abort marker) |

**Аналогия:** ProduceRequest — это как отправить партию товаров на склад. Вы указываете: в какой отдел (topic/partition), сколько товаров (records), и требуете ли вы подтверждение от кладовщика и его напарников (acks).

**Ключевые поля ответа (ProduceResponse):**

| Поле | Тип | Описание |
|------|-----|----------|
| `responses` | COMPACT_ARRAY | Ответы по топикам |
| `responses[].partition_responses` | COMPACT_ARRAY | Ответы по партициям |
| `partition_responses[].error_code` | INT16 | Код ошибки для партиции |
| `partition_responses[].base_offset` | INT64 | Offset первого сообщения в батче |
| `partition_responses[].log_append_time_ms` | INT64 | Время записи в лог (если message.timestamp.type=LogAppendTime) |

**Семантика acks:**

| acks | Поведение |
|------|-----------|
| 0 | Fire-and-forget. Брокер не отправляет подтверждение. Максимальная скорость, риск потери данных. |
| 1 | Лидер партиции записывает в локальный лог и отвечает. Риск потери при падении лидера до репликации. |
| -1 (all) | Лидер ждёт подтверждения от всех in-sync реплик (ISR). Гарантия durability ценой latency. |

### 3.3 Fetch (API Key = 1) — Чтение сообщений

**Назначение:** прочитать набор записей из одной или нескольких партиций топиков.

**Ключевые поля запроса (FetchRequest, поздние версии):**

| Поле | Тип | Описание |
|------|-----|----------|
| `max_wait_ms` | INT32 | Максимальное время ожидания (long poll). Брокер не отвечает сразу, если данных нет — ждёт до этого таймаута |
| `min_bytes` | INT32 | Минимальный объём данных перед ответом. Брокер накапливает данные, пока не наберётся этот минимум (в пределах max_wait_ms) |
| `max_bytes` | INT32 | Максимальный объём данных в ответе (на уровне партиции) |
| `isolation_level` | INT8 | Уровень изоляции: 0 = READ_UNCOMMITTED, 1 = READ_COMMITTED (транзакционные продюсеры) |
| `session_id` | INT32 | ID fetch session для инкрементального fetch |
| `session_epoch` | INT32 | Epoch fetch session |
| `topics` | COMPACT_ARRAY | Список топиков для чтения |
| `topics[].partitions` | COMPACT_ARRAY | Список партиций |
| `partitions[].partition` | INT32 | Номер партиции |
| `partitions[].current_leader_epoch` | INT32 | Эпоха лидера (для обнаружения truncation) |
| `partitions[].fetch_offset` | INT64 | Offset, с которого начинать чтение |
| `partitions[].partition_max_bytes` | INT32 | Лимит байт для этой партиции |

**Аналогия:** FetchRequest — это как система "заказать такси за 10 минут до выхода". `max_wait_ms` = "я подожду максимум 10 минут", `min_bytes` = "но пока не наберётся хотя бы 3 пассажира, не поедем". Long poll позволяет избежать пустых ответов и снизить latency.

**Ключевые поля ответа (FetchResponse):**

| Поле | Тип | Описание |
|------|-----|----------|
| `throttle_time_ms` | INT32 | Время троттлинга в миллисекундах |
| `error_code` | INT16 | Код ошибки верхнего уровня |
| `session_id` | INT32 | ID fetch session |
| `responses` | COMPACT_ARRAY | Ответы по топикам |
| `responses[].partitions[]` | ... | Ответы по партициям |
| `partitions[].error_code` | INT16 | Код ошибки для партиции |
| `partitions[].high_watermark` | INT64 | High watermark offset (последний закоммиченный) |
| `partitions[].last_stable_offset` | INT64 | Последний стабильный offset (для READ_COMMITTED) |
| `partitions[].log_start_offset` | INT64 | Начало доступного лога |
| `partitions[].records` | COMPACT_RECORDS | Записи (null если ошибка или данных нет) |

### 3.4 ListOffsets (API Key = 2) — Получение offset-ов

**Назначение:** узнать, какие offset-ы доступны для чтения в партиции.

**Режимы запроса (isolation_level):**

| Значение | Режим |
|----------|-------|
| 0 | READ_UNCOMMITTED — вернуть offset последнего записанного сообщения |
| 1 | READ_COMMITTED — вернуть offset последнего закоммиченного транзакционного сообщения |

**Ключевые поля запроса:**

| Поле | Тип | Описание |
|------|-----|----------|
| `replica_id` | INT32 | ID реплики (-1 для клиента) |
| `isolation_level` | INT8 | Уровень изоляции |
| `topics` | COMPACT_ARRAY | Топики |
| `topics[].partitions[]` | ... | Партиции |
| `partitions[].timestamp` | INT64 | Временная метка для поиска offset-а |

**Специальные значения `timestamp`:**

| Значение | Смысл |
|----------|-------|
| -2 (`EARLIEST_TIMESTAMP`) | Вернуть самый ранний доступный offset |
| -1 (`LATEST_TIMESTAMP`) | Вернуть самый последний offset (конец лога) |
| > 0 | Найти offset первого сообщения с timestamp ≥ указанного |

**Ключевые поля ответа:**

| Поле | Тип | Описание |
|------|-----|----------|
| `topics[].partitions[].offset` | INT64 | Найденный offset |
| `topics[].partitions[].timestamp` | INT64 | Timestamp сообщения по найденному offset-у |
| `topics[].partitions[].leader_epoch` | INT32 | Эпоха лидера |

### 3.5 Metadata (API Key = 3) — Получение метаданных кластера

**Назначение:** получить информацию о топиках, партициях, брокерах и их ролях. Это **первый запрос**, который должен сделать любой клиент для bootstrapping.

**Ключевые поля запроса (MetadataRequest):**

| Поле | Тип | Описание |
|------|-----|----------|
| `topics` | COMPACT_ARRAY | Список имён топиков (null/пустой = все топики) |
| `topics[].name` | COMPACT_STRING | Имя топика (null для запроса топика по ID) |
| `topics[].topic_id` | UUID | Topic ID |
| `allow_auto_topic_creation` | BOOLEAN | Разрешить автосоздание топика при `auto.create.topics.enable=true` |
| `include_cluster_authorized_operations` | BOOLEAN | Включить в ответ список разрешённых операций |
| `include_topic_authorized_operations` | BOOLEAN | Включить разрешённые операции для топиков |

**Ключевые поля ответа (MetadataResponse):**

| Поле | Тип | Описание |
|------|-----|----------|
| `brokers` | COMPACT_ARRAY | Список брокеров |
| `brokers[].node_id` | INT32 | ID брокера |
| `brokers[].host` | COMPACT_STRING | Хост |
| `brokers[].port` | INT32 | Порт |
| `cluster_id` | COMPACT_NULLABLE_STRING | Уникальный ID кластера |
| `controller_id` | INT32 | ID контроллера |
| `topics` | COMPACT_ARRAY | Метаданные топиков |
| `topics[].error_code` | INT16 | Код ошибки для топика |
| `topics[].name` | COMPACT_STRING | Имя топика |
| `topics[].topic_id` | UUID | Topic ID |
| `topics[].is_internal` | BOOLEAN | Внутренний топик Kafka? (например, `__consumer_offsets`) |
| `topics[].partitions` | COMPACT_ARRAY | Партиции топика |
| `partitions[].error_code` | INT16 | Код ошибки |
| `partitions[].partition_index` | INT32 | Индекс партиции |
| `partitions[].leader_id` | INT32 | ID лидера |
| `partitions[].leader_epoch` | INT32 | Эпоха лидера |
| `partitions[].replica_nodes` | INT32[] | ID всех реплик |
| `partitions[].isr_nodes` | INT32[] | ID in-sync реплик |

**Алгоритм bootstrapping клиента:**

1. Подключиться к **bootstrap-серверу** (одному из списка `bootstrap.servers`)
2. Отправить `ApiVersionsRequest` → согласовать версии
3. Отправить `MetadataRequest` → получить карту кластера
4. Кэшировать метаданные
5. Маршрутизировать Produce/Fetch запросы к лидерам соответствующих партиций
6. При получении `NOT_LEADER_OR_FOLLOWER` (код 6) или ошибки соединения → обновить метаданные

### 3.6 ApiVersions (API Key = 18) — Согласование версий протокола

**Назначение:** определить, какие API и какие версии поддерживает брокер.

**Запрос (v0-v3):** практически пустой. Содержит только `client_software_name` и `client_software_version` (строки) для телеметрии.

**Ответ (ApiVersionsResponse):**

| Поле | Тип | Описание |
|------|-----|----------|
| `error_code` | INT16 | 0 = NONE, 35 = UNSUPPORTED_VERSION (если клиент слишком новый) |
| `api_keys` | COMPACT_ARRAY | Поддерживаемые API |
| `api_keys[].api_key` | INT16 | API key |
| `api_keys[].min_version` | INT16 | Минимальная поддерживаемая версия |
| `api_keys[].max_version` | INT16 | Максимальная поддерживаемая версия |

**Особенность:** в версии брокера ≥ 2.4.0, если клиент отправляет `ApiVersionsRequest` с версией, неподдерживаемой брокером, брокер отвечает `ApiVersionsResponse` v0 с `error_code=UNSUPPORTED_VERSION` и полем `api_keys`, содержащим только поддерживаемую версию ApiVersionsRequest. Это позволяет клиенту сделать повторную попытку с более низкой версией.

---

## 4. Аутентификация в API-запросах

Kafka поддерживает многоуровневую модель аутентификации, реализованную на уровне wire-протокола.

### 4.1 Уровни безопасности listener-ов

Каждый listener (порт) брокера может иметь одну из комбинаций безопасности:

| Протокол | Аутентификация | Шифрование |
|----------|----------------|------------|
| `PLAINTEXT` | Нет | Нет |
| `SSL` | Опционально (client cert) | Да (TLS) |
| `SASL_PLAINTEXT` | Да (SASL) | Нет |
| `SASL_SSL` | Да (SASL) | Да (TLS) |

### 4.2 Последовательность SASL-аутентификации

```
Client                                          Broker
  │                                               │
  │──── TCP SYN ─────────────────────────────────>│
  │<─── TCP SYN-ACK ──────────────────────────────│
  │──── TCP ACK ─────────────────────────────────>│
  │  (если SSL — TLS handshake здесь)             │
  │                                               │
  │──── ApiVersionsRequest (опционально) ────────>│
  │<─── ApiVersionsResponse ──────────────────────│
  │                                               │
  │──── SaslHandshakeRequest ────────────────────>│
  │     (mechanism: "SCRAM-SHA-512")              │
  │<─── SaslHandshakeResponse ────────────────────│
  │     (error_code: 0, mechanisms: [PLAIN,       │
  │      SCRAM-SHA-256, SCRAM-SHA-512])           │
  │                                               │
  │──── SaslAuthenticateRequest ─────────────────>│
  │     (client SASL token)                       │
  │<─── SaslAuthenticateResponse ─────────────────│
  │     (server SASL token, error_code)           │
  │     ... (несколько round-trip при SCRAM)      │
  │                                               │
  │──── SaslAuthenticateRequest (final) ─────────>│
  │<─── SaslAuthenticateResponse ─────────────────│
  │     (error_code: 0 = успех, 58 = неудача)     │
  │                                               │
  │──── MetadataRequest ─────────────────────────>│ ← все последующие запросы аутентифицированы
  │<─── MetadataResponse ─────────────────────────│
```

**Важные моменты:**

- `SaslHandshakeRequest` (api_key=17) — согласование SASL-механизма. Если механизм не поддерживается, брокер возвращает список доступных и закрывает соединение.
- `SaslAuthenticateRequest` (api_key=36) — обмен SASL-токенами. Для SCRAM требуется несколько round-trip.
- `ApiVersionsRequest` отправляется **до** аутентификации — брокер отвечает на него независимо от состояния аутентификации.

### 4.3 Поддерживаемые SASL-механизмы

| Механизм | Описание | Применение |
|----------|----------|------------|
| `PLAIN` | Простой логин/пароль (BASE64) | Только с TLS (SASL_SSL), иначе пароль передаётся открытым текстом |
| `SCRAM-SHA-256` | Salted Challenge Response Authentication Mechanism | Безопаснее PLAIN, не передаёт пароль. Credentials хранятся в ZooKeeper/KRaft |
| `SCRAM-SHA-512` | SCRAM с SHA-512 | Более сильный хэш |
| `GSSAPI` (Kerberos) | Kerberos-аутентификация | Enterprise-среды с Active Directory |
| `OAUTHBEARER` | OAuth 2.0 Bearer токены | Интеграция с OAuth2-провайдерами |
| `Delegation Token` | Токены делегирования Kafka | Передача аутентификации без раскрытия Kerberos-креда |

### 4.4 SSL Client Certificate Authentication

При использовании `SSL` без SASL брокер может аутентифицировать клиента по сертификату. Это настраивается параметром `ssl.client.auth`:

| Значение | Поведение |
|----------|-----------|
| `none` | Клиентский сертификат не требуется |
| `requested` | Брокер запрашивает сертификат, но не требует (опционально) |
| `required` | Клиент **обязан** предоставить валидный сертификат |

После TLS handshake и успешной проверки сертификата все последующие Kafka-запросы аутентифицированы DN (Distinguished Name) сертификата.

### 4.5 Авторизация: ACL поверх аутентификации

После аутентификации identity клиента (principal) используется для ACL-проверок. На уровне wire-протокола авторизация не реализована отдельным RPC — она проверяется брокером внутри каждого запроса:

```
Client → ProduceRequest(topic="orders")
Broker:
  1. Определить principal (из SASL identity / SSL DN)
  2. Проверить ACL: имеет ли principal право WRITE на topic "orders"
  3. Если нет → вернуть ошибку TOPIC_AUTHORIZATION_FAILED (код 29)
  4. Если да → обработать запрос
```

---

## 5. Коды ошибок

Kafka использует 16-битные коды ошибок (`INT16`), возвращаемые в полях `error_code` ответов. Коды стандартизированы и покрывают все возможные сценарии — от сетевых проблем до логических ошибок в запросах.

### 5.1 Категории ошибок

| Категория | Коды | Описание |
|-----------|------|----------|
| Общие ошибки сервера | -1, 0 | `UNKNOWN_SERVER_ERROR`, `NONE` |
| Ошибки offset-ов | 1, 78, 88, 99 | `OFFSET_OUT_OF_RANGE`, `OFFSET_NOT_AVAILABLE`, `UNSTABLE_OFFSET_COMMIT`, `POSITION_OUT_OF_RANGE` |
| Ошибки сообщений | 2, 10, 18, 87 | `CORRUPT_MESSAGE`, `MESSAGE_TOO_LARGE`, `RECORD_LIST_TOO_LARGE`, `INVALID_RECORD` |
| Ошибки метаданных (топик/партиция) | 3, 5, 6, 17, 36, 100, 103 | `UNKNOWN_TOPIC_OR_PARTITION`, `LEADER_NOT_AVAILABLE`, `NOT_LEADER_OR_FOLLOWER`, `INVALID_TOPIC_EXCEPTION`, `TOPIC_ALREADY_EXISTS`, `UNKNOWN_TOPIC_ID`, `INCONSISTENT_TOPIC_ID` |
| Ошибки сети и координации | 4, 7, 8, 9, 13 | `INVALID_FETCH_SIZE`, `REQUEST_TIMED_OUT`, `BROKER_NOT_AVAILABLE`, `REPLICA_NOT_AVAILABLE`, `NETWORK_EXCEPTION` |
| Ошибки координатора | 11, 14, 15, 16, 41 | `STALE_CONTROLLER_EPOCH`, `COORDINATOR_LOAD_IN_PROGRESS`, `COORDINATOR_NOT_AVAILABLE`, `NOT_COORDINATOR`, `NOT_CONTROLLER` |
| Ошибки consumer group | 22-27, 68-69, 79, 81-82, 110-113 | `ILLEGAL_GENERATION`, `INCONSISTENT_GROUP_PROTOCOL`, `INVALID_GROUP_ID`, `UNKNOWN_MEMBER_ID`, `INVALID_SESSION_TIMEOUT`, `REBALANCE_IN_PROGRESS`, `NON_EMPTY_GROUP`, `GROUP_ID_NOT_FOUND`, `MEMBER_ID_REQUIRED`, `GROUP_MAX_SIZE_REACHED`, `FENCED_INSTANCE_ID`, `FENCED_MEMBER_EPOCH`, `UNRELEASED_INSTANCE_ID`, `UNSUPPORTED_ASSIGNOR`, `STALE_MEMBER_EPOCH` |
| Ошибки авторизации | 29, 30, 31, 53, 65 | `TOPIC_AUTHORIZATION_FAILED`, `GROUP_AUTHORIZATION_FAILED`, `CLUSTER_AUTHORIZATION_FAILED`, `TRANSACTIONAL_ID_AUTHORIZATION_FAILED`, `DELEGATION_TOKEN_AUTHORIZATION_FAILED` |
| Ошибки SASL | 33, 34, 58 | `UNSUPPORTED_SASL_MECHANISM`, `ILLEGAL_SASL_STATE`, `SASL_AUTHENTICATION_FAILED` |
| Ошибки конфигурации | 37-40 | `INVALID_PARTITIONS`, `INVALID_REPLICATION_FACTOR`, `INVALID_REPLICA_ASSIGNMENT`, `INVALID_CONFIG` |
| Ошибки транзакций | 45-53, 59, 90, 105, 120 | `OUT_OF_ORDER_SEQUENCE_NUMBER`, `DUPLICATE_SEQUENCE_NUMBER`, `INVALID_PRODUCER_EPOCH`, `INVALID_TXN_STATE`, `INVALID_PRODUCER_ID_MAPPING`, `INVALID_TRANSACTION_TIMEOUT`, `CONCURRENT_TRANSACTIONS`, `TRANSACTION_COORDINATOR_FENCED`, `UNKNOWN_PRODUCER_ID`, `PRODUCER_FENCED`, `TRANSACTIONAL_ID_NOT_FOUND`, `TRANSACTION_ABORTABLE` |
| Ошибки безопасности и токенов | 54, 61-67, 93 | `SECURITY_DISABLED`, `DELEGATION_TOKEN_AUTH_DISABLED`, `DELEGATION_TOKEN_NOT_FOUND`, `DELEGATION_TOKEN_OWNER_MISMATCH`, `DELEGATION_TOKEN_REQUEST_NOT_ALLOWED`, `DELEGATION_TOKEN_AUTHORIZATION_FAILED`, `DELEGATION_TOKEN_EXPIRED`, `INVALID_PRINCIPAL_TYPE`, `UNACCEPTABLE_CREDENTIAL` |
| Ошибки хранения | 56, 57 | `KAFKA_STORAGE_ERROR`, `LOG_DIR_NOT_FOUND` |
| Ошибки reassignment | 60, 85 | `REASSIGNMENT_IN_PROGRESS`, `NO_REASSIGNMENT_IN_PROGRESS` |
| Ошибки fetch session | 70, 71, 106 | `FETCH_SESSION_ID_NOT_FOUND`, `INVALID_FETCH_SESSION_EPOCH`, `FETCH_SESSION_TOPIC_ID_ERROR` |
| Ошибки лидера | 72, 74, 75, 80, 83, 84, 108 | `LISTENER_NOT_FOUND`, `FENCED_LEADER_EPOCH`, `UNKNOWN_LEADER_EPOCH`, `PREFERRED_LEADER_NOT_AVAILABLE`, `ELIGIBLE_LEADERS_NOT_AVAILABLE`, `ELECTION_NOT_NEEDED`, `NEW_LEADER_ELECTED` |
| Ошибки троттлинга | 89 | `THROTTLING_QUOTA_EXCEEDED` |
| KRaft-специфичные ошибки | 94-98, 101, 102, 104, 107, 116, 119, 125-127 | `INCONSISTENT_VOTER_SET`, `INVALID_UPDATE_VERSION`, `FEATURE_UPDATE_FAILED`, `PRINCIPAL_DESERIALIZATION_FAILURE`, `SNAPSHOT_NOT_FOUND`, `DUPLICATE_BROKER_REGISTRATION`, `BROKER_ID_NOT_REGISTERED`, `INCONSISTENT_CLUSTER_ID`, `INELIGIBLE_REPLICA`, `UNKNOWN_CONTROLLER_ID`, `INVALID_REGISTRATION`, `INVALID_VOTER_KEY`, `DUPLICATE_VOTER`, `VOTER_NOT_FOUND` |
| Tiered Storage | 109 | `OFFSET_MOVED_TO_TIERED_STORAGE` |
| Telemetry | 117, 118 | `UNKNOWN_SUBSCRIPTION_ID`, `TELEMETRY_TOO_LARGE` |
| Share Groups | 122-124 | `SHARE_SESSION_NOT_FOUND`, `INVALID_SHARE_SESSION_EPOCH`, `FENCED_STATE_EPOCH` |
| Прочее | 128, 129 | `INVALID_REGULAR_EXPRESSION`, `REBOOTSTRAP_REQUIRED` |

### 5.2 Свойство Retriable

Каждый код ошибки имеет флаг **Retriable** (`True` / `False`). Это ключевой атрибут для клиентских библиотек:

- **Retriable = True:** клиент **может** повторить запрос. Обычно это транзиентные ошибки (временное отсутствие лидера, переизбрание контроллера, сетевой сбой).
- **Retriable = False:** повторная попытка бессмысленна. Ошибка либо логическая (неверный group.id), либо требует изменения конфигурации (топик не существует), либо проблема с авторизацией.

**Примеры retriable ошибок:**

| Код | Ошибка | Retriable | Сценарий |
|-----|--------|-----------|----------|
| 3 | `UNKNOWN_TOPIC_OR_PARTITION` | True | Запрос к брокеру, который не хостит эту партицию → обновить метаданные и повторить |
| 5 | `LEADER_NOT_AVAILABLE` | True | Идёт выборы лидера → подождать и повторить |
| 6 | `NOT_LEADER_OR_FOLLOWER` | True | Запрос не тому брокеру → обновить метаданные |
| 7 | `REQUEST_TIMED_OUT` | True | Таймаут → повторить |
| 89 | `THROTTLING_QUOTA_EXCEEDED` | True | Превышена квота → подождать, повторить |

**Примеры не-retriable ошибок:**

| Код | Ошибка | Retriable | Сценарий |
|-----|--------|-----------|----------|
| 29 | `TOPIC_AUTHORIZATION_FAILED` | False | Нет прав на запись → повтор не поможет |
| 36 | `TOPIC_ALREADY_EXISTS` | False | Топик уже существует → надо выбрать другое имя |
| 1 | `OFFSET_OUT_OF_RANGE` | False | Offset вне допустимого диапазона → сбросить offset |
| 17 | `INVALID_TOPIC_EXCEPTION` | False | Некорректное имя топика |

### 5.3 Полная таблица кодов ошибок (избранное)

Ниже — наиболее часто встречающиеся коды. Полный список (130+ кодов) доступен в официальной документации.

| Код | Название | Retriable | Описание |
|-----|----------|-----------|----------|
| -1 | `UNKNOWN_SERVER_ERROR` | False | Неожиданная ошибка сервера |
| 0 | `NONE` | False | Успех |
| 1 | `OFFSET_OUT_OF_RANGE` | False | Запрошенный offset вне диапазона |
| 2 | `CORRUPT_MESSAGE` | True | Сообщение повреждено (CRC, размер, null-ключ в compacted topic) |
| 3 | `UNKNOWN_TOPIC_OR_PARTITION` | True | Сервер не хостит этот топик-партицию |
| 5 | `LEADER_NOT_AVAILABLE` | True | Нет лидера для партиции (переизбрание) |
| 6 | `NOT_LEADER_OR_FOLLOWER` | True | Брокер не является лидером/репликой этой партиции |
| 7 | `REQUEST_TIMED_OUT` | True | Таймаут запроса |
| 10 | `MESSAGE_TOO_LARGE` | False | Сообщение > max.message.bytes |
| 13 | `NETWORK_EXCEPTION` | True | Разрыв соединения до получения ответа |
| 15 | `COORDINATOR_NOT_AVAILABLE` | True | Координатор групп недоступен |
| 16 | `NOT_COORDINATOR` | True | Это неправильный координатор |
| 17 | `INVALID_TOPIC_EXCEPTION` | False | Некорректное имя топика |
| 19 | `NOT_ENOUGH_REPLICAS` | True | Недостаточно in-sync реплик |
| 22 | `ILLEGAL_GENERATION` | False | Неверный generation.id consumer group |
| 25 | `UNKNOWN_MEMBER_ID` | False | Координатор не знает этого члена группы |
| 26 | `INVALID_SESSION_TIMEOUT` | False | session.timeout.ms вне допустимого диапазона |
| 27 | `REBALANCE_IN_PROGRESS` | False | Группа перебалансируется |
| 29 | `TOPIC_AUTHORIZATION_FAILED` | False | Нет прав на топик |
| 30 | `GROUP_AUTHORIZATION_FAILED` | False | Нет прав на группу |
| 31 | `CLUSTER_AUTHORIZATION_FAILED` | False | Нет прав на кластер |
| 35 | `UNSUPPORTED_VERSION` | False | Неподдерживаемая версия API |
| 36 | `TOPIC_ALREADY_EXISTS` | False | Топик уже существует |
| 37 | `INVALID_PARTITIONS` | False | Количество партиций < 1 |
| 38 | `INVALID_REPLICATION_FACTOR` | False | Replication factor < 1 или > числа брокеров |
| 41 | `NOT_CONTROLLER` | True | Это не контроллер кластера |
| 52 | `TRANSACTION_COORDINATOR_FENCED` | False | Транзакционный координатор заменён |
| 58 | `SASL_AUTHENTICATION_FAILED` | False | SASL-аутентификация не удалась |
| 59 | `UNKNOWN_PRODUCER_ID` | False | Producer ID не найден (лог продюсера удалён по retention) |
| 68 | `NON_EMPTY_GROUP` | False | Группа не пуста (нельзя удалить) |
| 69 | `GROUP_ID_NOT_FOUND` | False | Group ID не существует |
| 74 | `FENCED_LEADER_EPOCH` | True | Эпоха лидера в запросе устарела |
| 82 | `FENCED_INSTANCE_ID` | False | Конфликт group.instance.id |
| 87 | `INVALID_RECORD` | False | Запись не прошла валидацию |
| 88 | `UNSTABLE_OFFSET_COMMIT` | True | Есть нестабильные offset-ы (транзакции) |
| 89 | `THROTTLING_QUOTA_EXCEEDED` | True | Превышена квота троттлинга |
| 90 | `PRODUCER_FENCED` | False | Более новый продюсер с тем же transactional.id |
| 100 | `UNKNOWN_TOPIC_ID` | True | Сервер не хостит топик с этим topic_id |
| 109 | `OFFSET_MOVED_TO_TIERED_STORAGE` | False | Offset перемещён в tiered storage |
| 110 | `FENCED_MEMBER_EPOCH` | False | Member epoch заблокирован координатором |
| 129 | `REBOOTSTRAP_REQUIRED` | False | Метаданные клиента устарели — требуется rebootstrap |

---

## 6. Admin API: операции управления кластером

Admin API Kafka позволяет программно управлять кластером — создавать/удалять топики, изменять конфигурацию, управлять consumer groups, ACL, квотами. На wire-уровне каждая операция Admin API реализована отдельным RPC (свой `api_key`), но Java-клиент `AdminClient` предоставляет унифицированный интерфейс.

### 6.1 Архитектура AdminClient

```
┌────────────────────────────────────────────────┐
│              Application Code                   │
├────────────────────────────────────────────────┤
│            AdminClient (Java)                   │
│  ┌──────────────────────────────────────────┐  │
│  │  createTopics() → CreateTopicsRequest    │  │
│  │  deleteTopics() → DeleteTopicsRequest    │  │
│  │  describeCluster() → DescribeCluster     │  │
│  │  listConsumerGroups() → ListGroups       │  │
│  │  describeConfigs() → DescribeConfigs     │  │
│  │  alterConfigs() → AlterConfigs           │  │
│  │  ...                                      │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│           Kafka Wire Protocol (TCP)             │
├────────────────────────────────────────────────┤
│                 Kafka Broker                    │
└────────────────────────────────────────────────┘
```

**Важно:** AdminClient — это Java-абстракция над wire-протоколом. В других языках доступность Admin-функциональности зависит от клиентской библиотеки (см. статью [01-client-libraries.md](./01-client-libraries.md) — матрицу возможностей клиентов).

### 6.2 Операции с топиками

#### CreateTopics (API Key = 19)

```java
// Создание топика через AdminClient
Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

try (AdminClient admin = AdminClient.create(props)) {
    NewTopic topic = new NewTopic("orders", 3, (short) 2); // 3 партиции, RF=2
    topic.configs(Map.of(
        "retention.ms", "86400000",      // хранить 1 день
        "cleanup.policy", "compact"       // compaction вместо delete
    ));
    CreateTopicsResult result = admin.createTopics(Collections.singleton(topic));
    result.all().get(); // дождаться завершения
    System.out.println("Топик создан успешно");
}
```

**Что происходит на wire-уровне:**

```
CreateTopicsRequest
├── topics[]
│   ├── name: "orders"
│   ├── num_partitions: 3
│   ├── replication_factor: 2
│   └── configs[]
│       ├── {name: "retention.ms", value: "86400000"}
│       └── {name: "cleanup.policy", value: "compact"}
├── timeout_ms: 30000
└── validate_only: false
```

**Возможные варианты создания:**

| Параметр | Описание |
|----------|----------|
| `num_partitions` + `replication_factor` | Простой путь: Kafka сама распределяет реплики |
| `replica_assignment` | Ручное распределение: указать, на каких брокерах каждая партиция |
| `validate_only=true` | Только провалидировать запрос, не создавать |

#### DeleteTopics (API Key = 20)

```java
DeleteTopicsResult result = admin.deleteTopics(
    Arrays.asList("orders", "notifications"));
result.all().get();
```

**Wire-уровень:**

```
DeleteTopicsRequest
├── topics: [{name: "orders"}, {name: "notifications"}]
└── timeout_ms: 30000
```

#### CreatePartitions (API Key = 37) — увеличение числа партиций

```java
Map<String, NewPartitions> newPartitions = Map.of(
    "orders", NewPartitions.increaseTo(6) // было 3 → станет 6
);
admin.createPartitions(newPartitions).all().get();
```

**Ограничение:** уменьшить количество партиций **нельзя**. Только увеличение. Это архитектурное ограничение Kafka — партиции не могут быть объединены.

### 6.3 Операции с конфигурацией

#### DescribeConfigs (API Key = 24)

```java
ConfigResource resource = new ConfigResource(
    ConfigResource.Type.TOPIC, "orders");
DescribeConfigsResult result = admin.describeConfigs(
    Collections.singleton(resource));
Config config = result.all().get().get(resource);
config.entries().forEach(entry -> 
    System.out.printf("%s = %s (default: %s)%n",
        entry.name(), entry.value(), entry.isDefault()));
```

**Wire-уровень:**

```
DescribeConfigsRequest
├── resources[]
│   └── {resource_type: TOPIC, resource_name: "orders"}
└── include_synonyms: true
```

```
DescribeConfigsResponse
├── resources[]
│   └── configs[]
│       ├── {name: "retention.ms", value: "86400000", source: DYNAMIC_TOPIC_CONFIG, ...}
│       ├── {name: "segment.bytes", value: "1073741824", source: DEFAULT_CONFIG, ...}
│       └── ...
```

#### AlterConfigs (API Key = 25) — изменение конфигурации

```java
ConfigResource resource = new ConfigResource(
    ConfigResource.Type.TOPIC, "orders");
ConfigEntry entry = new ConfigEntry("retention.ms", "172800000"); // 2 дня
Map<ConfigResource, Collection<AlterConfigOp>> configs = Map.of(
    resource, Collections.singleton(new AlterConfigOp(entry, AlterConfigOp.OpType.SET))
);
admin.incrementalAlterConfigs(configs).all().get();
```

**Типы ресурсов для конфигурации:**

| Тип | Wire-значение | Пример имени |
|-----|---------------|-------------|
| `TOPIC` | 2 | `"orders"` |
| `BROKER` | 4 | `"0"` (broker ID) |
| `BROKER_LOGGER` | 8 | `"0"` |

**Типы операций (incrementalAlterConfigs):**

| OpType | Описание |
|--------|----------|
| `SET` | Установить значение |
| `DELETE` | Удалить кастомное значение (вернуться к default) |
| `APPEND` | Добавить к существующему списковому значению |
| `SUBTRACT` | Удалить из спискового значения |

### 6.4 Операции с кластером

#### DescribeCluster (API Key = 30)

```java
DescribeClusterResult result = admin.describeCluster();
KafkaFuture<Collection<Node>> nodes = result.nodes();
KafkaFuture<Node> controller = result.controller();
KafkaFuture<String> clusterId = result.clusterId();

System.out.println("Cluster ID: " + clusterId.get());
System.out.println("Controller: " + controller.get());
nodes.get().forEach(node -> 
    System.out.printf("Broker %d: %s:%d%n", 
        node.id(), node.host(), node.port()));
```

**Wire-уровень:**

```
DescribeClusterResponse
├── cluster_id: "abc123def456"
├── controller_id: 1
├── brokers[]
│   ├── {node_id: 0, host: "broker0.kafka.local", port: 9092, rack: "us-east-1a"}
│   ├── {node_id: 1, host: "broker1.kafka.local", port: 9092, rack: "us-east-1b"}
│   └── {node_id: 2, host: "broker2.kafka.local", port: 9092, rack: "us-east-1c"}
└── cluster_authorized_operations: [...]
```

### 6.5 Операции с Consumer Groups

#### ListConsumerGroups (API Key = 16)

```java
ListConsumerGroupsResult result = admin.listConsumerGroups();
result.all().get().forEach(group -> 
    System.out.printf("Group: %s, State: %s, Protocol: %s%n", 
        group.groupId(), group.state(), group.protocolType()));
```

#### DescribeConsumerGroups (API Key = 15)

```java
DescribeConsumerGroupsResult result = admin.describeConsumerGroups(
    Arrays.asList("my-consumer-group"));
ConsumerGroupDescription desc = result.all().get().get("my-consumer-group");

System.out.println("State: " + desc.state());
System.out.println("Coordinator: " + desc.coordinator());
desc.members().forEach(member -> {
    System.out.printf("  Member: %s (%s)%n", member.consumerId(), member.host());
    member.assignment().topicPartitions().forEach(tp ->
        System.out.printf("    assigned: %s-%d%n", tp.topic(), tp.partition()));
});
```

#### DeleteConsumerGroups (API Key = 42)

```java
admin.deleteConsumerGroups(
    Collections.singletonList("my-consumer-group")).all().get();
```

### 6.6 Операции с ACL

#### DescribeAcls (API Key = 29)

```java
AclBindingFilter filter = new AclBindingFilter(
    new ResourcePatternFilter(ResourceType.TOPIC, "orders", PatternType.LITERAL),
    new AccessControlEntryFilter(null, null, AclOperation.ANY, AclPermissionType.ANY)
);
DescribeAclsResult result = admin.describeAcls(filter);
result.values().get().forEach(acl ->
    System.out.printf("Principal: %s, Host: %s, Operation: %s%n",
        acl.entry().principal(), acl.entry().host(), acl.entry().operation()));
```

#### CreateAcls (API Key = 28)

```java
AclBinding acl = new AclBinding(
    new ResourcePattern(ResourceType.TOPIC, "orders", PatternType.LITERAL),
    new AccessControlEntry("User:alice", "*", AclOperation.WRITE, AclPermissionType.ALLOW)
);
admin.createAcls(Collections.singleton(acl)).all().get();
```

### 6.7 Операции с квотами

#### DescribeClientQuotas (API Key = 46)

```java
DescribeClientQuotasResult result = admin.describeClientQuotas(
    ClientQuotaFilter.all());
result.entities().get().forEach((entry) -> {
    entry.entity().forEach((type, name) -> 
        System.out.printf("Entity: %s=%s%n", type, name));
    entry.quotas().forEach((key, value) ->
        System.out.printf("  %s = %.2f%n", key, value));
});
```

**Типы квот:**

| Quota | Описание |
|-------|----------|
| `producer_byte_rate` | Ограничение bandwidth продюсера (байт/с) |
| `consumer_byte_rate` | Ограничение bandwidth консьюмера (байт/с) |
| `request_percentage` | Доля от общего числа request handler thread-ов |

### 6.8 Типовые паттерны AdminClient

#### Атомарное создание топика с проверкой

```java
public void createTopicIfNotExists(AdminClient admin, String name, 
                                    int partitions, short rf) {
    try {
        admin.describeTopics(Collections.singletonList(name))
             .allTopicNames().get();
        System.out.println("Топик '" + name + "' уже существует");
    } catch (ExecutionException e) {
        if (e.getCause() instanceof UnknownTopicOrPartitionException) {
            NewTopic topic = new NewTopic(name, partitions, rf);
            admin.createTopics(Collections.singleton(topic)).all().get();
            System.out.println("Топик '" + name + "' создан");
        } else {
            throw new RuntimeException(e);
        }
    }
}
```

#### Массовое удаление топиков по префиксу

```java
public void deleteTopicsByPrefix(AdminClient admin, String prefix) {
    Set<String> allTopics = admin.listTopics().names().get();
    Set<String> toDelete = allTopics.stream()
        .filter(name -> name.startsWith(prefix))
        .collect(Collectors.toSet());
    
    if (toDelete.isEmpty()) {
        System.out.println("Нет топиков с префиксом '" + prefix + "'");
        return;
    }
    
    System.out.println("Удаляю " + toDelete.size() + " топиков: " + toDelete);
    admin.deleteTopics(toDelete).all().get();
}
```

---

## 7. Инструменты для работы с wire-протоколом

### 7.1 Wireshark — захват и анализ трафика

Wireshark (начиная с версии 4.x) имеет встроенный dissector для Kafka-протокола:

```
wireshark -k -i lo0 -f "port 9092" -Y "kafka"
```

Это позволяет видеть полную структуру каждого запроса/ответа с именами полей и их значениями. Dissector поддерживает большинство версий сообщений (зависит от версии Wireshark).

### 7.2 kcat (ранее kafkacat) — CLI для wire-level отладки

```bash
# Просмотр метаданных
kcat -b localhost:9092 -L

# Чтение с указанием offset и количества сообщений
kcat -b localhost:9092 -t orders -p 0 -o beginning -c 10 -f '%o: %k → %s\n'

# Отправка сообщения
echo "hello kafka" | kcat -b localhost:9092 -t orders -P
```

### 7.3 Прямое подключение через telnet/netcat

Для низкоуровневой отладки можно использовать raw TCP:

```bash
# Подключиться
nc localhost 9092

# ... и вручную отправлять бинарные данные (через hex-редактор или скрипт)
```

### 7.4 Kio (Python) — библиотека для работы с wire-протоколом

[Kio](https://github.com/Aiven-Open/kio) — Python-библиотека от Aiven для сериализации/десериализации Kafka wire-протокола без использования librdkafka. Полезна для кастомных клиентов и инструментов.

```python
from kio import serialize_request_header
# прямой доступ к бинарной сериализации запросов
```

---

## 8. Практические рекомендации

### Чеклист для реализации клиента с нуля

1. ✅ Реализовать чтение/запись примитивных типов (INT8..INT64, STRING, BYTES, ARRAY)
2. ✅ Реализовать COMPACT-варианты (VARINT/VARLONG, COMPACT_STRING, COMPACT_ARRAY, TAG_BUFFER)
3. ✅ Реализовать Request/Response Header (v0, v1, v2)
4. ✅ Реализовать `ApiVersionsRequest`/`Response` — согласование версий
5. ✅ Реализовать `SaslHandshakeRequest` + `SaslAuthenticateRequest` — аутентификация (если нужна)
6. ✅ Реализовать `MetadataRequest`/`Response` — bootstrapping
7. ✅ Реализовать `ProduceRequest`/`Response` — отправка данных
8. ✅ Реализовать `FetchRequest`/`Response` — чтение данных
9. ✅ Реализовать retry-логику с учётом `Retriable` флагов ошибок
10. ✅ Реализовать обновление метаданных при `NOT_LEADER_OR_FOLLOWER`

### Распространённые ошибки при работе с протоколом

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `UNSUPPORTED_VERSION (35)` | Клиент использует версию, неподдерживаемую брокером | Использовать `ApiVersionsRequest` для согласования |
| `NOT_LEADER_OR_FOLLOWER (6)` | Запрос отправлен не тому брокеру | Обновить метаданные через `MetadataRequest` |
| `OFFSET_OUT_OF_RANGE (1)` | Запрошен offset, который уже удалён retention policy | Использовать `ListOffsets` для получения актуальных offset-ов |
| Разрыв соединения после отправки большого запроса | Превышен `socket.request.max.bytes` | Уменьшить размер батча или увеличить лимит на брокере |
| `CORRUPT_MESSAGE (2)` | Неверная CRC записей | Проверить wire-сериализацию RECORDS, особенно CRC |
| `MESSAGE_TOO_LARGE (10)` | Сообщение > `max.message.bytes` | Разбить на меньшие сообщения или увеличить лимит |

---

## 9. Связанные статьи

- [01-client-libraries.md](./01-client-libraries.md) — Клиентские библиотеки Kafka: сравнительная таблица языков и возможностей
- [03-patterns.md](./03-patterns.md) — Паттерны использования Kafka API
- [04-testing.md](./04-testing.md) — Тестирование Kafka-приложений
- [../02-basics/02-how-it-works.md](../02-basics/02-how-it-works.md) — Как работает Kafka: архитектура и внутреннее устройство
- [../02-basics/03-core-concepts.md](../02-basics/03-core-concepts.md) — Ключевые концепции Kafka

---

## Источники

1. **Apache Kafka 4.0 Protocol Guide** — официальная документация wire-протокола. Apache Software Foundation, 2024. `kafka.apache.org/40/design/protocol/`
2. **Apache Kafka 4.0 Protocol Errors** — официальный справочник кодов ошибок. Apache Software Foundation, 2024. `kafka.apache.org/40/generated/protocol_errors.html`
3. **A Guide To The Kafka Protocol** — оригинальное руководство по протоколу на Apache Wiki. `cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol`
4. **Ivan Yurchenko, "Kafka Protocol Practical Guide"** — практическое руководство с Python-примерами, сентябрь 2024. `ivanyu.me/blog/2024/09/08/kafka-protocol-practical-guide/`
5. **Apache Kafka 4.0 AdminClient Javadoc** — официальная документация Admin API. `kafka.apache.org/40/javadoc/org/apache/kafka/clients/admin/Admin.html`
6. **Conduktor Kafka Protocol Explorer** — интерактивный обозреватель версий протокола. `kafka-options-explorer.conduktor.io/protocol`
7. **KIP-482: Tagged Fields** — спецификация tagged fields в протоколе. `cwiki.apache.org/confluence/display/KAFKA/KIP-482`
8. **KIP-35: Retrieving Protocol Version** — спецификация ApiVersions запроса. `cwiki.apache.org/confluence/display/KAFKA/KIP-35`
9. **KIP-511: Client Name and Version** — спецификация клиентской телеметрии. `cwiki.apache.org/confluence/display/KAFKA/KIP-511`
10. **Kafka Wire Format (Apache Wiki)** — историческая спецификация wire-формата (2011). `cwiki.apache.org/confluence/display/KAFKA/Wire+Format`
11. **Apache Kafka Source Code** — репозиторий на GitHub, директория `clients/src/main/resources/common/message/` — JSON-схемы всех API-сообщений. `github.com/apache/kafka`
12. **Kio Library** — Python-библиотека для сериализации Kafka протокола, Aiven. `github.com/Aiven-Open/kio`

---

*Статья 14/37 проекта «Apache Kafka: полное руководство». Track 5 (Development & API), часть 2 из 4.*
