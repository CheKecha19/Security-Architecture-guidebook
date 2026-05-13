# Тестирование приложений Apache Kafka: Unit, Integration, Mock и Load-тестирование

> **Нижняя строка:** Тестирование Kafka-приложений требует четырёхуровневой стратегии: юнит-тесты через MockProducer/MockConsumer (быстро, изолированно), TopologyTestDriver для Kafka Streams (детерминированно, без брокера), интеграционные тесты через Testcontainers или EmbeddedKafka (реальный брокер) и нагрузочное тестирование через kafka-producer-perf-test/Trogdor. Золотое правило: не поднимайте брокер ради теста сериализации JSON — начинайте с моков и эскалируйте к контейнерам только когда нужно реальное поведение кластера. Типичный CI-пайплайн, построенный по этому принципу, сокращается с 18 до 3 минут.

---

## 1. Пирамида тестирования для Kafka: 4 уровня

Тестирование распределённых систем сложнее синхронного кода. Сообщения не падают немедленно, оффсеты коммитятся в фоне, ребалансировка происходит асинхронно. Чтобы не утонуть в нестабильных тестах (flaky tests), нужна чёткая стратификация.

**Аналогия:** представьте, что вы тестируете аэропорт. Юнит-тесты — проверка каждого конвейера сортировки багажа отдельно. Интеграционные — запуск всего терминала с настоящими чемоданами, но на полигоне. Нагрузочные — симуляция часов пик с тысячами пассажиров. E2E — пробный рейс с реальными людьми.

### 1.1 Четыре уровня и их характеристики

| Уровень | Инструмент | Скорость | Когда использовать |
|---------|------------|----------|-------------------|
| **Unit (Mock)** | MockProducer, MockConsumer | ~1ms | Сериализация, маршрутизация, бизнес-логика обработки |
| **Unit (Streams)** | TopologyTestDriver | ~10ms | Топологии Kafka Streams, фильтрация, агрегация, join-ы |
| **Integration** | Testcontainers, EmbeddedKafka | 5-15s | End-to-end пайплайны, Schema Registry, транзакции, ребалансировка |
| **Load / Chaos** | kafka-producer-perf-test, Trogdor | Минуты-часы | Пропускная способность, отказоустойчивость, сетевая деградация |

### 1.2 Правило 80/20 распределения

На практике большинство команд перегружает интеграционные тесты. Оптимальное распределение:

```
60% Unit-тестов (MockProducer/MockConsumer + TopologyTestDriver)
25% Интеграционных тестов (Testcontainers)
10% Контрактных тестов (Schema Registry, wire-формат)
 5% Нагрузочных тестов (в CI — облегчённые smoke-тесты)
```

**Антипаттерн:** «У нас 200 тестов и все поднимают Testcontainers — они идут 45 минут, и никто не запускает их локально».

---

## 2. Unit-тестирование: MockProducer и MockConsumer

### 2.1 MockProducer: тестирование продюсера без брокера

Kafka предоставляет `MockProducer<K, V>` прямо в `kafka-clients` — без дополнительных зависимостей. Он перехватывает вызовы `send()`, сохраняет отправленные записи в `history()` и позволяет симулировать ошибки через `errorNext()`.

#### 2.1.1 Базовый тест отправки

```java
import org.apache.kafka.clients.producer.MockProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

@Test
void shouldSendOrderToCorrectTopic() {
    // given: MockProducer в autoComplete=true — send() завершается немедленно
    MockProducer<String, String> mockProducer = new MockProducer<>(
        true,                                // autoComplete
        new StringSerializer(),              // key-сериализатор
        new StringSerializer()               // value-сериализатор
    );

    OrderService service = new OrderService(mockProducer);
    
    // when: отправляем заказ через бизнес-логику
    service.placeOrder("order-123", "{\"item\": \"widget\"}");

    // then: проверяем, что сообщение ушло в правильный топик
    assertEquals(1, mockProducer.history().size());
    ProducerRecord<String, String> sent = mockProducer.history().get(0);
    assertEquals("orders", sent.topic());
    assertEquals("order-123", sent.key());
}
```

**Что проверяет:** сериализацию ключа/значения, правильность имени топика, наличие заголовков, корректность ключа партицирования (через `partition()`).

#### 2.1.2 Тестирование обработки ошибок

Главное преимущество MockProducer — возможность внедрять ошибки детерминированно:

```java
@Test
void shouldRetryOnTransientFailure() {
    // autoComplete=false → send() не завершается автоматически
    MockProducer<String, String> mockProducer = new MockProducer<>(
        false,                               // autoComplete=false
        new StringSerializer(),
        new StringSerializer()
    );

    OrderService service = new OrderService(mockProducer);

    // when: асинхронная отправка
    service.placeOrderAsync("order-123", "{}");

    // внедряем ошибку таймаута
    mockProducer.errorNext(new TimeoutException("Broker not available"));
    // завершаем pending-запрос с ошибкой — это триггерит retry-логику
    mockProducer.completeNext();

    // then: должно быть 2 сообщения в history (оригинал + ретрай)
    assertEquals(2, mockProducer.history().size());

    // verify: ошибка не привела к потере — финальное сообщение успешно
    mockProducer.completeNext(); // завершаем ретрай успехом
    // проверяем callback-и через assertDoesNotThrow
}
```

**Ключевой момент:** `autoComplete=false` + `completeNext()` даёт контроль над тем, когда и как завершается `send()`. Это позволяет тестировать retry-логику, dead-letter очереди, fallback-стратегии — всё, что происходит при сбоях.

#### 2.1.3 Тестирование идемпотентного продюсера

Идемпотентный продюсер (см. [core-concepts](../../02-basics/03-core-concepts.md)) требует проверки детерминированности ключей:

```java
@Test
void shouldAssignConsistentPartitionForSameKey() {
    MockProducer<String, String> mockProducer = new MockProducer<>(
        true, new StringSerializer(), new StringSerializer()
    );

    PaymentService service = new PaymentService(mockProducer);
    service.process("user-42", "{\"amount\": 100}");
    service.process("user-42", "{\"amount\": 200}");

    List<ProducerRecord<String, String>> history = mockProducer.history();
    assertEquals(history.get(0).partition(), history.get(1).partition(),
        "Одна и та же партиция для одного ключа — гарантия порядка");
}
```

**Ограничения MockProducer:**
- Не тестирует реальную сериализацию на уровне брокера (Schema Registry не участвует)
- Не симулирует ребалансировку
- Не проверяет сетевые таймауты на уровне TCP
- Не тестирует взаимодействие с ACL

> **Правило:** MockProducer — для бизнес-логики, НЕ для вопроса «работает ли мой конфиг?»

### 2.2 MockConsumer: тестирование консьюмера без брокера

`MockConsumer<K, V>` сложнее MockProducer-а — вы должны вручную управлять состоянием подписки, оффсетами и циклами poll.

#### 2.2.1 Базовый тест потребления

```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.MockConsumer;
import org.apache.kafka.clients.consumer.OffsetResetStrategy;

@Test
void shouldProcessOrderFromKafka() {
    // given: создаём MockConsumer с политикой earliest
    MockConsumer<String, String> mockConsumer = new MockConsumer<>(
        OffsetResetStrategy.EARLIEST
    );

    // назначаем партицию вручную (MockConsumer не ходит в брокер)
    TopicPartition tp = new TopicPartition("orders", 0);
    Map<TopicPartition, Long> beginningOffsets = Collections.singletonMap(tp, 0L);
    mockConsumer.assign(Collections.singletonList(tp));
    mockConsumer.updateBeginningOffsets(beginningOffsets);

    // добавляем записи во внутреннюю очередь
    mockConsumer.addRecord(
        new ConsumerRecord<>("orders", 0, 0L, "order-123", "{\"item\":\"widget\"}")
    );

    // подписываемся (после assign)
    mockConsumer.subscribe(Collections.singletonList("orders"));

    // when: вызываем бизнес-логику, которая делает poll()
    OrderProcessor processor = new OrderProcessor(mockConsumer);
    processor.processBatch();

    // then: проверяем результат обработки
    assertEquals(1, processor.getProcessedCount());
}
```

**Аналогия:** MockConsumer — это «ручной конвейер». Вы сами кладёте записи на ленту, сами управляете скоростью, сами решаете, когда остановить. Полный контроль ценой большего объёма setup-кода.

#### 2.2.2 Тестирование seek-операций и оффсет-менеджмента

```java
@Test
void shouldSeekToBeginningOnRebalance() {
    MockConsumer<String, String> mockConsumer = new MockConsumer<>(
        OffsetResetStrategy.EARLIEST
    );

    TopicPartition tp = new TopicPartition("orders", 0);
    mockConsumer.assign(Collections.singletonList(tp));
    mockConsumer.updateBeginningOffsets(Map.of(tp, 0L));
    mockConsumer.updateEndOffsets(Map.of(tp, 100L));

    // when: добавляем записи и эмулируем ребалансировку (seek)
    mockConsumer.addRecord(new ConsumerRecord<>("orders", 0, 50L, "k", "v"));
    mockConsumer.seek(tp, 0L); // ← эмуляция перемотки на начало

    // then: position сброшен
    assertEquals(0L, mockConsumer.position(tp));
}
```

#### 2.2.3 Тестирование graceful shutdown

```java
@Test
void shouldStopGracefullyOnShutdownSignal() throws InterruptedException {
    MockConsumer<String, String> consumer = new MockConsumer<>(OffsetResetStrategy.EARLIEST);
    consumer.assign(Collections.singletonList(new TopicPartition("orders", 0)));

    // given: большой бэклог
    for (int i = 0; i < 1000; i++) {
        consumer.addRecord(new ConsumerRecord<>("orders", 0, i, "k", "{}"));
    }

    AtomicInteger processed = new AtomicInteger(0);
    consumer.schedulePollTask(() -> {
        // эмулируем shutdown-флаг после обработки первых 100 записей
        if (processed.incrementAndGet() > 100) {
            consumer.wakeup(); // прерываем poll()
        }
    });

    // when/then: не зависает на бесконечном poll-е
    assertDoesNotThrow(() -> {
        OrderProcessor processor = new OrderProcessor(consumer);
        processor.processUntilShutdown();
    });

    // проверяем: обработано >100 записей (часть батча)
    assertTrue(processed.get() > 100);
    assertTrue(processed.get() < 1000); // не все — остановились gracefully
}
```

### 2.3 Сравнение MockProducer vs MockConsumer

| Характеристика | MockProducer | MockConsumer |
|---------------|-------------|-------------|
| Сложность setup-а | Низкая | Средняя |
| Контроль ошибок | `errorNext()` | Ручное управление |
| Имитация брокера | Нет | Нет |
| Тестирование транзакций | Частично (через `transactionAbortedAt`) | Частично |
| Тестирование retry | Да | Нет (нужен ручной control loop) |
| Подходит для | Producer-логики, сериализации, error handling | Consumer-логики, seek, graceful shutdown |

---

## 3. Unit-тестирование Kafka Streams: TopologyTestDriver

### 3.1 Зачем нужен TopologyTestDriver

Kafka Streams требует работающего брокера для выполнения топологий. Тестовый драйвер заменяет брокер полностью: он синхронно прогоняет записи через топологию, захватывает вывод в `TestOutputTopic` и даёт доступ к state stores. **Никакого брокера, никакой сети, никакой асинхронности.**

Зависимость (Maven):
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams-test-utils</artifactId>
    <version>4.0.2</version>
    <scope>test</scope>
</dependency>
```

**Аналогия:** TopologyTestDriver — это аэродинамическая труба. Вы прогоняете отдельные детали (записи) через тестируемый воздушный поток (топологию) и смотрите, как они себя поведут, не поднимая в воздух весь самолёт (кластер).

### 3.2 Базовый тест: фильтрация

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.test.TestRecord;
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class OrderFilterTopologyTest {

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, Order> inputTopic;
    private TestOutputTopic<String, Order> outputTopic;

    @BeforeEach
    void setUp() {
        // 1. Строим топологию (как в production, но изолированно)
        StreamsBuilder builder = new StreamsBuilder();
        builder.<String, Order>stream("orders-raw")
            .filter((key, order) -> order.getAmount() > 0)
            .to("orders-valid");

        Properties props = new Properties();
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, OrderSerde.class.getName());

        // 2. Создаём драйвер
        testDriver = new TopologyTestDriver(builder.build(), props);

        // 3. Создаём тестовые топики
        inputTopic = testDriver.createInputTopic(
            "orders-raw",
            Serdes.String().serializer(),
            new OrderSerde().serializer()
        );
        outputTopic = testDriver.createOutputTopic(
            "orders-valid",
            Serdes.String().deserializer(),
            new OrderSerde().deserializer()
        );
    }

    @AfterEach
    void tearDown() {
        testDriver.close();
    }

    @Test
    void shouldFilterNegativeAmountOrders() {
        // given: негативная сумма
        inputTopic.pipeInput("order-1", new Order(-50));

        // then: output-топик пуст — запись отфильтрована
        assertThat(outputTopic.isEmpty()).isTrue();
    }

    @Test
    void shouldPassValidOrders() {
        // given: корректный заказ
        inputTopic.pipeInput("order-1", new Order(100));

        // then: запись прошла фильтр
        assertThat(outputTopic.getQueueSize()).isEqualTo(1);
        assertThat(outputTopic.readValue().getAmount()).isEqualTo(100);
    }
}
```

### 3.3 Тестирование агрегаций и state stores

```java
@Test
void shouldAggregateOrdersByCustomer() {
    // given: топология, считающая сумму заказов по customerId
    StreamsBuilder builder = new StreamsBuilder();
    builder.<String, Order>stream("orders")
        .groupByKey()
        .aggregate(
            () -> 0L,
            (key, order, total) -> total + order.getAmount(),
            Materialized.<String, Long>as(Stores.inMemoryKeyValueStore("totals"))
                     .withKeySerde(Serdes.String())
                     .withValueSerde(Serdes.Long())
        );

    Properties props = new Properties();
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, OrderSerde.class.getName());

    TopologyTestDriver driver = new TopologyTestDriver(builder.build(), props);
    TestInputTopic<String, Order> input = driver.createInputTopic(
        "orders", Serdes.String().serializer(), new OrderSerde().serializer()
    );

    // when: три заказа от одного customer
    input.pipeInput("customer-1", new Order(100)); // 100
    input.pipeInput("customer-1", new Order(200)); // 300
    input.pipeInput("customer-1", new Order(300)); // 600

    // then: state store содержит ожидаемое значение
    KeyValueStore<String, Long> store = driver.getKeyValueStore("totals");
    assertThat(store.get("customer-1")).isEqualTo(600L);

    driver.close();
}
```

### 3.4 Тестирование пунктуаций (Punctuations)

`TopologyTestDriver` даёт полный контроль над временем:

```java
@Test
void shouldPunctuateOnWallClockTime() {
    // given: топология с WALL_CLOCK_TIME-пунктуацией каждые 60 секунд
    StreamsBuilder builder = new StreamsBuilder();
    builder.<String, Long>stream("scores")
        .process(MyPunctuatingProcessor::new);

    TopologyTestDriver driver = new TopologyTestDriver(builder.build(), props);
    TestInputTopic<String, Long> input = driver.createInputTopic(
        "scores", Serdes.String().serializer(), Serdes.Long().serializer()
    );

    // when: отправляем запись
    input.pipeInput("key1", 42L);

    // then: пунктуация ещё не сработала (прошло < 60 сек)
    TestOutputTopic<String, Long> output = driver.createOutputTopic(
        "result", Serdes.String().deserializer(), Serdes.Long().deserializer()
    );
    assertThat(output.isEmpty()).isTrue();

    // when: продвигаем wall-clock на 60 секунд
    driver.advanceWallClockTime(Duration.ofSeconds(60));

    // then: пунктуация сработала
    assertThat(output.isEmpty()).isFalse();
    assertThat(output.readValue()).isEqualTo(42L);

    driver.close();
}
```

**Два типа пунктуаций:**
- `STREAM_TIME` (event-time) — срабатывают автоматически по временны́м меткам записей
- `WALL_CLOCK_TIME` — требуют явного вызова `advanceWallClockTime()`

### 3.5 Pre-populating state stores

```java
@Test
void shouldUpdatePrePopulatedStore() {
    // given: топология с state store
    // ...

    TopologyTestDriver driver = new TopologyTestDriver(topology, props);

    // pre-populate: загружаем начальное состояние ДО теста
    KeyValueStore<String, Long> store = driver.getKeyValueStore("store-name");
    store.put("key1", 100L);  // начальное значение

    // when: обрабатываем новую запись
    inputTopic.pipeInput("key1", 50L);

    // then: проверяем, что логика обновила store правильно
    assertThat(store.get("key1")).isEqualTo(150L); // 100 + 50

    driver.close();
}
```

### 3.6 Анти-паттерны TopologyTestDriver

| Антипаттерн | Последствия |
|------------|------------|
| Тестировать всю production-топологию как единое целое | Тесты становятся нечитаемыми, сложно понять какая часть сломалась |
| Забывать `driver.close()` | Утечка памяти, неочищенные временные файлы RocksDB |
| Использовать реальные Serdes с внешними зависимостями | Замедление тестов, ложные падения из-за network I/O |
| Смешивать event-time и wall-clock-time в одном тесте | Неожиданный порядок пунктуаций |
| Использовать `Thread.sleep()` вместо `advanceWallClockTime()` | Тесты становятся медленными и flaky |

---

## 4. Интеграционное тестирование: Testcontainers

### 4.1 Когда моков недостаточно — нужен реальный брокер

Есть сценарии, которые **невозможно** протестировать через моки:

- Ребалансировка consumer group
- Retention и compaction — реальная очистка сегментов на диске
- Взаимодействие с Schema Registry (Avro/Protobuf-сериализация с проверкой схемы)
- Транзакционная запись produce+offsets commit атомарно
- ACL и авторизация
- Межброкерная репликация
- KRaft election

Для этого используется Testcontainers — библиотека, поднимающая настоящие Docker-контейнеры с Kafka во время тестов.

**Аналогия:** Testcontainers — это тестовый полигон. Не компьютерная симуляция, а настоящий автомобиль на настоящем треке, но в контролируемой среде без риска для окружающих.

### 4.2 Подключение зависимостей

Maven:
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.20.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.20.0</version>
    <scope>test</scope>
</dependency>
```

### 4.3 Базовый Java-тест с Testcontainers

```java
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import static org.assertj.core.api.Assertions.*;
import static org.awaitility.Awaitility.*;

@Testcontainers
class OrderPipelineIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("apache/kafka:3.9.0")
    );

    @Test
    void shouldProduceAndConsumeEndToEnd() throws Exception {
        // given: properties с динамическим портом контейнера
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        // ... остальные настройки ...

        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");

        // when: отправляем сообщение
        try (KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {
            producer.send(new ProducerRecord<>("orders", "order-1", "{}")).get();
        }

        // then: потребляем
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singletonList("orders"));

            await().atMost(Duration.ofSeconds(10))
                .untilAsserted(() -> {
                    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                    assertThat(records.count()).isGreaterThan(0);
                });
        }
    }
}
```

**Ключевой момент:** Всегда используйте `kafka.getBootstrapServers()` для получения динамического порта. Никогда не хардкодьте `localhost:9092` — в CI порт будет случайным.

### 4.4 Тестирование с Schema Registry

Schema Registry требует сети между контейнерами:

```java
@Testcontainers
class SchemaRegistryIntegrationTest {

    static Network network = Network.newNetwork();

    @Container
    static GenericContainer<?> kafka = new GenericContainer<>(
        DockerImageName.parse("confluentinc/cp-kafka:7.8.0")
    )
        .withNetwork(network)
        .withNetworkAliases("kafka")
        .withEnv("KAFKA_LISTENERS", "BROKER://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093")
        .withEnv("KAFKA_LISTENER_SECURITY_PROTOCOL_MAP", "BROKER:PLAINTEXT,CONTROLLER:PLAINTEXT")
        .withEnv("KAFKA_INTER_BROKER_LISTENER_NAME", "BROKER")
        .withEnv("KAFKA_PROCESS_ROLES", "broker,controller")
        .withEnv("KAFKA_NODE_ID", "1")
        .withEnv("KAFKA_CONTROLLER_QUORUM_VOTERS", "1@kafka:9093")
        .withEnv("KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR", "1");

    @Container
    static GenericContainer<?> schemaRegistry = new GenericContainer<>(
        DockerImageName.parse("confluentinc/cp-schema-registry:7.8.0")
    )
        .withNetwork(network)
        .withNetworkAliases("schemaregistry")
        .withEnv("SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS", "kafka:9092")
        .withEnv("SCHEMA_REGISTRY_HOST_NAME", "schemaregistry")
        .withEnv("SCHEMA_REGISTRY_LISTENERS", "http://0.0.0.0:8081")
        .dependsOn(kafka);

    @Test
    void shouldProduceAndValidateAvroSchema() {
        String srUrl = "http://" + schemaRegistry.getHost() + ":" + schemaRegistry.getMappedPort(8081);
        // ... тест с Avro-сериализатором, указывающим на srUrl ...
    }
}
```

### 4.5 Ожидание асинхронных результатов: Awaitility

**Никогда не используйте `Thread.sleep()`** в тестах — это делает тесты медленными и flaky (падения при перегрузке CI).

```java
import static org.awaitility.Awaitility.*;

@Test
void shouldEventuallyReceiveAllRecords() {
    // ... setup ...

    // ✅ Правильно: Awaitility с timeout и polling interval
    await()
        .atMost(Duration.ofSeconds(10))      // максимальное ожидание
        .pollInterval(Duration.ofMillis(100)) // частота опроса
        .untilAsserted(() -> {
            assertThat(results.size()).isEqualTo(10);
        });

    // ❌ Неправильно:
    // Thread.sleep(5000);
    // assertThat(results.size()).isEqualTo(10);
}
```

### 4.6 Multi-broker кластер в Testcontainers

```java
// Три брокера для тестирования репликации и отказоустойчивости
static Network network = Network.newNetwork();

static List<GenericContainer<?>> brokers = IntStream.rangeClosed(1, 3)
    .mapToObj(id -> new GenericContainer<>(
        DockerImageName.parse("apache/kafka:3.9.0")
    )
        .withNetwork(network)
        .withNetworkAliases("broker-" + id)
        .withEnv("KAFKA_NODE_ID", String.valueOf(id))
        .withEnv("KAFKA_PROCESS_ROLES", "broker,controller")
        .withEnv("KAFKA_CONTROLLER_QUORUM_VOTERS",
            "1@broker-1:9093,2@broker-2:9093,3@broker-3:9093")
        // ...
    ).collect(Collectors.toList());

@Test
void shouldReplicateAcrossThreeBrokers() {
    // Тест: отправляем с RF=3, убиваем один брокер, проверяем доступность
}
```

### 4.7 Testcontainers Performance Tips

| Совет | Эффект |
|-------|--------|
| Pre-pull образы в CI (`docker pull apache/kafka:3.9.0`) | Избегаем timeout на первом старте |
| Singleton container — переиспользовать между тестами | Экономия 5-10 секунд на тест |
| `.withReuse(true)` — контейнер живёт между запусками | Радикальная экономия (но требует cleanup) |
| `withStartupTimeout(Duration.ofMinutes(3))` | Страховка от медленного CI |
| Выделять общий `Network` для Schema Registry | Избегаем конфликтов портов |

### 4.8 Проблемы и решения

| Проблема | Причина | Решение |
|----------|---------|---------|
| «Connection refused on random port» | Неправильное свойство bootstrap-servers | Использовать `bootstrapServersProperty` для переопределения |
| Container startup timeout | Образ не загружен в CI | Pre-pull или увеличить timeout до 3 минут |
| «Address already in use» | Конфликт портов | Не хардкодить порты, использовать динамические |
| Тесты идут слишком долго | Каждый тест поднимает новый контейнер | Singleton container / `@TestInstance(PER_CLASS)` |

---

## 5. EmbeddedKafka (Spring Boot)

### 5.1 Что это и когда использовать

Spring Kafka предоставляет `@EmbeddedKafka` — встроенный брокер, запускаемый в том же JVM-процессе. Быстрее Testcontainers (3-5s vs 5-10s), но менее реалистичен.

Начиная с Kafka 4.0, `EmbeddedKafka` использует **только KRaft** (без ZooKeeper) — класс `EmbeddedKafkaKraftBroker`.

**Аналогия:** EmbeddedKafka — это тренажёр водителя в автошколе. Безопасно, быстро, но вы не на настоящей дороге.

### 5.2 Базовый Spring Boot тест

```java
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.test.utils.KafkaTestUtils;

@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"orders"},
    bootstrapServersProperty = "spring.kafka.bootstrap-servers"
)
class OrderListenerTest {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private OrderProcessor processor;

    @Test
    void shouldProcessOrder() throws Exception {
        // given: отправляем сообщение
        kafkaTemplate.send("orders", "order-123", "{\"item\":\"widget\"}").get();

        // then: ждём обработки (Awaitility)
        await().atMost(Duration.ofSeconds(10))
            .until(() -> processor.getProcessedCount() > 0);
    }
}
```

### 5.3 Использование KafkaTestUtils

```java
import static org.springframework.kafka.test.utils.KafkaTestUtils.*;

// Получение consumer-пропертей в одну строку
Map<String, Object> consumerProps = consumerProps("testGroup", "false", embeddedKafka);

// Создание consumer-а
DefaultKafkaConsumerFactory<Integer, String> cf = new DefaultKafkaConsumerFactory<>(consumerProps);
Consumer<Integer, String> consumer = cf.createConsumer();

// Потребление всех embedded-топиков
embeddedKafka.consumeFromAllEmbeddedTopics(consumer);

// Получение одной записи
ConsumerRecord<Integer, String> record = getSingleRecord(consumer, "orders");

// Получение всех записей
ConsumerRecords<Integer, String> records = getRecords(consumer);
```

### 5.4 Глобальный EmbeddedKafka (JUnit Platform)

Начиная с Spring Kafka 3.0, можно использовать один глобальный брокер на весь test plan:

```
# application.properties или JUnit Platform конфигурация
spring.kafka.global.embedded.enabled=true
spring.kafka.embedded.count=1
spring.kafka.embedded.ports=0
spring.kafka.embedded.topics=orders,payments,notifications
spring.kafka.embedded.partitions=3
spring.kafka.embedded.broker.properties.location=classpath:embedded-broker.properties
```

Это запускает брокер **один раз** на весь тестовый запуск и выключает в конце — радикальная экономия времени.

### 5.5 EmbeddedKafka vs Testcontainers: сравнительная таблица

| Характеристика | @EmbeddedKafka | Testcontainers |
|---------------|----------------|----------------|
| Время старта | ~2-5s | ~5-15s |
| Требуется Docker | Нет | Да |
| Реалистичность | 85% (тот же код, но in-process) | 100% (настоящий контейнер) |
| Schema Registry | Только через mock | Поддержка реального контейнера |
| Multi-broker | Сложно (ручное управление) | Легко |
| Интеграция со Spring | Нативная | Ручная (DynamicPropertySource) |
| KRaft (Kafka 4.0+) | Только KRaft | Поддержка всех режимов |
| Подходит для | Простых Spring-тестов | Schema Registry, мульти-брокер, «честных» интеграционных тестов |

---

## 6. Нагрузочное тестирование: kafka-producer-perf-test и Trogdor

### 6.1 kafka-producer-perf-test: CLI-бенчмарк продюсера

Встроенный в дистрибутив Kafka инструмент для измерения throughput и latency продюсера:

```bash
# Базовый тест: 1M сообщений по 100 байт, 10 потоков
bin/kafka-producer-perf-test.sh \
    --topic perf-test \
    --num-records 1000000 \
    --record-size 100 \
    --throughput -1 \           # -1 = без ограничения (максимально быстро)
    --producer-props \
        bootstrap.servers=localhost:9092 \
        acks=all \
        compression.type=lz4 \
        linger.ms=5 \
        batch.size=65536
```

**Вывод:**

```
1000000 records sent, 234567.890 records/sec (22.36 MB/sec),
418.5 ms avg latency, 923.0 ms max latency,
45 ms 50th, 120 ms 95th, 250 ms 99th, 800 ms 99.9th.
```

**Ключевые метрики:**
- `records/sec` — пропускная способность (основной показатель)
- `MB/sec` — пропускная способность в байтах
- `avg latency` — средняя задержка (не очень информативна)
- `p50/p95/p99/p99.9` — перцентили задержки (гораздо важнее среднего)

### 6.2 kafka-consumer-perf-test: бенчмарк консьюмера

```bash
bin/kafka-consumer-perf-test.sh \
    --topic perf-test \
    --messages 1000000 \
    --bootstrap-server localhost:9092 \
    --group perf-consumer-group \
    --show-detailed-stats
```

**Вывод:**

```
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec,
rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2024-01-15 10:00:00, 2024-01-15 10:00:10, 95.37, 9.5370, 1000000, 100000.0000,
120, 9880, 9.6542, 101214.5749
```

### 6.3 Trogdor: фреймворк нагрузочного и chaos-тестирования

**Trogdor** — встроенный в Apache Kafka фреймворк для:
- Нагрузочных тестов (ProduceBench, ConsumeBench, RoundTripWorkload)
- Chaos-инжиниринга: сетевая партиция, остановка процесса, задержки

**Архитектура:** Coordinator (управляет задачами) + Agent (по одному на узел, исполняет нагрузку/фаулты).

#### 6.3.1 Запуск

```bash
# Запуск Agent-а
./bin/trogdor.sh agent -c ./config/trogdor.conf -n node0 &

# Запуск Coordinator-а
./bin/trogdor.sh coordinator -c ./config/trogdor.conf -n node0 &
```

#### 6.3.2 Тест пропускной способности продюсера

Спецификация задачи (`produce-bench.json`):
```json
{
  "class": "org.apache.kafka.trogdor.workload.ProduceBenchSpec",
  "durationMs": 10000000,
  "producerNode": "node0",
  "bootstrapServers": "localhost:9092",
  "targetMessagesPerSec": 50000,
  "maxMessages": 500000,
  "activeTopics": {
    "perf[1-3]": {
      "numPartitions": 10,
      "replicationFactor": 1
    }
  },
  "keyGenerator": {
    "type": "sequential",
    "size": 8,
    "offset": 1
  }
}
```

Отправка задачи и проверка результатов:
```bash
# Отправка задачи
./bin/trogdor.sh client createTask -t localhost:8889 -i produce-bench \
    --spec ./produce-bench.json

# Проверка статуса
./bin/trogdor.sh client showTask -t localhost:8889 -i produce-bench

# Результаты с детализацией
./bin/trogdor.sh client showTask -t localhost:8889 -i produce-bench --show-status
```

**Ответ:**
```json
{
  "totalSent": 500000,
  "averageLatencyMs": 17.83,
  "p50LatencyMs": 12,
  "p95LatencyMs": 75,
  "p99LatencyMs": 96,
  "transactionsCommitted": 0
}
```

#### 6.3.3 Round-Trip: Producer + Consumer end-to-end

`RoundTripWorkload` — самый комплексный тест: один агент одновременно производит и потребляет сообщения, измеряя сквозную задержку:

```json
{
  "class": "org.apache.kafka.trogdor.workload.RoundTripWorkloadSpec",
  "durationMs": 300000,
  "clientNode": "node0",
  "bootstrapServers": "localhost:9092",
  "targetMessagesPerSec": 10000,
  "maxMessages": 100000,
  "activeTopics": {
    "roundtrip": {
      "numPartitions": 5,
      "replicationFactor": 1
    }
  }
}
```

#### 6.3.4 Chaos Engineering: сетевая партиция

```json
{
  "class": "org.apache.kafka.trogdor.fault.NetworkPartitionFaultSpec",
  "startMs": 1000,
  "durationMs": 30000,
  "partitions": [["node1", "node2"], ["node3"]]
}
```

Это изолирует `node1` и `node2` от `node3` через iptables на 30 секунд — тест на поведение кластера при split-brain.

**Другие fault-типы:**
- `ProcessStopFault` — отправка SIGSTOP процессу брокера (пауза, не убийство)
- `ExternalCommandWorker` — запуск произвольного скрипта/программы для реализации custom fault-ов

#### 6.3.5 Exec-режим: быстрый запуск без Coordinator-а

Для одиночных тестов:
```bash
./bin/trogdor.sh agent -n node0 -c ./config/trogdor.conf \
    --exec ./tests/spec/simple_produce_bench.json
```

Agent выполняет задачу и завершается — удобно для CI-пайплайнов.

### 6.4 Методология нагрузочного тестирования

**План тестирования (checklist):**

1. **Baseline:** пустой кластер, 1 продюсер, 1 партиция, `acks=1` — снимаем «потолок»
2. **Acks impact:** те же параметры с `acks=all` — измеряем накладные расходы репликации
3. **Partition scaling:** фиксированный throughput, варьируем количество партиций (1 → 10 → 100)
4. **Message size sweep:** фиксированный records/sec, варьируем размер (100b → 1KB → 10KB → 1MB)
5. **Compression:** сравниваем `none` vs `lz4` vs `snappy` vs `zstd`
6. **Consumer lag:** параллельный producer + consumer — измеряем consumer lag через `kafka-consumer-groups`
7. **Graceful degradation:** убиваем 1 из 3 брокеров, проверяем throughput
8. **Recovery:** возвращаем брокер, проверяем время восстановления

### 6.5 Интерпретация результатов

| Метрика | Что считается «хорошо» (на современном железе) |
|---------|-----------------------------------------------|
| Producer throughput | 1M+ msg/sec (100-байтные сообщения, 1 партиция) |
| Consumer throughput | 2M+ msg/sec (с одной партиции, без обработки) |
| End-to-end latency (p99) | <100ms (при нагрузке до saturation point) |
| ISR shrink time | <5s после отказа брокера |
| Recovery time | <30s для восстановления ISR после возврата брокера |

> **Важно:** не относитесь к этим цифрам как к абсолютной истине. Они зависят от железа, дисков (NVMe vs HDD), сети (10G vs 1G) и конфигурации. Всегда бенчмаркайте **на своём окружении**.

---

## 7. Тестирование транзакций и Exactly-Once семантики

### 7.1 Что тестировать

Транзакции Kafka (idempotent producer + transactional API, см. [core-concepts](../../02-basics/03-core-concepts.md)) создают уникальные сложности для тестирования:

1. **Атомарность:** все сообщения в транзакции либо закоммичены, либо нет
2. **Изоляция:** незакоммиченные сообщения не видны консьюмерам с `isolation.level=read_committed`
3. **Idempotency:** повторная отправка одного и того же сообщения не создаёт дубликатов

### 7.2 Интеграционный тест транзакционного продюсера

```java
@Test
void shouldAtomicallyCommitMultipleMessages() {
    // given: транзакционный продюсер
    Properties props = new Properties();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-test-1");
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

    // консьюмер с read_committed
    Properties consumerProps = new Properties();
    consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
    consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
    consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");

    try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
        producer.initTransactions();

        // Транзакция 1: commit
        producer.beginTransaction();
        producer.send(new ProducerRecord<>("tx-test", "key1", "msg1"));
        producer.send(new ProducerRecord<>("tx-test", "key2", "msg2"));
        producer.commitTransaction();

        // Транзакция 2: abort
        producer.beginTransaction();
        producer.send(new ProducerRecord<>("tx-test", "key3", "aborted-msg"));
        producer.abortTransaction();
    }

    // then: read_committed консьюмер видит только msg1 и msg2
    try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
        consumer.subscribe(Collections.singletonList("tx-test"));
        // seek to beginning для получения всех сообщений
        consumer.poll(Duration.ofMillis(100));
        consumer.seekToBeginning(consumer.assignment());

        List<String> received = new ArrayList<>();
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                consumer.poll(Duration.ofMillis(500))
                    .forEach(r -> received.add(r.value()));
                assertThat(received).containsExactlyInAnyOrder("msg1", "msg2");
                assertThat(received).doesNotContain("aborted-msg");
            });
    }
}
```

### 7.3 Тестирование consume-process-produce цикла

```java
@Test
void shouldExactlyOnceProcessFromInputToOutput() {
    // Паттерн: читаем из input, обрабатываем, пишем в output — атомарно
    Properties consumerProps = getTransactionalConsumerProps(kafka.getBootstrapServers());
    Properties producerProps = getTransactionalProducerProps(kafka.getBootstrapServers());

    // given: pre-populate input topic
    try (KafkaProducer<String, String> preloader = new KafkaProducer<>(getPlainProducerProps())) {
        preloader.send(new ProducerRecord<>("input", "k1", "10")).get();
        preloader.send(new ProducerRecord<>("input", "k2", "20")).get();
    }

    // when: транзакционный consume-process-produce
    try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
         KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {

        producer.initTransactions();
        consumer.subscribe(Collections.singletonList("input"));

        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));

        producer.beginTransaction();
        for (ConsumerRecord<String, String> r : records) {
            int value = Integer.parseInt(r.value());
            // process: умножаем на 2
            producer.send(new ProducerRecord<>("output", r.key(), String.valueOf(value * 2)));
        }

        // атомарно коммитим оффсеты + output
        producer.sendOffsetsToTransaction(
            getConsumerOffsets(consumer), consumer.groupMetadata()
        );
        producer.commitTransaction();
    }

    // then: проверяем output
    // ... read from output topic, assert values are 20 and 40
}
```

---

## 8. Тестирование за пределами Java: Python, Go, Node.js

### 8.1 Python: confluent-kafka-python + Testcontainers

`confluent-kafka-python` (обёртка над librdkafka) — не предоставляет встроенных MockProducer/MockConsumer. Стратегия:

```python
import pytest
from confluent_kafka import Producer, Consumer
from testcontainers.kafka import KafkaContainer

@pytest.fixture(scope="module")
def kafka_broker():
    """Testcontainers: один контейнер на модуль"""
    kafka = KafkaContainer()
    kafka.start()
    yield kafka.get_bootstrap_server()
    kafka.stop()

class TestOrderProducer:
    def test_should_send_order(self, kafka_broker):
        """Интеграционный тест — реальный брокер"""
        producer = Producer({'bootstrap.servers': kafka_broker})
        # ... send and verify via consumer

    def test_should_retry_on_failure(self, monkeypatch):
        """Юнит-тест через monkeypatching — mock вместо интеграции"""
        # Подменяем Producer.produce()
        calls = []
        def mock_produce(topic, value, key=None, on_delivery=None, **kwargs):
            calls.append({
                'topic': topic,
                'value': value
            })
        
        monkeypatch.setattr('confluent_kafka.Producer.produce', mock_produce)
        
        from myapp import OrderService
        service = OrderService()
        service.send_order("order-1", {"item": "widget"})
        
        assert len(calls) == 1
        assert calls[0]['topic'] == 'orders'
```

**Python-альтернативы:**
- `pytest-kafka` — фикстуры для ZooKeeper, Kafka server, Kafka consumer (использует локальный бинарник Kafka)
- `kafka-mocha` — mock-библиотека для librdkafka, эмулирующая брокер

### 8.2 Go: Testcontainers + интерфейсы

В Go удобный паттерн — интерфейс для producer/consumer:

```go
type MessageProducer interface {
    Send(topic string, key, value []byte) error
}

type KafkaProducer struct {
    producer sarama.SyncProducer
}

func (kp *KafkaProducer) Send(topic string, key, value []byte) error {
    _, _, err := kp.producer.SendMessage(&sarama.ProducerMessage{
        Topic: topic, Key: sarama.ByteEncoder(key), Value: sarama.ByteEncoder(value),
    })
    return err
}

// В тестах:
type MockProducer struct {
    Messages []ProducerMessage
}

func (mp *MockProducer) Send(topic string, key, value []byte) error {
    mp.Messages = append(mp.Messages, ProducerMessage{Topic: topic, Key: key, Value: value})
    return nil
}

func TestOrderService(t *testing.T) {
    mock := &MockProducer{}
    service := NewOrderService(mock)
    service.PlaceOrder("order-1", []byte(`{"item":"widget"}`))

    assert.Equal(t, 1, len(mock.Messages))
    assert.Equal(t, "orders", mock.Messages[0].Topic)
}
```

### 8.3 Node.js: kafka.js + Testcontainers

```typescript
import { Kafka } from 'kafkajs';
import { KafkaContainer, StartedKafkaContainer } from '@testcontainers/kafka';

describe('OrderService', () => {
    let kafkaContainer: StartedKafkaContainer;
    let kafka: Kafka;

    beforeAll(async () => {
        kafkaContainer = await new KafkaContainer().start();
        kafka = new Kafka({
            brokers: [kafkaContainer.getBootstrapServer()],
        });
    }, 60000); // увеличенный timeout для первого pull-а

    afterAll(async () => {
        await kafkaContainer.stop();
    });

    it('should produce and consume order', async () => {
        const producer = kafka.producer();
        const consumer = kafka.consumer({ groupId: 'test-group' });

        await producer.connect();
        await consumer.connect();
        await consumer.subscribe({ topic: 'orders', fromBeginning: true });

        // Отправка
        await producer.send({
            topic: 'orders',
            messages: [{ key: 'order-1', value: JSON.stringify({ item: 'widget' }) }],
        });

        // Проверка
        const messages: any[] = [];
        await consumer.run({
            eachMessage: async ({ message }) => {
                messages.push(JSON.parse(message.value!.toString()));
            },
        });

        // Ждём (Awaitility-эквивалент для Node.js)
        await new Promise(resolve => setTimeout(resolve, 2000));
        expect(messages).toHaveLength(1);

        await producer.disconnect();
        await consumer.disconnect();
    });
});
```

---

## 9. Чеклисты тестирования

### 9.1 Что тестировать в продюсере

- [ ] Сериализация ключа и значения (в том числе с Schema Registry)
- [ ] Маршрутизация в правильный топик
- [ ] Партицирование (одинаковый ключ → одинаковая партиция)
- [ ] Обработка ошибок: retry на TimeoutException, fallback на не-retryable ошибки
- [ ] Идемпотентность (нет дубликатов при повторной отправке)
- [ ] Транзакционность (атомарный commit/abort)
- [ ] Batching: `linger.ms` и `batch.size` корректно группируют сообщения
- [ ] Compression: сообщения сжимаются ожидаемым алгоритмом
- [ ] Callback: `onCompletion()` вызывается для каждого сообщения

### 9.2 Что тестировать в консьюмере

- [ ] Десериализация (включая schema evolution)
- [ ] Обработка сообщений (бизнес-логика)
- [ ] Graceful shutdown (завершение обработки текущего батча при `wakeup()`)
- [ ] Offset-менеджмент (ручной commit vs auto-commit)
- [ ] Rebalance handling: `onPartitionsRevoked`, `onPartitionsAssigned`
- [ ] Poison pill handling: сообщения, которые невозможно обработать (DLQ)
- [ ] Idempotency consumer-а (повторная обработка одного сообщения не ломает состояние)
- [ ] Retry логика: retry с backoff при transient failure

### 9.3 Что тестировать в Kafka Streams

- [ ] Правильность топологии (filter, map, flatMap, groupBy, join)
- [ ] Состояние state stores после обработки
- [ ] Пунктуации (wall-clock-time и event-time)
- [ ] Join-ы (KStream-KStream, KStream-KTable, KTable-KTable)
- [ ] Windowed агрегации (tumbling, hopping, sliding, session)
- [ ] Repartitioning (сообщения правильно перераспределяются)

### 9.4 Антипаттерны тестирования Kafka

| Антипаттерн | Почему плохо | Исправление |
|------------|-------------|-------------|
| `Thread.sleep()` в тестах | Flaky + медленно | Awaitility |
| Хардкод `localhost:9092` | Ломается в CI (рандомные порты) | `kafka.getBootstrapServers()` |
| Один брокер-контейнер на тест | 200 тестов × 5s = 16 минут CI | Singleton container |
| Интеграционные тесты для сериализации | Медленно, не даёт дополнительной информации | Unit-тест с MockProducer |
| Тестировать всю топологию одним тестом | Непонятно что сломалось | Разбить на unit-тесты отдельных processor-ов |
| Игнорировать cleanup контейнеров | Утечка ресурсов в CI | `@AfterAll` / `@AfterEach` с `.stop()` |
| Не тестировать негативные сценарии | Сюрпризы в production | Mock-тесты с error injection |

---

## 10. CI/CD интеграция

### 10.1 Рекомендуемая структура CI-пайплайна

```
Stage 1: Unit Tests (2 min)
├── MockProducer/MockConsumer тесты
├── TopologyTestDriver тесты
└── Junit 5 параллельно, forkCount=1C

Stage 2: Integration Tests (8 min)
├── Testcontainers с apache/kafka:3.9.0 (pre-pulled)
├── Schema Registry (при необходимости)
└── Awaitility-based assertion

Stage 3: Smoke Load Tests (10 min) — опционально, только на PR в main
├── kafka-producer-perf-test: 10M сообщений baseline
└── Проверка throughput > порогового значения

Stage 4: Chaos Tests (15 min) — только nightly / pre-release
├── Trogdor produce + consume benchmark
└── Network partition fault → graceful degradation
```

### 10.2 GitHub Actions пример

```yaml
name: Kafka Tests
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      - run: mvn test -P unit-tests

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      # Pre-pull образы для избежания timeout
      - run: docker pull apache/kafka:3.9.0
      - run: docker pull confluentinc/cp-schema-registry:7.8.0
      - run: mvn test -P integration-tests
```

---

## 11. Сводка: стратегия выбора инструмента тестирования

### Дерево решений

```
Нужно протестировать Kafka-код
│
├─ Это логика продюсера/консьюмера БЕЗ взаимодействия с брокером?
│  └─ ДА → MockProducer / MockConsumer
│     ├─ Проверяем сериализацию, маршрутизацию, retry
│     └─ < 1ms на тест, детерминированно
│
├─ Это топология Kafka Streams?
│  └─ ДА → TopologyTestDriver
│     ├─ Проверяем фильтрацию, агрегацию, join-ы, пунктуации
│     └─ < 10ms на тест, доступ к state stores
│
├─ Нужен Schema Registry, транзакции, ребалансировка?
│  └─ ДА → Testcontainers
│     ├─ Реальный брокер в Docker
│     ├─ ~5-10s на старт контейнера
│     └─ Максимальная реалистичность
│
├─ Spring Boot, простые тесты без Schema Registry?
│  └─ ДА → @EmbeddedKafka
│     ├─ In-process брокер, быстрый старт (~3s)
│     └─ Нативная интеграция со Spring
│
└─ Нужно измерить throughput / chaos-stability?
   └─ ДА → kafka-producer-perf-test / Trogdor
      ├─ Реалистичные бенчмарки
      ├─ Fault injection (network partition, process stop)
      └─ Минуты выполнения, только для performance/chaos инженерии
```

---

## Ссылки на связанные статьи

- [Клиентские библиотеки Kafka](../01-client-libraries.md) — обзор клиентов по языкам
- [API-справочник Kafka](../02-api-reference.md) — wire protocol, операции, коды ошибок
- [Паттерны использования Kafka](../03-patterns.md) — Event Sourcing, CQRS, CDC и другие архитектурные паттерны
- [Core Concepts: Delivery Guarantees](../../02-basics/03-core-concepts.md) — Exactly-Once семантика, идемпотентность, транзакции
- [Как работает Kafka](../../02-basics/02-how-it-works.md) — внутреннее устройство: продюсер, консьюмер, репликация

---

## Источники

1. Conduktor, «Testing Kafka Applications: Testcontainers, Embedded Kafka, and Mocks», 2024 — conduktor.io/blog/testing-kafka-testcontainers-embedded-mocks
2. Apache Kafka, «Testing a Streams Application», официальная документация, v4.0 — kafka.apache.org/40/streams/developer-guide/testing
3. Spring Kafka, «Testing Applications», официальная документация, v4.0.5 — docs.spring.io/spring-kafka/reference/testing.html
4. Testcontainers, «Kafka Module», официальная документация — testcontainers.com/modules/kafka
5. Confluent, «Advanced Testing Techniques for Spring Kafka», 2023 — confluent.io/de-de/blog/advanced-testing-techniques-for-spring-kafka
6. Docker Docs, «Write tests with Testcontainers (Kafka + Spring Boot)», 2024 — docs.docker.com/guides/testcontainers-java-spring-boot-kafka/write-tests
7. Testcontainers, «Testing Spring Boot Kafka Listener using Testcontainers» — testcontainers.com/guides/testing-spring-boot-kafka-listener-using-testcontainers
8. Apache Kafka, «Trogdor README», GitHub, trunk — github.com/apache/kafka/blob/trunk/trogdor/README.md
9. Confluent, «Testing Kafka Streams», обучающий курс — developer.confluent.io/courses/kafka-streams/testing
10. Baeldung, «Using Kafka MockProducer», 2024 — baeldung.com/kafka-mockproducer
11. Baeldung, «Using Kafka MockConsumer» — baeldung.com/kafka-mockconsumer
12. GitHub, «Apache Kafka PR #21100: MockProducer and MockConsumer testing examples» — github.com/apache/kafka/pull/21100
13. GitHub, «Apache Kafka PR #20361: Command-line arguments for producer perf test» — github.com/apache/kafka/pull/20361
14. pytest-kafka, официальная документация v0.8.1 — pytest-kafka.readthedocs.io/en/latest/readme.html
15. Total Shift Left, «Testing Kafka-Based Microservices: Complete Guide (2026)» — totalshiftleft.ai/blog/testing-kafka-based-microservices
