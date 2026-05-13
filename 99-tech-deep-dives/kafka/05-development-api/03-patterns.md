# Паттерны использования Apache Kafka: Event Sourcing, CQRS, CDC, Streaming ETL и Pub/Sub

> **Нижняя строка:** Apache Kafka — это не просто message broker, а фундамент для шести ключевых архитектурных паттернов: Event Sourcing (журнал событий как источник истины), CQRS (разделение чтения и записи), CDC (захват изменений из БД), Streaming ETL (потоковая трансформация данных), Pub/Sub (издатель-подписчик) и Outbox Pattern (атомарная запись в БД + Kafka). Каждый паттерн решает конкретный класс проблем, но имеет чёткие границы применимости и характерные антипаттерны. В этой статье — идиоматичный код на Java, Python, Go и Node.js, best practices для producer/consumer и чеклист выбора паттерна под задачу.

---

## 1. Шесть ключевых паттернов: карта решений

Прежде чем погружаться в детали каждого паттерна, сориентируемся по задачам:

| Паттерн | Какую проблему решает | Когда НЕ применять |
|---------|----------------------|-------------------|
| **Event Sourcing** | Сохранять не текущее состояние, а все изменения (аудит, временны́е срезы) | Частая реконструкция агрегатов, малые объёмы событий |
| **CQRS** | Разделить модель чтения и записи для независимого масштабирования | Простые CRUD без существенной асимметрии нагрузки |
| **CDC** | Захватывать изменения из БД без модификации прикладного кода | СУБД без support-а CDC (нет WAL-доступа), монолит без потребности в событиях |
| **Streaming ETL** | Трансформировать и перемещать данные в реальном времени между системами | Пакетная обработка без требований к latency < 1 мин |
| **Pub/Sub** | Доставлять одно событие множеству независимых потребителей | Гарантированная доставка каждому подписчику в реальном времени (требует consumer groups) |
| **Outbox Pattern** | Атомарно записать в БД и отправить событие в Kafka | Одна БД без CDC-инструмента, простая архитектура без микросервисов |

**Аналогия:** представьте аэропорт. Event Sourcing — это журнал всех рейсов (событий) с момента открытия. CQRS — разные табло для пассажиров (чтение) и диспетчеров (запись). CDC — автоматическая трансляция изменений в расписании всем системам. Streaming ETL — конвейер багажа с сортировкой и переупаковкой. Pub/Sub — оповещение о рейсе всем заинтересованным службам одновременно.

---

## 2. Event Sourcing на Kafka

### 2.1 Суть паттерна

**Event Sourcing** — хранение состояния системы как последовательности неизменяемых событий (events), а не как текущего значения (state). Вместо:

```sql
UPDATE orders SET status = 'SHIPPED' WHERE id = 123;
```

Сохраняем:

```json
{"type": "OrderCreated", "orderId": "123", "items": [...]}
{"type": "OrderPaid", "orderId": "123", "amount": 1500}
{"type": "OrderShipped", "orderId": "123", "carrier": "DHL"}
```

**Почему Kafka — естественная среда для Event Sourcing:**
- Append-only лог — события неизменяемы и упорядочены
- Consumer groups — параллельная обработка одного потока разными проекциями
- Бесконечное хранение (`retention.ms=-1`) — события навсегда

### 2.2 Проектирование топиков

**Правило:** один топик на тип агрегата. Ключ сообщения = идентификатор агрегата.

```bash
# Топик с бесконечным хранением — события никогда не удаляются
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic order-events \
  --partitions 12 \
  --config retention.ms=-1 \
  --config cleanup.policy=delete
```

**Почему retention.ms=-1:** события — source of truth. Удаление события = потеря истории. Если нужна очистка по TTL для compliance, используйте отдельный compacted-топик для снапшотов.

### 2.3 Проблема реконструкции агрегатов и решения

**Главная боль:** Kafka не имеет операции «дай все события для ключа X». Чтобы восстановить состояние одного агрегата, нужно просканировать всю партицию — для топика с миллионами событий это неприемлемо.

**Решение 1: поддерживать состояние (KTable, материализованное представление)**

```java
// Kafka Streams — инкрементальная агрегация при поступлении событий
KStream<String, OrderEvent> events = builder.stream("order-events");

KTable<String, Order> orders = events
    .groupByKey()
    .aggregate(
        Order::new,                                      // начальное состояние
        (orderId, event, order) -> order.apply(event),   // применение события
        Materialized.<String, Order, KeyValueStore<Bytes, byte[]>>
            .as("order-store")                           // RocksDB-стор
            .withKeySerde(Serdes.String())
            .withValueSerde(new OrderSerde())
    );
```

На Python (через Faust или вручную через БД):

```python
from kafka import KafkaConsumer
import json
import psycopg2

consumer = KafkaConsumer(
    'order-events',
    bootstrap_servers='localhost:9092',
    group_id='order-state-builder',
    enable_auto_commit=False,
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Поддерживаем состояние в PostgreSQL (O(1)-доступ к агрегату)
conn = psycopg2.connect("dbname=orders user=app")

for message in consumer:
    event = message.value
    order_id = message.key.decode('utf-8') if message.key else None

    with conn:
        with conn.cursor() as cur:
            # UPSERT — либо создаём, либо обновляем агрегат
            cur.execute("""
                INSERT INTO order_projections (order_id, status, amount, version)
                VALUES (%s, %s, %s, %s)
                ON CONFLICT (order_id) DO UPDATE SET
                    status = CASE
                        WHEN order_projections.version < EXCLUDED.version
                        THEN EXCLUDED.status
                        ELSE order_projections.status
                    END,
                    version = GREATEST(order_projections.version, EXCLUDED.version)
            """, (order_id, event.get('status'), event.get('amount', 0), event.get('version', 1)))

    consumer.commit()  # at-least-once: коммитим после успешной записи в БД
```

**Решение 2: снапшоты агрегатов**

Периодически сохраняем полное состояние агрегата в compacted-топик. При реконструкции загружаем снапшот и накатываем события после него.

```bash
# Снапшоты — compacted (храним только последнее состояние для каждого ключа)
kafka-topics --create --topic order-snapshots \
  --config cleanup.policy=compact
```

```java
// Псевдокод реконструкции агрегата со снапшотом
public Order reconstruct(String orderId) {
    // 1. Ищем последний снапшот
    OrderSnapshot snapshot = findLatestSnapshot(orderId);  // O(log N) через индекс
    Order order = snapshot != null ? snapshot.getState() : new Order();

    long lastEventOffset = snapshot != null
        ? snapshot.getOffset() + 1
        : 0;

    // 2. Накатываем только события после снапшота
    List<OrderEvent> eventsAfterSnapshot = consumer.pollFrom(
        "order-events", orderId, lastEventOffset);

    for (OrderEvent event : eventsAfterSnapshot) {
        order = order.apply(event);
    }
    return order;
}
```

**Важно:** Event Sourcing на Kafka — компромисс. Kafka даёт масштабируемость и throughput, но не даёт встроенной оптимистичной конкурентности (как EventStoreDB). Если ваш основной use-case — частая реконструкция отдельных агрегатов, рассмотрите специализированные event stores.

---

## 3. CQRS с Kafka

### 3.1 Суть паттерна

**Command Query Responsibility Segregation** — разделение модели на:
- **Command side (запись):** валидирует команды, порождает события, отправляет в Kafka
- **Query side (чтение):** потребляет события, строит денормализованные проекции

**Аналогия:** банковская система. Кассир (command side) принимает депозит и записывает операцию в журнал. Приложение на телефоне (query side) показывает баланс из кэшированной проекции, а не вычисляет его каждый раз из журнала.

### 3.2 Реализация на Java + Spring + Kafka

**Command Service — публикует события:**

```java
@RestController
@RequestMapping("/api/orders")
public class OrderCommandController {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @PostMapping
    @Transactional("kafkaTransactionManager")
    public ResponseEntity<OrderCreatedEvent> createOrder(@Valid @RequestBody CreateOrderCommand cmd) {
        // 1. Валидация команды
        if (cmd.getItems().isEmpty()) {
            throw new InvalidCommandException("Order must contain at least one item");
        }

        // 2. Генерация события (command → event transformation)
        OrderCreatedEvent event = new OrderCreatedEvent(
            UUID.randomUUID().toString(),
            cmd.getCustomerId(),
            cmd.getItems(),
            Instant.now()
        );

        // 3. Публикация в Kafka (ключ = orderId для партицирования)
        kafkaTemplate.send("order-events", event.getOrderId(), event);

        return ResponseEntity.accepted().body(event);
    }
}
```

**Query Service — строит проекции:**

```java
@Service
public class OrderSearchProjection {

    private final ElasticsearchClient esClient;

    @KafkaListener(topics = "order-events", groupId = "search-projection")
    public void onOrderEvent(ConsumerRecord<String, OrderEvent> record) {
        OrderEvent event = record.value();

        switch (event.getType()) {
            case "OrderCreated":
                esClient.index(idx -> idx
                    .index("orders")
                    .id(event.getOrderId())
                    .document(new OrderSearchDoc(event)));
                break;

            case "OrderShipped":
                esClient.update(upd -> upd
                    .index("orders")
                    .id(event.getOrderId())
                    .doc(Map.of("status", "SHIPPED")));
                break;

            // Можно добавить новые проекции без изменения command side!
        }
    }
}

// Вторая проекция — аналитика (независимо от поисковой)
@KafkaListener(topics = "order-events", groupId = "analytics-projection")
public void onOrderForAnalytics(OrderEvent event) {
    analyticsService.recordOrderEvent(event);  // своя модель данных
}
```

### 3.3 CQRS на Python с отдельными сервисами

```python
# === Command Service ===
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    acks='all',
    enable_idempotence=True
)

def handle_create_order(command: dict) -> dict:
    """Обработчик команды. Валидирует и превращает в событие."""
    order_id = str(uuid.uuid4())
    event = {
        "type": "OrderCreated",
        "orderId": order_id,
        "customerId": command["customerId"],
        "items": command["items"],
        "timestamp": datetime.utcnow().isoformat()
    }
    # Ключ — orderId для детерминированного партицирования
    producer.send("order-events", key=order_id, value=event)
    producer.flush()
    return event


# === Query Service: проекция для пользовательского интерфейса ===
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'order-events',
    bootstrap_servers='localhost:9092',
    group_id='ui-projection',
    enable_auto_commit=False,
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Redis как хранилище проекций (ключ = order_id, значение = JSON)
import redis
cache = redis.Redis(host='localhost', port=6379, decode_responses=True)

for msg in consumer:
    event = msg.value
    order_id = event['orderId']

    if event['type'] == 'OrderCreated':
        cache.set(f"order:{order_id}", json.dumps({
            "id": order_id,
            "customerId": event['customerId'],
            "items": event['items'],
            "status": "CREATED"
        }))
    elif event['type'] == 'OrderShipped':
        order = json.loads(cache.get(f"order:{order_id}") or '{}')
        order['status'] = 'SHIPPED'
        cache.set(f"order:{order_id}", json.dumps(order))

    # Ручной коммит offset-а после успешной записи в проекцию
    consumer.commit()
```

**Ключевое преимущество CQRS на Kafka:** добавляйте новые проекции когда угодно. Новая consumer group просто начинает читать топик с начала (или с текущего offset-а). Command side не знает о существовании query side — это настоящий decoupling.

---

## 4. Change Data Capture (CDC)

### 4.1 Суть паттерна

**CDC** — захват изменений из транзакционного лога базы данных (WAL в PostgreSQL, binlog в MySQL, redo log в Oracle) и публикация их в Kafka без модификации прикладного кода.

**Аналогия:** охранная камера в магазине. Вы не спрашиваете каждого покупателя «что вы купили?», а просто смотрите запись. CDC так же наблюдает за БД.

### 4.2 Архитектура Debezium + Kafka

```
┌───────────┐     ┌────────────┐     ┌───────────┐     ┌──────────────┐
│ PostgreSQL│────▶│  Debezium  │────▶│   Kafka   │────▶│ Потребители  │
│  (WAL)    │     │  Connector │     │  Topics   │     │ (проекции,   │
└───────────┘     └────────────┘     └───────────┘     │  аналитика)  │
                                                       └──────────────┘
```

```json
// Пример конфигурации Debezium connector для PostgreSQL
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "debezium_secret",
    "database.dbname": "orders_db",
    "topic.prefix": "dbserver1",
    "table.include.list": "public.orders,public.order_items",
    "plugin.name": "pgoutput",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
  }
}
```

После настройки Debezium автоматически создаёт топики:
- `dbserver1.public.orders` — все изменения таблицы orders
- `dbserver1.public.order_items` — все изменения таблицы order_items

Формат сообщения (после unwrap-трансформации):

```json
{
  "id": 123,
  "customer_id": 456,
  "status": "SHIPPED",
  "amount": 1500.00,
  "updated_at": "2025-01-15T10:30:00Z"
}
```

### 4.3 Преимущества CDC перед написанием producer-ов вручную

| Ручной producer | CDC через Debezium |
|----------------|-------------------|
| Требует изменения кода приложения | Ноль изменений в коде |
| Две операции не атомарны (БД + Kafka) → риск рассинхрона | Гарантированная доставка (WAL — source of truth) |
| Нужно следить за всеми точками записи (ORM, миграции, ad-hoc SQL) | Все изменения захватываются на уровне БД |
| Высокий coupling с инфраструктурой | Приложение не знает о Kafka |

### 4.4 Outbox Pattern: CDC без прямого подключения к WAL

**Проблема:** не все БД поддерживают CDC (SQLite, старые версии MySQL). Нужно гарантировать атомарность «запись в БД + событие в Kafka» без распределённых транзакций.

**Решение: Outbox Pattern**

```sql
-- Добавляем outbox-таблицу
CREATE TABLE outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    published BOOLEAN DEFAULT FALSE
);
```

```java
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepo;
    private final OutboxRepository outboxRepo;

    // АТОМАРНО: оба INSERT в одной транзакции БД
    public Order createOrder(CreateOrderCommand cmd) {
        Order order = orderRepo.save(new Order(cmd));

        OutboxMessage outboxMsg = OutboxMessage.builder()
            .aggregateType("Order")
            .aggregateId(order.getId().toString())
            .eventType("OrderCreated")
            .payload(toJson(order))  // {"orderId": "...", "customerId": "...", ...}
            .build();
        outboxRepo.save(outboxMsg);

        return order;
    }
}
```

Теперь Debezium захватывает изменения из таблицы `outbox` и публикует их в Kafka. Если транзакция БД откатывается — outbox-запись тоже исчезает. Атомарность гарантирована на уровне СУБД.

**На Go с ручным polling (без Debezium):**

```go
// Outbox Poller — альтернатива CDC: периодически забирает записи из outbox-таблицы
func (p *OutboxPoller) Run(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            p.pollAndPublish()
        }
    }
}

func (p *OutboxPoller) pollAndPublish() {
    tx, _ := p.db.Begin()
    defer tx.Rollback()

    // SELECT ... FOR UPDATE SKIP LOCKED — конкурентный polling без блокировок
    rows, err := tx.Query(`
        SELECT id, aggregate_type, aggregate_id, event_type, payload
        FROM outbox
        WHERE published = FALSE
        ORDER BY created_at
        LIMIT 100
        FOR UPDATE SKIP LOCKED
    `)
    if err != nil {
        return
    }
    defer rows.Close()

    for rows.Next() {
        var msg OutboxMessage
        rows.Scan(&msg.ID, &msg.AggregateType, &msg.AggregateID,
                  &msg.EventType, &msg.Payload)

        topic := fmt.Sprintf("%s.events", strings.ToLower(msg.AggregateType))

        // Отправка в Kafka (ключ = aggregate_id для партицирования)
        p.producer.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{
                Topic: &topic,
                Partition: kafka.PartitionAny,
            },
            Key:   []byte(msg.AggregateID),
            Value: msg.Payload,
        }, nil)

        // Помечаем как опубликованное
        tx.Exec("UPDATE outbox SET published = TRUE WHERE id = $1", msg.ID)
    }

    tx.Commit()
    // Примечание: если Kafka-отправка упала — транзакция откатывается,
    // записи НЕ помечаются как published, и следующий poll их подхватит
}
```

---

## 5. Streaming ETL

### 5.1 Суть паттерна

**Streaming ETL** — извлечение, трансформация и загрузка данных в реальном времени (в отличие от batch-ETL, который работает по расписанию). Kafka выступает шиной данных.

Типичный pipeline:

```
Базы данных → CDC → Kafka (raw) → Kafka Streams (transform) → Kafka (enriched) → S3/ClickHouse/Elasticsearch
```

### 5.2 Пример: enrichment заказов данными о клиентах

```java
// Kafka Streams: join потока заказов с таблицей клиентов
StreamsBuilder builder = new StreamsBuilder();

// Поток заказов (ключ: customerId)
KStream<String, Order> orders = builder
    .stream("orders.raw", Consumed.with(Serdes.String(), orderSerde));

// Таблица клиентов (загружается из compacted-топика)
KTable<String, Customer> customers = builder
    .table("customers", Consumed.with(Serdes.String(), customerSerde));

// Join — обогащение заказа данными клиента
KStream<String, EnrichedOrder> enriched = orders
    .join(customers, (order, customer) -> EnrichedOrder.builder()
        .orderId(order.getId())
        .amount(order.getAmount())
        .customerName(customer.getName())
        .customerTier(customer.getTier())    // VIP, Regular — влияет на SLA
        .customerRegion(customer.getRegion()) // для гео-аналитики
        .processedAt(Instant.now())
        .build());

// Результат в выходной топик
enriched.to("orders.enriched");
```

### 5.3 Streaming ETL на Python: filter-transform-route

```python
from kafka import KafkaConsumer, KafkaProducer
import json

consumer = KafkaConsumer(
    'raw-events',
    bootstrap_servers='localhost:9092',
    group_id='etl-processor',
    enable_auto_commit=False,
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Три отдельных producer-а — маршрутизация по типам данных
producers = {
    'metrics': KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        compression_type='snappy'   # для high-volume метрик
    ),
    'alerts': KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        acks='all'                  # для критичных алертов
    ),
    'dead_letter': KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )
}

for msg in consumer:
    try:
        raw = msg.value

        # Трансформация: нормализация и валидация
        if raw.get('temperature', -999) > 1000:
            raise ValueError(f"Implausible temperature: {raw['temperature']}")

        event = {
            'sensor_id': raw['sensor_id'],
            'value': float(raw['value']),
            'unit': raw.get('unit', 'unknown').lower(),
            'timestamp': raw.get('ts', datetime.utcnow().isoformat()),
            'processed_by': 'etl-v2',
        }

        # Маршрутизация (routing): fan-out по типу события
        if raw.get('priority') == 'CRITICAL':
            producers['alerts'].send('alerts', key=event['sensor_id'], value=event)
        else:
            producers['metrics'].send('metrics', key=event['sensor_id'], value=event)

        consumer.commit()

    except Exception as e:
        # Dead Letter Queue — сохраняем проблемные сообщения для анализа
        producers['dead_letter'].send(
            'dead-letter',
            value={'error': str(e), 'original': raw, 'timestamp': datetime.utcnow().isoformat()}
        )
        consumer.commit()  # коммитим даже ошибки — не блокируем поток
```

**Шаблонные элементы Streaming ETL:**
- **Filter:** пропускаем только релевантные события
- **Transform:** enrichment, normalisation, schema mapping
- **Route:** направляем результат в нужные топики (fan-out)
- **Dead Letter Queue (DLQ):** сохраняем необработанные сообщения для расследования

---

## 6. Pub/Sub (Publish-Subscribe)

### 6.1 Суть паттерна

**Pub/Sub** — один producer публикует событие, множество consumer-ов получают его независимо. В Kafka это реализуется через consumer groups: каждая группа получает полную копию потока.

**Отличие от очереди сообщений (RabbitMQ):**
- **Очередь:** сообщение потребляется один раз и удаляется
- **Pub/Sub в Kafka:** событие хранится в логе и может быть прочитано сколько угодно раз разными группами

### 6.2 Мультисервисная нотификация: Node.js

```javascript
// === Producer: Order Service (публикует событие выполненного заказа) ===
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['localhost:9092'],
});

const producer = kafka.producer({
  maxInFlightRequests: 5,
  idempotent: true,       // KafkaJS 2.0+: предотвращает дубли при retry
  transactionalId: 'order-service-tx', // стабильный ID для EOS
});

await producer.connect();

// Публикация события — все подписчики получат копию
await producer.send({
  topic: 'order-completed',
  messages: [
    {
      key: orderId,
      value: JSON.stringify({
        type: 'OrderCompleted',
        orderId,
        customerId: 'cust-456',
        amount: 1500,
        timestamp: new Date().toISOString(),
      }),
    },
  ],
});


// === Consumer Group 1: Уведомления клиентов ===
const notificationConsumer = kafka.consumer({
  groupId: 'notification-service',
});

await notificationConsumer.connect();
await notificationConsumer.subscribe({ topic: 'order-completed', fromBeginning: false });

await notificationConsumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value.toString());
    await sendPushNotification(event.customerId, `Заказ ${event.orderId} готов!`);
    // Ручной коммит не требуется: KafkaJS по умолчанию auto-commit после обработки
  },
});


// === Consumer Group 2: Складская система (совершенно независимо) ===
const warehouseConsumer = kafka.consumer({
  groupId: 'warehouse-service',
});

await warehouseConsumer.connect();
await warehouseConsumer.subscribe({ topic: 'order-completed', fromBeginning: false });

await warehouseConsumer.run({
  eachBatchAutoResolve: false, // полный контроль над коммитом в батче
  eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning }) => {
    for (const message of batch.messages) {
      const event = JSON.parse(message.value.toString());
      await reserveInventory(event.orderId);
      resolveOffset(message.offset); // коммитим ПОСЛЕ успешной обработки
    }
  },
});
```

### 6.3 Pub/Sub: таблица ответственности

| Аспект | Реализация в Kafka |
|--------|-------------------|
| Один publisher → много subscribers | Consumer groups: каждая — отдельный «подписчик» |
| Независимая скорость обработки | Каждая группа — свой offset, своя скорость |
| Добавление нового подписчика | Новая consumer group + подписка на топик |
| Отказоустойчивость | Ребалансировка внутри группы при падении consumer-а |
| Replay старых событий | `auto.offset.reset=earliest` — перечитать с начала |

---

## 7. Best Practices для Producer

### 7.1 Чеклист надёжного продюсера

| # | Что | Почему |
|---|-----|--------|
| 1 | `acks=all` | Ждать подтверждения от всех in-sync реплик — ноль потерь данных |
| 2 | `enable.idempotence=true` | Предотвращает дубли при retry |
| 3 | `retries > 0` (с idempotence = MAX_INT) | Автоматическое восстановление после transient-ошибок |
| 4 | `max.in.flight.requests.per.connection=5` (с idempotence) | Пайплайнинг для throughput без потери порядка |
| 5 | Осмысленный ключ сообщения | Детерминированное партицирование + гарантия порядка для ключа |
| 6 | Сжатие (`compression.type=lz4` или `zstd`) | Сокращение сети и диска на 50-80% |
| 7 | `linger.ms=5-10` + `batch.size=65536+` | Баланс latency/throughput |
| 8 | Обработка ошибок в callback | Не терять контекст ошибки (retryable vs fatal) |
| 9 | Мониторинг: `record-send-rate`, `record-error-rate`, `compression-rate` | Видимость проблем ДО того, как они станут инцидентами |
| 10 | Schema Registry + Avro/Protobuf | Evolution схем без поломки consumer-ов |

### 7.2 Антипаттерны Producer

**❌ Антипаттерн 1: `acks=0` для важных данных**

```java
// ТАК НЕ НАДО: сообщения теряются при сбое брокера ДО репликации
props.put(ProducerConfig.ACKS_CONFIG, "0");
```

Правильно:

```java
// ТАК НАДО: данные доставлены, даже если брокер упадёт
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
```

**❌ Антипаттерн 2: retry БЕЗ idempotence**

```java
// ТАК НЕ НАДО: повторные попытки создают дубликаты!
props.put(ProducerConfig.RETRIES_CONFIG, 10);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, false); // ← без этого
```

Почему это проблема: producer отправляет batch, лидер-брокер принимает и пишет в лог, но соединение рвётся до отправки ack. Producer ретраит → брокер получает тот же batch второй раз. Без idempotence это два дублирующихся сообщения.

**❌ Антипаттерн 3: синхронная отправка для high-throughput**

```java
// ТАК НЕ НАДО: блокировка на каждом send() — один поток = один send в момент времени
for (Order o : orders) {
    producer.send(record).get();  // синхронный вызов
}
```

Правильно:

```java
// ТАК НАДО: асинхронный send + callback для обработки результата
for (Order o : orders) {
    producer.send(record, (metadata, exception) -> {
        if (exception != null) {
            log.error("Failed to send order {}", o.getId(), exception);
            metrics.errorCounter.increment();
        }
    });
}
// Не вызываем producer.flush() в цикле — даём batching-у работать
```

**❌ Антипаттерн 4: случайный или отсутствующий ключ**

```
// ТАК НЕ НАДО: все записи в случайные партиции → нельзя гарантировать порядок
producer.send(new ProducerRecord<>("topic", value));
```

Правильно — выбирать ключ осмысленно:
- Для агрегатов (orders, users) → ключ = ID агрегата
- Для CDC → ключ = primary key строки БД
- Для алертов и метрик → ключ = идентификатор источника

**❌ Антипаттерн 5: producer.flush() после каждого сообщения**

```java
// ТАК НЕ НАДО: каждый flush принудительно отправляет недозаполненный batch
for (int i = 0; i < 1_000_000; i++) {
    producer.send(new ProducerRecord<>("topic", message));
    producer.flush(); // ← убивает batching, throughput падает в 10x
}
```

---

## 8. Best Practices для Consumer

### 8.1 Чеклист надёжного консьюмера

| # | Что | Почему |
|---|-----|--------|
| 1 | Ручной коммит offset (`enable.auto.commit=false`) | Контроль: коммитим только после успешной обработки |
| 2 | `isolation.level=read_committed` (если producer транзакционный) | Не читаем aborted-сообщения |
| 3 | Идемпотентная обработка | Обработка может повториться (at-least-once) → результат должен быть одинаковым |
| 4 | `max.poll.records` под размер обработки | Не брать больше, чем можно обработать за `max.poll.interval.ms` |
| 5 | `max.poll.interval.ms` > времени обработки max.poll.records | Избегать spurious rebalance (consumer исключается из группы, если сердцебиение пришло, а poll() не вернулся) |
| 6 | Dead Letter Queue для poison messages | Не блокировать весь поток из-за одного битого сообщения |
| 7 | Graceful shutdown (через `consumer.wakeup()`) | Корректный финальный коммит offset, rebalance без потерь |
| 8 | Мониторинг consumer lag | Lag растёт → consumer не справляется → пора скейлиться |

### 8.2 Идемпотентный Consumer (at-least-once)

```java
/**
 * Паттерн идемпотентного consumer-а:
 * 1. Принимаем, что сообщение МОЖЕТ прийти повторно (at-least-once)
 * 2. Проверяем, не обработано ли оно уже
 * 3. Обрабатываем идемпотентно
 * 4. Коммитим offset ПОСЛЕ обработки
 */
@KafkaListener(topics = "payments", groupId = "payment-processor")
public void processPayment(ConsumerRecord<String, PaymentEvent> record) {
    PaymentEvent event = record.value();
    String idempotencyKey = event.getTransactionId(); // бизнес-ключ

    // Проверка: не обработано ли уже?
    if (paymentRepo.existsByIdempotencyKey(idempotencyKey)) {
        log.info("Duplicate event detected and skipped: {}", idempotencyKey);
        return; // идемпотентность: дубликат пропускается молча
    }

    // Обработка — идемпотентная операция
    try {
        paymentService.execute(event);

        // Записываем идемпотентный ключ (атомарно внутри execute() или отдельно)
        paymentRepo.saveIdempotencyKey(idempotencyKey, event);

    } catch (Exception e) {
        log.error("Processing failed for {}", idempotencyKey, e);
        throw e; // НЕ коммитим offset — обработаем повторно при перезапуске
    }
}
```

### 8.3 Graceful Shutdown Consumer

```java
@Component
public class GracefulConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final AtomicBoolean running = new AtomicBoolean(true);

    @EventListener(ApplicationReadyEvent.class)
    public void start() {
        consumer.subscribe(List.of("topic"));
        new Thread(this::consumeLoop).start();
    }

    @PreDestroy
    public void shutdown() {
        running.set(false);
        consumer.wakeup(); // прерывает poll() — выходим из обработки корректно
    }

    private void consumeLoop() {
        try {
            while (running.get()) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, String> r : records) {
                    processRecord(r);
                }
                if (!records.isEmpty()) {
                    consumer.commitSync(); // финальный коммит перед shutdown
                }
            }
        } catch (WakeupException e) {
            // wakeup() вызван — игнорируем, выходим из цикла
        } finally {
            consumer.commitSync(); // гарантированный коммит перед закрытием
            consumer.close();
            log.info("Consumer gracefully shut down");
        }
    }
}
```

### 8.4 Антипаттерны Consumer

**❌ Антипаттерн 1: Auto-commit (потеря данных)**

```java
// ТАК НЕ НАДО: Kafka закоммитил offset, а сообщение ещё не обработано.
// Consumer упал → сообщение потеряно.
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
```
→ Используйте `enable.auto.commit=false` и коммитьте вручную ПОСЛЕ обработки.

**❌ Антипаттерн 2: Бесконечная обработка в poll-цикле**

```java
// ТАК НЕ НАДО: poll-цикл не возвращается → брокер считает consumer мёртвым → rebalance
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> r : records) {
        callExternalApiWithNoTimeout(r); // зависает на минуты!
    }
}
```

Правильно: ограничить `max.poll.records` и обработку с таймаутом. Если API медленный — асинхронная обработка + коммит в фоне.

**❌ Антипаттерн 3: Игнорирование ошибок десериализации**

```java
// ТАК НЕ НАДО: битый байт-массив → ErrorDeserializer падает → consumer стопится
// и не может пройти дальше
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
    AvroDeserializer.class);
```

Правильно: использовать `ErrorHandlingDeserializer`:

```java
// ТАК НАДО: битое сообщение → dead-letter topic, consumer продолжает обработку
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
    ErrorHandlingDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS,
    AvroDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_FUNCTION, FailedRecordProcessor.class);
```

**❌ Антипаттерн 4: Слишком частый коммит (commit каждое сообщение)**

```java
// ТАК НЕ НАДО: commitSync() для каждого сообщения = одна операция записи в
// __consumer_offsets на каждый record → деградация throughput в 100-1000x
for (ConsumerRecord<String, String> r : records) {
    process(r);
    consumer.commitSync(); // ← синхронный коммит 1M раз для 1M сообщений
}
```

Правильно: коммитить батчами (каждые N сообщений или раз в секунду):

```java
int processed = 0;
for (ConsumerRecord<String, String> r : records) {
    process(r);
    if (++processed % 1000 == 0) {
        consumer.commitAsync(); // асинхронный коммит каждые 1000 сообщений
    }
}
consumer.commitSync(); // финальный синхронный коммит для батча
```

---

## 9. Распространённые архитектурные антипаттерны

### 9.1 Топик как очередь

**Описание:** создание топика с одним partition и одной consumer group — использование Kafka как RabbitMQ.

**Почему плохо:** теряются все преимущества Kafka (параллельное чтение, replay, fan-out). Если нужна очередь — берите RabbitMQ или NATS.

**Как исправить:** проектируйте топики с количеством partition, соответствующим параллелизму consumer-ов. Kafka-топик — это НЕ очередь, это распределённый лог.

### 9.2 Топик-на-таблицу (зеркалирование схемы БД)

**Описание:** каждый раз, когда в БД появляется новая таблица, создаётся соответствующий топик Kafka.

**Почему плохо:** десятки/сотни мелких топиков порождают operational complexity. Каждый partition требует ресурсов (файловые дескрипторы, память).

**Как исправить:** группируйте события по бизнес-доменам. Вместо `orders`, `order_items`, `order_payments` — один топик `order-events` с type-дискриминатором в payload.

### 9.3 Бесконечный retry без Dead Letter Queue

**Описание:** при ошибке обработки сообщения consumer делает retry и никогда не сдаётся.

**Почему плохо:** одно «ядовитое» сообщение блокирует всю партицию. Все последующие сообщения в этой партиции не могут быть обработаны (Kafka гарантирует порядок в пределах partition).

**Как исправить:** Dead Letter Queue + ограниченное число попыток:

```java
@Component
public class RetryableConsumer {

    private final DeadLetterPublisher dlq;
    private final Map<String, Integer> retryCount = new ConcurrentHashMap<>();

    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 5000, multiplier = 2.0),
        dltTopicSuffix = "-dead-letter"
    )
    @KafkaListener(topics = "payments")
    public void process(ConsumerRecord<String, String> record) {
        String id = record.key();
        int attempts = retryCount.getOrDefault(id, 0) + 1;

        if (attempts > 3) {
            dlq.publish("payments-dead-letter", record);
            retryCount.remove(id);
            return; // не бросаем исключение → offset коммитится → партиция идёт дальше
        }

        try {
            doActualProcessing(record);
            retryCount.remove(id);  // успех — сбрасываем счётчик
        } catch (TransientException e) {
            retryCount.put(id, attempts);
            throw e; // бросаем → retry через @RetryableTopic
        }
    }
}
```

### 9.4 Слишком много partition

**Описание:** создание топика с 1000 partition «на вырост».

**Почему плохо:** каждый partition = как минимум 1 файловый дескриптор (лидер) × replicationFactor. Rebalance занимает O(partitions × consumers). Метаданные кластера раздуваются.

**Как исправить:** правило — количество partition = max(throughput / per-partition-throughput, parallelism). Для большинства случаев 6-24 partition достаточно. Масштабируйте горизонтально (добавляйте partition) только когда измеренный throughput per partition достиг предела.

### 9.5 Log Compaction для Event Sourcing

**Описание:** использование `cleanup.policy=compact` для топика событий.

**Почему это катастрофа:** compaction оставляет только последнее значение для каждого ключа. Все промежуточные события удаляются — история теряется.

**Как исправить:** `retention.ms=-1`, `cleanup.policy=delete` для топиков событий. Compaction — ТОЛЬКО для снапшотов и KTable changelog-ов.

---

## 10. Идиоматичный код: паттерны на 4 языках

### 10.1 Java (Spring Kafka) — Transactional Outbox + CQRS

Полный пример выше (разделы 4.4 и 3.2).

### 10.2 Python (kafka-python) — At-Least-Once Consumer с идемпотентностью

```python
"""
Надёжный consumer: at-least-once, идемпотентная обработка,
Dead Letter Queue для poison messages.
"""
import json
from kafka import KafkaConsumer
import psycopg2
from datetime import datetime, timedelta

consumer = KafkaConsumer(
    'transactions',
    bootstrap_servers='localhost:9092',
    group_id='fraud-detector',
    enable_auto_commit=False,
    max_poll_records=500,
    max_poll_interval_ms=300_000,  # 5 минут на обработку батча
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    key_deserializer=lambda k: k.decode('utf-8') if k else None
)

conn = psycopg2.connect("dbname=fraud user=detector")
DLQ_TOPIC = 'transactions-dead-letter'

for msg in consumer:
    try:
        event = msg.value
        txn_id = event['transactionId']

        # Идемпотентность: проверяем, не обработано ли уже
        with conn.cursor() as cur:
            cur.execute(
                "SELECT 1 FROM processed_transactions WHERE transaction_id = %s",
                (txn_id,)
            )
            if cur.fetchone():
                continue  # уже обработано — пропускаем

        # Основная бизнес-логика: fraud detection
        score = fraud_model.predict(event)

        if score > 0.95:
            alert_fraud_team(event, score)

        # Запись результата (идемпотентно)
        with conn.cursor() as cur:
            cur.execute(
                """INSERT INTO processed_transactions (transaction_id, score, processed_at)
                   VALUES (%s, %s, %s)
                   ON CONFLICT (transaction_id) DO NOTHING""",
                (txn_id, score, datetime.utcnow())
            )
        conn.commit()

    except Exception as e:
        # Записываем в DLQ (через отдельную транзакцию БД — не Kafka producer)
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO dead_letter_queue (payload, error, created_at) VALUES (%s, %s, %s)",
                (json.dumps(event), str(e)[:1000], datetime.utcnow())
            )
        conn.commit()
        # ВАЖНО: коммитим offset даже для ошибок — не блокируем партицию

    finally:
        consumer.commit()  # ручной коммит после обработки батча
```

### 10.3 Go (confluent-kafka-go) — Producer с транзакциями

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
    "time"
)

type PaymentService struct {
    producer *kafka.Producer
}

func NewPaymentService(brokers string) (*PaymentService, error) {
    p, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers":   brokers,
        "enable.idempotence":  true,               // предотвращает дубли
        "acks":                "all",              // все ISR должны подтвердить
        "compression.type":    "zstd",             // zstd лучше сжимает, чем lz4 (но чуть выше CPU)
        "linger.ms":           5,                  // 5ms ожидания для заполнения батча
        "batch.size":          131072,             // 128KB batch
        "transactional.id":    "payment-processor", // включение транзакций
        "transaction.timeout.ms": 30000,
    })
    if err != nil {
        return nil, err
    }

    // Инициализация транзакций — регистрация в transaction coordinator
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := p.InitTransactions(ctx); err != nil {
        return nil, fmt.Errorf("initTransactions: %w", err)
    }

    return &PaymentService{producer: p}, nil
}

// ProcessPayment: атомарная запись в 3 партиции (2 топика) в рамках одной транзакции
func (s *PaymentService) ProcessPayment(payment Payment) error {
    ctx := context.Background()

    if err := s.producer.BeginTransaction(); err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    // Подготовка атомарных сообщений
    messages := []*kafka.Message{
        {
            TopicPartition: kafka.TopicPartition{
                Topic:     kafka.StringPtr("payment.processed"),
                Partition: kafka.PartitionAny,
            },
            Key:   []byte(payment.ID),
            Value: mustMarshal(PaymentProcessedEvent{
                PaymentID: payment.ID,
                Amount:    payment.Amount,
                Timestamp: time.Now(),
            }),
        },
        {
            TopicPartition: kafka.TopicPartition{
                Topic:     kafka.StringPtr("account.debited"),
                Partition: kafka.PartitionAny,
            },
            Key:   []byte(payment.AccountID),
            Value: mustMarshal(AccountDebitedEvent{
                AccountID: payment.AccountID,
                Amount:    payment.Amount,
                PaymentID: payment.ID,
            }),
        },
    }

    for _, msg := range messages {
        deliveryChan := make(chan kafka.Event, 1)
        if err := s.producer.Produce(msg, deliveryChan); err != nil {
            s.producer.AbortTransaction(ctx)
            return fmt.Errorf("produce: %w", err)
        }

        e := <-deliveryChan
        m := e.(*kafka.Message)
        if m.TopicPartition.Error != nil {
            s.producer.AbortTransaction(ctx)
            return fmt.Errorf("delivery failed: %w", m.TopicPartition.Error)
        }
    }

    // Commit транзакции — все сообщения становятся видны потребителям одновременно
    if err := s.producer.CommitTransaction(ctx); err != nil {
        // Если другая инстанция с тем же transactional.id взяла на себя работу,
        // это ProducerFenced — просто закрываемся, не делаем abort
        if err.(kafka.Error).Code() == kafka.ErrProducerFenced {
            s.producer.Close()
            return fmt.Errorf("producer fenced by newer instance: %w", err)
        }
        return fmt.Errorf("commit tx: %w", err)
    }

    return nil
}

func mustMarshal(v interface{}) []byte {
    b, _ := json.Marshal(v)
    return b
}
```

### 10.4 Node.js (KafkaJS) — Consumer Group с graceful shutdown

```javascript
const { Kafka } = require('kafkajs');
const { v4: uuid } = require('uuid');

const kafka = new Kafka({
  clientId: 'inventory-service',
  brokers: ['localhost:9092'],
  retry: {
    initialRetryTime: 100,
    retries: 8,
  },
});

const consumer = kafka.consumer({
  groupId: 'inventory-service',
  sessionTimeout: 30_000,
  heartbeatInterval: 3_000,
  maxBytesPerPartition: 1_048_576, // 1MB
});

const processed = new Set(); // в production — Redis/BД с TTL

const processOrder = async (message) => {
  const event = JSON.parse(message.value.toString());

  // Идемпотентность через бизнес-ключ
  const idempotencyKey = `${event.orderId}-${message.offset}`;
  if (processed.has(idempotencyKey)) {
    console.log(`Duplicate skipped: ${event.orderId}`);
    return;
  }

  // Бизнес-логика: резервирование товара на складе
  await inventoryService.reserve(event.orderId, event.items);
  processed.add(idempotencyKey);

  // Очистка старых ключей каждые 10 минут
  if (processed.size > 100_000) {
    processed.clear();
  }
};

async function main() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-completed', fromBeginning: false });

  // Graceful shutdown
  const shutdown = async () => {
    console.log('Shutting down consumer...');
    await consumer.disconnect();
    process.exit(0);
  };
  process.on('SIGTERM', shutdown);
  process.on('SIGINT', shutdown);

  await consumer.run({
    autoCommitInterval: 5000, // коммит каждые 5 секунд
    eachBatchAutoResolve: true,
    eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning }) => {
      for (const message of batch.messages) {
        if (!isRunning()) break; // shutdown во время обработки

        try {
          await processOrder(message);
          resolveOffset(message.offset);
        } catch (err) {
          console.error(`Failed to process ${message.key}:`, err.message);
          // НЕ resolveOffset → offset не коммитится → повторная обработка
        }

        await heartbeat(); // отправляем heartbeat, чтобы брокер не думал что consumer умер
      }
    },
  });
}

main().catch(console.error);
```

---

## 11. Таблица сравнения паттернов

| Характеристика | Event Sourcing | CQRS | CDC | Streaming ETL | Pub/Sub | Outbox |
|---------------|---------------|------|-----|---------------|---------|--------|
| **Модификация кода приложения** | Требуется | Требуется | Не требуется | Требуется | Требуется | Требуется |
| **Source of Truth** | Kafka | Kafka + БД проекций | БД (WAL) | Исходные системы | Producer-ы | БД |
| **Сложность внедрения** | Высокая | Средняя | Низкая (Debezium) | Средняя | Низкая | Средняя |
| **Аудит и история** | Полная (все события) | Только в командной модели | Полная (WAL-level) | По желанию | Только текущие события | Как настроить |
| **Задержка (latency)** | Миллисекунды | Миллисекунды | Секунды (Debezium snapshot + streaming) | Миллисекунды | Миллисекунды | Зависит от polling-интервала |
| **Идемпотентность** | Естественная (события неизменяемы) | Требуется в проекциях | Требуется | Требуется | Требуется (consumer) | Требуется (consumer) |
| **Минимальный масштаб** | Средний (10+ events/sec) | Любой | Любой (Debezium = 1 connector) | Высокий (1000+ events/sec) | Любой | Любой |

---

## 12. Дерево решений: какой паттерн выбрать?

```
Вам нужно гарантировать, что запись в БД и событие в Kafka атомарны?
├── ДА → Outbox Pattern (раздел 4.4)
└── НЕТ → Идём дальше
    │
    Вам нужна полная история всех изменений (аудит, temporal queries)?
    ├── ДА → Event Sourcing (раздел 2)
    └── НЕТ → Идём дальше
        │
        Чтение и запись имеют принципиально разную нагрузку/модель?
        ├── ДА → CQRS (раздел 3)
        └── НЕТ → Идём дальше
            │
            Нужно получать изменения из существующей БД БЕЗ модификации кода?
            ├── ДА → CDC (раздел 4)
            └── НЕТ → Идём дальше
                │
                Нужна потоковая трансформация/обогащение данных между системами?
                ├── ДА → Streaming ETL (раздел 5)
                └── НЕТ → Идём дальше
                    │
                    Одно событие → множество независимых потребителей?
                    ├── ДА → Pub/Sub (раздел 6)
                    └── НЕТ → Возможно, вам не нужна Kafka. Рассмотрите REST API или gRPC.
```

---

## 13. Заключение

Шесть паттернов вокруг Kafka закрывают практически любой сценарий работы с потоковыми данными. Ключевые takeaways:

1. **Event Sourcing + CQRS** — мощная комбинация для систем, где важны аудит и асимметричное масштабирование чтения/записи. Но требует дисциплины: infinite retention для событий, снапшоты для агрегатов, идемпотентные consumer-ы.

2. **CDC (Debezium + Kafka)** — наименьший coupling с приложением. Данные из БД текут в Kafka автоматически. Идеален для микросервисов: каждый сервис читает проекцию своих данных, не завязываясь на API других сервисов.

3. **Outbox Pattern** — когда CDC недоступен, а атомарность БД+Kafka критична. Запись в outbox-таблицу внутри транзакции БД, затем асинхронная публикация в Kafka.

4. **Producer best practices:** `acks=all`, `enable.idempotence=true`, сжатие, осмысленный ключ. Не жертвуйте durability ради скорости.

5. **Consumer best practices:** `enable.auto.commit=false`, идемпотентная обработка, Dead Letter Queue, graceful shutdown. Lag — ваш главный production-метрик.

6. **Антипаттерны, которых нужно избегать как огня:** compaction для event sourcing, бесконечный retry без DLQ, топик как очередь, слишком много partition, auto-commit для критичных данных.

Эти паттерны не взаимоисключающи. Реальная production-система часто комбинирует их: CDC для захвата изменений из БД, Streaming ETL для обогащения, CQRS для разделения чтения/записи, Pub/Sub для доставки событий множеству consumer-ов. Главное — понимать границы каждого паттерна и не применять их «потому что модно».

---

## Источники

1. Conduktor — «Event sourcing with Kafka: patterns and pitfalls», Stéphane Derosiaux, July 2025. [conduktor.io/blog/event-sourcing-kafka-patterns-pitfalls](https://www.conduktor.io/blog/event-sourcing-kafka-patterns-pitfalls). Tier 2: блог вендора инструментов Kafka.

2. CsCode.io — «Kafka Transactions & Idempotent Producers — Deep Dive», 2025. [cscode.io/kafka/KafkaTransactionsDeep/](https://cscode.io/kafka/KafkaTransactionsDeep/). Tier 2: техническое руководство с кодом.

3. ActiveWizards — «Kafka: Producer & Consumer Best Practices», 2025. [activewizards.com/blog/kafka-producer-and-consumer-best-practices](https://activewizards.com/blog/kafka-producer-and-consumer-best-practices). Tier 2: engineering-компания, специализирующаяся на data platforms.

4. Red Hat Developer — «The outbox pattern with Apache Kafka and Debezium», Don Schenck, September 2021. [developers.redhat.com/articles/2021/09/01/outbox-pattern-apache-kafka-and-debezium](https://developers.redhat.com/articles/2021/09/01/outbox-pattern-apache-kafka-and-debezium). Tier 1: официальный блог разработчиков Red Hat.

5. Debezium Blog — «Reliable Microservices Data Exchange With the Outbox Pattern», Gunnar Morling, February 2019. [debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/). Tier 1: официальный блог проекта Debezium.

6. Streamkap — «Exactly-Once Delivery: How It Actually Works Under the Hood», May 2025. [streamkap.com/resources-and-guides/exactly-once-delivery-how-it-works](https://streamkap.com/resources-and-guides/exactly-once-delivery-how-it-works). Tier 2: специализированный ресурс по stream processing.

7. Confluent Developer — «Event Sourcing and Event Storage with Apache Kafka», курс. [developer.confluent.io/courses/event-sourcing/](https://developer.confluent.io/courses/event-sourcing/). Tier 1: официальный образовательный ресурс Confluent.

8. AxonOps — «Kafka Design Patterns» и «Kafka Anti-Patterns». [docs.axonops.com/data-platforms/kafka/application-development/patterns/](https://docs.axonops.com/data-platforms/kafka/application-development/patterns/) и [docs.axonops.com/data-platforms/kafka/application-development/anti-patterns/](https://docs.axonops.com/data-platforms/kafka/application-development/anti-patterns/). Tier 2: документация от операторов production-кластеров Kafka.

9. Apache Kafka Documentation — «Producer Configs», «Consumer Configs», «Message Delivery Guarantees», официальная документация Apache Kafka 4.0. [kafka.apache.org/documentation/](https://kafka.apache.org/documentation/). Tier 1: официальная документация проекта.

10. GitHub: debezium/debezium-examples — «Outbox Pattern». [github.com/debezium/debezium-examples/blob/main/outbox/README.md](https://github.com/debezium/debezium-examples/blob/main/outbox/README.md). Tier 1: официальные примеры Debezium.

11. Thorben Janssen — «Implementing the Outbox Pattern with CDC using Debezium». [thorben-janssen.com/outbox-pattern-with-cdc-and-debezium](https://thorben-janssen.com/outbox-pattern-with-cdc-and-debezium). Tier 2: технический блог (Hibernate/JPA эксперт).

12. Собственные знания автора — архитектура Kafka, семантика доставки (at-most-once, at-least-once, exactly-once), паттерны проектирования распределённых систем на Kafka Streams. Верифицировано через официальную документацию и практический опыт.

---

*Статья 16/37 в серии «Apache Kafka: Полное руководство». Следующая статья: [04-testing.md](04-testing.md) — тестирование Kafka-приложений.*
