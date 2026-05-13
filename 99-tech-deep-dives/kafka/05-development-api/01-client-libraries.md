# Клиентские библиотеки Apache Kafka: полный обзор по языкам

> **Ключевой вывод:** Клиентская библиотека — это «нервное окончание» вашего приложения, через которое оно общается с кластером Kafka. Выбор правильной библиотеки критически важен: он определяет производительность, надёжность и поддерживаемость вашего кода. На 2026 год экосистема клиентов Kafka охватывает 7 основных языков с официальной поддержкой Confluent (Java, C/C++, Python, Go, .NET, JavaScript/Node.js, Rust — через librdkafka), а также десятки community-библиотек для Scala, Elixir, PHP, Ruby и других языков. **Главное правило:** если у языка есть биндинг к librdkafka — используйте его, это самый производительный, зрелый и поддерживаемый вариант за пределами Java-экосистемы.

---

## 1. Архитектура клиентских библиотек Kafka

Клиентские библиотеки Kafka можно разделить на три архитектурных типа:

| Тип | Описание | Примеры | Производительность | Поддержка фич |
|-----|----------|---------|-------------------|---------------|
| **Native (JVM)** | Чистая Java-реализация, входит в Apache Kafka | `kafka-clients` (org.apache.kafka) | ★★★★★ | Полная (producer, consumer, streams, connect, admin) |
| **librdkafka-based** | C-обёртка над нативной C/C++ библиотекой | confluent-kafka-python, confluent-kafka-go, confluent-kafka-dotnet, rdkafka (Rust) | ★★★★★ | Producer, Consumer, Admin (Streams и Connect — только в Java) |
| **Pure-language** | Полная реализация протокола Kafka на целевом языке | franz-go (Go), rust-rdkafka (частично), kafka.js (Node.js) | ★★★★☆ | Зависит от зрелости (от базового producer/consumer до транзакций) |

**Почему librdkafka доминирует:** C-библиотека librdkafka от Магнуса Эденхилла (Magnus Edenhill), впоследствии перешедшая под крыло Confluent — это «золотой стандарт» для всех не-JVM языков. Она использует zero-copy, page cache, асинхронный event-driven I/O и содержит более 10 лет оптимизаций. Все официальные клиенты Confluent для Python, Go, .NET, JavaScript и Rust — это обёртки (bindings) над librdkafka.

Единственное исключение из правила «всегда используй librdkafka» — Java, где нативный клиент `kafka-clients` является эталонным.

---

## 2. Сравнительная таблица клиентских библиотек (Feature Parity)

| Возможность | Java | Python (confluent-kafka) | Go (franz-go) | Go (confluent-kafka) | .NET | Node.js (kafka.js) | Node.js (confluent-kafka) | Rust (rdkafka) | C/C++ (librdkafka) |
|---|---|---|---|---|---|---|---|---|---|
| **Producer** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Consumer (group)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Admin API** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Idempotent Producer** | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ ограничено | ✅ | ✅ | ✅ |
| **Transactions** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Exactly-Once Semantics** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Kafka Streams** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Kafka Connect** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Schema Registry** | ✅ | ✅ | ✅ (через franz-go) | ✅ | ✅ | ⚠️ через rest | ✅ | ⚠️ через rest | ✅ (через C++) |
| **Compression** | ✅ gzip/snappy/lz4/zstd | ✅ все | ✅ все | ✅ все | ✅ все | ✅ gzip/snappy/lz4 | ✅ все | ✅ все | ✅ все |
| **SASL (все типы)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **mTLS** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **KRaft-aware** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **KIP-848 (new consumer protocol)** | ✅ 4.0+ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Share Consumer (KIP-932)** | ✅ 4.1+ | ⏳ roadmap | ❌ | ⏳ roadmap | ⏳ roadmap | ❌ | ❌ | ❌ | ✅ |
| **Async/await native** | ❌ (thread-based) | ❌ (callback-based) | ✅ native Go | ❌ | ✅ async/await | ✅ async/await | ✅ async/await | ✅ futures/tokio | ❌ |
| **Лицензия** | Apache 2.0 | Apache 2.0 | BSD-3 | Apache 2.0 | Apache 2.0 | MIT | Apache 2.0 | MIT | Apache 2.0 |
| **Репозиторий GitHub** | [apache/kafka](https://github.com/apache/kafka) | [confluentinc/confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python) | [twmb/franz-go](https://github.com/twmb/franz-go) | [confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go) | [confluentinc/confluent-kafka-dotnet](https://github.com/confluentinc/confluent-kafka-dotnet) | [tulios/kafkajs](https://github.com/tulios/kafkajs) | [confluentinc/confluent-kafka-javascript](https://github.com/confluentinc/confluent-kafka-javascript) | [fede1024/rust-rdkafka](https://github.com/fede1024/rust-rdkafka) | [confluentinc/librdkafka](https://github.com/confluentinc/librdkafka) |

> ⚠️ = частичная поддержка, ⏳ = в разработке, ❌ = не поддерживается

---

## 3. Совместимость версий (Version Compatibility)

### 3.1. Базовое правило совместимости клиент-брокер

Apache Kafka поддерживает **обратную совместимость клиентов**: более старый клиент может работать с более новым брокером (но без новых фич). Основные правила:

| Направление | Совместимость | Детали |
|---|---|---|
| **Старый клиент → Новый брокер** | ✅ Обратная совместимость | Клиент 3.x работает с брокером 4.x без проблем (базовый producer/consumer) |
| **Новый клиент → Старый брокер** | ⚠️ Частичная | Клиент 4.x может общаться с брокером 3.x, но новые фичи (KIP-848) недоступны |
| **Client < 2.1 → Брокер 4.0+** | ❌ Несовместимо | Протоколы до 0.10.x полностью удалены в Kafka 4.0 ([KIP-896](https://cwiki.apache.org/confluence/x/K5sODg)) |

### 3.2. Матрица совместимости Kafka Client-Broker (официальная, Kafka 4.0+)

| Версия клиента | Совместимость с Kafka 4.0 | Ключевые ограничения |
|---|---|---|
| **0.x, 1.x, 2.0** | ❌ Несовместимы | Pre-0.10.x протоколы удалены (KIP-896) |
| **2.1 – 2.8** | ⚠️ Частичная | Базовый producer/consumer — работают. Streams и Connect — ограничены |
| **3.x** | ✅ Полная | Все API работают |
| **4.x** | ✅ Полная | Включая KIP-848 (новый consumer protocol), KIP-932 (share consumer) |

### 3.3. JDK-совместимость для Java-клиентов

С выходом Kafka 4.0 **Java 8 удалена**. Минимальные требования:

| Kafka Client Version | Java 11 | Java 17 | Java 21 | Java 23 | Java 25 |
|---|---|---|---|---|---|
| 4.0.x | ✅ | ✅ | ✅ | ✅ | ❌ |
| 4.1.x | ✅ | ✅ | ✅ | ✅ | ❌ |
| 4.2.x | ✅ | ✅ | ✅ | ❌ | ✅ |

### 3.4. Версии librdkafka

Текущая стабильная версия librdkafka: **2.8.x** (февраль 2026). Совместима с Kafka от 0.8 до 4.2+.

### 3.5. Confluent-специфика

Официальные Confluent-клиенты имеют свой цикл релизов, **не совпадающий** с релизным циклом Apache Kafka:

- **Стандартная поддержка (Standard Support):** 2 года после выхода минорной версии
- **Премиум-поддержка (Platinum/Premier):** 3 года
- Патчи безопасности выпускаются в течение всего периода поддержки

---

## 4. Клиентские библиотеки по языкам

### 4.1. Java — эталонный клиент

**Библиотека:** `org.apache.kafka:kafka-clients` (Maven Central)

Java-клиент — самый полный и стабильный. Он входит в исходный код Apache Kafka и развивается синхронно с брокером. Это единственный клиент, который поддерживает **Kafka Streams** и **Kafka Connect** runtime.

#### Producer (Java)

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import java.util.Properties;

public class SimpleProducer {
    public static void main(String[] args) {
        // Конфигурация продюсера
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", StringSerializer.class.getName());
        props.put("value.serializer", StringSerializer.class.getName());
        // Гарантия доставки: ждём подтверждения от всех in-sync реплик
        props.put("acks", "all");
        // Идемпотентность: предотвращает дубликаты при retry
        props.put("enable.idempotence", "true");

        try (Producer<String, String> producer = new KafkaProducer<>(props)) {
            for (int i = 0; i < 100; i++) {
                String key = "key-" + (i % 10);  // распределение по партициям
                String value = "Сообщение номер " + i;

                ProducerRecord<String, String> record =
                    new ProducerRecord<>("my-topic", key, value);

                // Асинхронная отправка с callback
                producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        System.err.println("Ошибка отправки: " + exception.getMessage());
                    } else {
                        System.out.printf("Отправлено в partition=%d, offset=%d%n",
                            metadata.partition(), metadata.offset());
                    }
                });
            }
            // flush() гарантирует отправку всех накопленных сообщений перед закрытием
            producer.flush();
        }
    }
}
```

#### Consumer (Java)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class SimpleConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "my-consumer-group");  // группа для координации
        props.put("key.deserializer", StringDeserializer.class.getName());
        props.put("value.deserializer", StringDeserializer.class.getName());
        // Отключаем auto-commit — коммитим вручную после обработки
        props.put("enable.auto.commit", "false");
        // Если offset не найден — читаем с самого начала
        props.put("auto.offset.reset", "earliest");

        try (Consumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));  // ждём до 1 секунды

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Получено: partition=%d, offset=%d, key=%s, value=%s%n",
                        record.partition(), record.offset(), record.key(), record.value());
                }

                // Ручной commit offset-ов после успешной обработки
                try {
                    consumer.commitSync();
                } catch (CommitFailedException e) {
                    System.err.println("Commit не удался: " + e.getMessage());
                }
            }
        }
    }
}
```

#### Spring Kafka (альтернатива высокого уровня)

Для Spring-приложений рекомендуется `spring-kafka` (текущая версия 4.0.x), который предоставляет `KafkaTemplate` и аннотацию `@KafkaListener`:

```java
// Продюсер одной строкой
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;
kafkaTemplate.send("my-topic", "key", "value");

// Консьюмер — декларативно
@KafkaListener(topics = "my-topic", groupId = "my-group")
public void listen(String message) {
    System.out.println("Получено: " + message);
}
```

---

### 4.2. Python — библиотеки на выбор

В Python-экосистеме есть три основных клиента. **Приоритет выбора:**

1. **confluent-kafka-python** — для production, максимальная производительность (librdkafka)
2. **aiokafka** — для asyncio-приложений (FastAPI, aiohttp)
3. **kafka-python** — только для прототипирования (чистый Python, низкая производительность, не поддерживает новейшие фичи)

#### Producer (Python, confluent-kafka)

```python
from confluent_kafka import Producer
import json

# Конфигурация: все параметры передаются как dict
config = {
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',                          # ждём подтверждения от всех ISR
    'enable.idempotence': True,             # предотвращает дубликаты
    'compression.type': 'snappy',           # сжатие для экономии bandwidth
}

producer = Producer(config)


def delivery_callback(err, msg):
    """Callback вызывается для каждого сообщения после отправки."""
    if err:
        print(f'Ошибка доставки: {err}')
    else:
        print(f'Доставлено: topic={msg.topic()}, '
              f'partition={msg.partition()}, offset={msg.offset()}')


# Отправка сообщений
for i in range(100):
    value = json.dumps({'id': i, 'data': f'запись_{i}'})
    # produce() — неблокирующий, сообщение попадает во внутренний буфер
    producer.produce(
        topic='my-topic',
        key=str(i % 10),            # распределение по партициям
        value=value,
        callback=delivery_callback,
    )
    # poll() обрабатывает callback'и — вызывайте периодически
    producer.poll(0)

# flush() гарантирует отправку всех буферизованных сообщений
producer.flush()
```

#### Consumer (Python, confluent-kafka)

```python
from confluent_kafka import Consumer, KafkaException

config = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'my-consumer-group',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,  # ручной commit после обработки
}

consumer = Consumer(config)
consumer.subscribe(['my-topic'])

try:
    while True:
        # poll ожидает сообщения (макс. 1 секунда)
        msg = consumer.poll(1.0)

        if msg is None:
            continue  # таймаут — новых сообщений нет
        if msg.error():
            raise KafkaException(msg.error())

        # Обработка сообщения
        print(f'Получено: partition={msg.partition()}, '
              f'offset={msg.offset()}, key={msg.key()}, value={msg.value()}')

        # Ручной commit offset-а (синхронный)
        consumer.commit(msg)

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

#### aiokafka — asyncio-native клиент

Для высоконагруженных asyncio-приложений:

```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import asyncio

async def produce():
    producer = AIOKafkaProducer(bootstrap_servers='localhost:9092')
    await producer.start()
    try:
        for i in range(100):
            await producer.send('my-topic', value=f'msg_{i}'.encode())
    finally:
        await producer.stop()

async def consume():
    consumer = AIOKafkaConsumer(
        'my-topic',
        bootstrap_servers='localhost:9092',
        group_id='my-group',
    )
    await consumer.start()
    try:
        async for msg in consumer:
            print(f'Получено: {msg.value.decode()}')
    finally:
        await consumer.stop()

# asyncio.run(consume())
```

---

### 4.3. Go — две стратегии

В Go-экосистеме два доминирующих подхода:

| Характеристика | **franz-go** (twmb/franz-go) | **confluent-kafka-go** |
|---|---|---|
| Реализация | Pure Go (без CGO) | CGO-обёртка над librdkafka |
| Производительность | ★★★★☆ (очень близка к librdkafka) | ★★★★★ (максимальная) |
| Зависимости | Нет нативных зависимостей | Требует установки librdkafka в системе |
| Кросскомпиляция | Тривиальна (чистый Go) | Сложна (нужен C toolchain) |
| Поддержка фич | Полная, включая транзакции и KIP-848 | Полная |
| Звёзды GitHub | ~3K | ~4.5K |
| Рекомендация | ✅ **Primary choice 2025-2026** | Для legacy или если нужна максимальная производительность |

#### Producer (Go, franz-go)

```go
package main

import (
    "context"
    "fmt"
    "github.com/twmb/franz-go/pkg/kgo"
)

func main() {
    // Создаём клиент — один экземпляр на всё приложение
    client, err := kgo.NewClient(
        // Адреса bootstrap-серверов
        kgo.SeedBrokers("localhost:9092"),
        // Топик по умолчанию (можно переопределить в каждой отправке)
        kgo.DefaultProduceTopic("my-topic"),
        // Идемпотентный продюсер
        kgo.RequiredAcks(kgo.AllISRAcks()),
        kgo.IdempotentWrite(),
        // Сжатие
        kgo.ProducerBatchCompression(kgo.ZstdCompression()),
    )
    if err != nil {
        panic(err)
    }
    defer client.Close()

    ctx := context.Background()

    // Асинхронная отправка (fire-and-forget)
    for i := 0; i < 100; i++ {
        record := &kgo.Record{
            Key:   []byte(fmt.Sprintf("key-%d", i%10)),
            Value: []byte(fmt.Sprintf("сообщение-%d", i)),
        }
        client.Produce(ctx, record, func(r *kgo.Record, err error) {
            if err != nil {
                fmt.Printf("Ошибка: %v\n", err)
            } else {
                fmt.Printf("Отправлено: partition=%d, offset=%d\n",
                    r.Partition, r.Offset)
            }
        })
    }

    // Синхронный flush перед выходом
    if err := client.Flush(ctx); err != nil {
        fmt.Printf("Flush error: %v\n", err)
    }
}
```

#### Consumer (Go, franz-go)

```go
package main

import (
    "context"
    "fmt"
    "github.com/twmb/franz-go/pkg/kgo"
)

func main() {
    client, err := kgo.NewClient(
        kgo.SeedBrokers("localhost:9092"),
        // Группа консьюмеров
        kgo.ConsumerGroup("my-consumer-group"),
        // Топики для чтения
        kgo.ConsumeTopics("my-topic"),
        // Считывать с начала, если offset не найден
        kgo.ConsumeResetOffset(kgo.NewOffset().AtStart()),
        // Не коммитить автоматически
        kgo.DisableAutoCommit(),
    )
    if err != nil {
        panic(err)
    }
    defer client.Close()

    ctx := context.Background()

    for {
        // PollFetches — блокирующее чтение батча сообщений
        fetches := client.PollFetches(ctx)
        if fetches.IsClientClosed() {
            return
        }

        // Итерация по всем ошибкам
        fetches.EachError(func(topic string, partition int32, err error) {
            fmt.Printf("Ошибка чтения: topic=%s, partition=%d, err=%v\n",
                topic, partition, err)
        })

        // Итерация по всем записям
        fetches.EachRecord(func(record *kgo.Record) {
            fmt.Printf("Получено: partition=%d, offset=%d, key=%s, value=%s\n",
                record.Partition, record.Offset, record.Key, record.Value)
        })

        // Ручной commit offset-ов
        if err := client.CommitUncommittedOffsets(ctx); err != nil {
            fmt.Printf("Ошибка commit: %v\n", err)
        }
    }
}
```

---

### 4.4. .NET / C# — две эры

До 2023 года доминировал community-клиент `Confluent.Kafka` (v1.x). С 2024 года Confluent официально выпустила новый клиент `Confluent.Kafka` v2.x с нативной поддержкой `async/await`, современных .NET версий и улучшенным API.

#### Producer (C#, современный API)

```csharp
using Confluent.Kafka;
using System.Text.Json;

var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    // Idempotent producer — предотвращает дубликаты при retry
    EnableIdempotence = true,
    // Ждём подтверждения от всех ISR
    Acks = Acks.All,
    // Сжатие
    CompressionType = CompressionType.Snappy,
};

using var producer = new ProducerBuilder<string, string>(config).Build();

for (int i = 0; i < 100; i++)
{
    var message = new Message<string, string>
    {
        Key = $"key-{i % 10}",
        Value = JsonSerializer.Serialize(new { id = i, data = $"запись_{i}" }),
    };

    try
    {
        var result = await producer.ProduceAsync("my-topic", message);
        Console.WriteLine(
            $"Отправлено: partition={result.Partition}, offset={result.Offset}");
    }
    catch (ProduceException<string, string> ex)
    {
        Console.WriteLine($"Ошибка отправки: {ex.Error.Reason}");
    }
}

producer.Flush(TimeSpan.FromSeconds(10));
```

#### Consumer (C#, современный API)

```csharp
using Confluent.Kafka;

var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    GroupId = "my-consumer-group",
    AutoOffsetReset = AutoOffsetReset.Earliest,
    EnableAutoCommit = false,  // ручной commit
};

using var consumer = new ConsumerBuilder<string, string>(config).Build();
consumer.Subscribe("my-topic");

var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;
    cts.Cancel();
};

try
{
    while (!cts.Token.IsCancellationRequested)
    {
        try
        {
            var consumeResult = consumer.Consume(cts.Token);
            Console.WriteLine(
                $"Получено: partition={consumeResult.Partition}, " +
                $"offset={consumeResult.Offset}, " +
                $"key={consumeResult.Message.Key}, " +
                $"value={consumeResult.Message.Value}");

            // Ручной commit после успешной обработки
            consumer.Commit(consumeResult);
        }
        catch (ConsumeException ex)
        {
            Console.WriteLine($"Ошибка потребления: {ex.Error.Reason}");
        }
    }
}
catch (OperationCanceledException)
{
    // Штатное завершение
}
finally
{
    consumer.Close();
}
```

---

### 4.5. Node.js / JavaScript

Два основных варианта для Node.js:

| Характеристика | **kafka.js** (tulios/kafkajs) | **confluent-kafka-javascript** |
|---|---|---|
| Реализация | Pure JavaScript | Node.js native addon (librdkafka) |
| Производительность | ★★★☆☆ | ★★★★★ |
| Зависимости | 0 нативных | Требует компиляции |
| Транзакции | ❌ | ✅ |
| Поддержка Schema Registry | ⚠️ Ручная (REST/HTTP) | ✅ Нативная |
| KIP-848 (new consumer) | ❌ | ✅ |
| Рекомендация | Прототипирование, простые сценарии | Production, высокие нагрузки |

#### Producer (Node.js, kafka.js)

```javascript
const { Kafka, CompressionTypes } = require('kafkajs');

const kafka = new Kafka({
    clientId: 'my-app',
    brokers: ['localhost:9092'],
});

const producer = kafka.producer();

const run = async () => {
    // Подключение к кластеру
    await producer.connect();

    for (let i = 0; i < 100; i++) {
        await producer.send({
            topic: 'my-topic',
            compression: CompressionTypes.GZIP,
            messages: [
                {
                    key: `key-${i % 10}`,
                    value: JSON.stringify({ id: i, data: `запись_${i}` }),
                },
            ],
        });
        console.log(`Отправлено сообщение ${i}`);
    }

    await producer.disconnect();
};

run().catch(console.error);
```

#### Consumer (Node.js, kafka.js)

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
    clientId: 'my-app',
    brokers: ['localhost:9092'],
});

const consumer = kafka.consumer({ groupId: 'my-consumer-group' });

const run = async () => {
    await consumer.connect();
    await consumer.subscribe({ topic: 'my-topic', fromBeginning: true });

    await consumer.run({
        // autoCommit: false для ручного commit после обработки
        autoCommit: false,
        eachMessage: async ({ topic, partition, message }) => {
            console.log({
                partition,
                offset: message.offset,
                key: message.key?.toString(),
                value: message.value?.toString(),
            });

            // Ручной commit offset-а
            await consumer.commitOffsets([
                { topic, partition, offset: (Number(message.offset) + 1).toString() }
            ]);
        },
    });
};

run().catch(console.error);
```

#### Producer (Node.js, confluent-kafka-javascript)

```javascript
const { KafkaProducer } = require('@confluentinc/kafka-javascript');

async function produce() {
    const producer = new KafkaProducer({
        'bootstrap.servers': 'localhost:9092',
        'acks': 'all',
        'enable.idempotence': true,
    }, {}, {});

    await producer.connect();

    // Асинхронная отправка с промисами
    const deliveries = [];
    for (let i = 0; i < 100; i++) {
        deliveries.push(producer.send({
            topic: 'my-topic',
            key: `key-${i % 10}`,
            value: JSON.stringify({ id: i, data: `запись_${i}` }),
        }));
    }

    // Ждём доставки всех сообщений
    await Promise.all(deliveries);
    console.log('Все сообщения доставлены');

    await producer.disconnect();
}

produce().catch(console.error);
```

---

### 4.6. Rust

Rust-экосистема Kafka клиентов:

1. **rdkafka** (fede1024/rust-rdkafka) — безопасная Rust-обёртка над librdkafka. Наиболее зрелый вариант (~1.6K звёзд GitHub).
2. **franz** (twmb/franz — это Go-библиотека, не путать; для Rust аналогов pure-Rust с полной совместимостью пока нет) — в 2025-2026 появились экспериментальные проекты, но они пока не production-ready.

#### Producer (Rust, rdkafka)

```rust
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};
use std::time::Duration;

#[tokio::main]
async fn main() {
    // Создание продюсера через builder-паттерн
    let producer: FutureProducer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        .set("acks", "all")
        .set("enable.idempotence", "true")
        .set("compression.type", "snappy")
        .create()
        .expect("Не удалось создать продюсера");

    // Асинхронная отправка (futures-based)
    let mut deliveries = Vec::new();
    for i in 0..100 {
        let record = FutureRecord::to("my-topic")
            .key(&format!("key-{}", i % 10))
            .payload(&format!("сообщение-{}", i));

        deliveries.push(producer.send(record, Duration::from_secs(5)));
    }

    // Ждём результатов всех отправок
    for delivery in deliveries {
        match delivery.await {
            Ok((partition, offset)) => {
                println!("Отправлено: partition={partition}, offset={offset}");
            }
            Err((err, _)) => {
                eprintln!("Ошибка отправки: {err}");
            }
        }
    }
}
```

#### Consumer (Rust, rdkafka)

```rust
use rdkafka::config::ClientConfig;
use rdkafka::consumer::{stream_consumer::StreamConsumer, Consumer};
use rdkafka::message::BorrowedMessage;
use rdkafka::Message;
use futures::stream::StreamExt;

#[tokio::main]
async fn main() {
    // Создание stream-консьюмера
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        .set("group.id", "my-consumer-group")
        .set("auto.offset.reset", "earliest")
        .set("enable.auto.commit", "false")
        .create()
        .expect("Не удалось создать консьюмера");

    consumer
        .subscribe(&["my-topic"])
        .expect("Не удалось подписаться на топик");

    // stream() возвращает бесконечный поток сообщений
    let mut stream = consumer.stream();

    println!("Ожидание сообщений...");
    while let Some(result) = stream.next().await {
        match result {
            Ok(borrowed_msg) => {
                print_message(&borrowed_msg);
                // Ручной commit
                if let Err(e) = consumer.commit_message(&borrowed_msg,
                    rdkafka::consumer::CommitMode::Sync) {
                    eprintln!("Ошибка commit: {e}");
                }
            }
            Err(e) => eprintln!("Ошибка получения: {e}"),
        }
    }
}

fn print_message(msg: &BorrowedMessage) {
    let payload = msg.payload_view::<str>().unwrap_or(Ok(""));
    let key = msg.key_view::<str>().unwrap_or(Ok(""));
    println!(
        "Получено: partition={}, offset={}, key={}, value={}",
        msg.partition(),
        msg.offset(),
        key.unwrap_or(""),
        payload.unwrap_or("")
    );
}
```

---

### 4.7. C/C++ — librdkafka

librdkafka — это основа для всех не-JVM клиентов. Хотя писать на чистом C/C++ для Kafka редкость в прикладных приложениях, она широко используется в embedded-системах и высокопроизводительных сервисах на C++.

#### Producer (C, librdkafka)

```c
#include <stdio.h>
#include <string.h>
#include <librdkafka/rdkafka.h>

static void dr_msg_cb(rd_kafka_t *rk,
                       const rd_kafka_message_t *rkmessage,
                       void *opaque) {
    if (rkmessage->err) {
        fprintf(stderr, "Ошибка доставки: %s\n",
                rd_kafka_err2str(rkmessage->err));
    } else {
        fprintf(stdout, "Доставлено в partition %d, offset %lld\n",
                rkmessage->partition, (long long)rkmessage->offset);
    }
}

int main() {
    rd_kafka_conf_t *conf;
    rd_kafka_t *rk;
    char errstr[512];

    // Создание конфигурации
    conf = rd_kafka_conf_new();

    // Установка параметров
    if (rd_kafka_conf_set(conf, "bootstrap.servers", "localhost:9092",
                          errstr, sizeof(errstr)) != RD_KAFKA_CONF_OK) {
        fprintf(stderr, "%s\n", errstr);
        return 1;
    }

    // Callback для подтверждения доставки
    rd_kafka_conf_set_dr_msg_cb(conf, dr_msg_cb);

    // Создание продюсера
    rk = rd_kafka_new(RD_KAFKA_PRODUCER, conf, errstr, sizeof(errstr));
    if (!rk) {
        fprintf(stderr, "Не удалось создать продюсера: %s\n", errstr);
        return 1;
    }

    // Отправка сообщений
    for (int i = 0; i < 100; i++) {
        char value[128], key[32];

        snprintf(value, sizeof(value), "сообщение-%d", i);
        snprintf(key, sizeof(key), "key-%d", i % 10);

        rd_kafka_producev(
            rk,
            RD_KAFKA_V_TOPIC("my-topic"),
            RD_KAFKA_V_KEY(key, strlen(key)),
            RD_KAFKA_V_VALUE(value, strlen(value)),
            RD_KAFKA_V_END
        );

        // poll обрабатывает callback'и — нужно вызывать периодически
        rd_kafka_poll(rk, 0);
    }

    // flush — ждём доставки всех сообщений (макс. 10 секунд)
    rd_kafka_flush(rk, 10000);

    rd_kafka_destroy(rk);
    return 0;
}
```

#### Consumer (C, librdkafka)

```c
#include <stdio.h>
#include <signal.h>
#include <librdkafka/rdkafka.h>

static volatile sig_atomic_t run = 1;

static void stop(int sig) { run = 0; }

int main() {
    rd_kafka_conf_t *conf;
    rd_kafka_t *rk;
    rd_kafka_topic_partition_list_t *topics;
    char errstr[512];

    signal(SIGINT, stop);
    signal(SIGTERM, stop);

    conf = rd_kafka_conf_new();
    if (rd_kafka_conf_set(conf, "bootstrap.servers", "localhost:9092",
                          errstr, sizeof(errstr)) != RD_KAFKA_CONF_OK) {
        fprintf(stderr, "%s\n", errstr);
        return 1;
    }
    if (rd_kafka_conf_set(conf, "group.id", "my-consumer-group",
                          errstr, sizeof(errstr)) != RD_KAFKA_CONF_OK) {
        fprintf(stderr, "%s\n", errstr);
        return 1;
    }
    if (rd_kafka_conf_set(conf, "auto.offset.reset", "earliest",
                          errstr, sizeof(errstr)) != RD_KAFKA_CONF_OK) {
        fprintf(stderr, "%s\n", errstr);
        return 1;
    }

    rk = rd_kafka_new(RD_KAFKA_CONSUMER, conf, errstr, sizeof(errstr));
    if (!rk) {
        fprintf(stderr, "%s\n", errstr);
        return 1;
    }

    // Подписка на топики
    topics = rd_kafka_topic_partition_list_new(1);
    rd_kafka_topic_partition_list_add(topics, "my-topic",
                                      RD_KAFKA_PARTITION_UA);
    rd_kafka_subscribe(rk, topics);
    rd_kafka_topic_partition_list_destroy(topics);

    // Основной цикл чтения
    while (run) {
        rd_kafka_message_t *rkm = rd_kafka_consumer_poll(rk, 1000);
        if (!rkm) continue;  // таймаут

        if (rkm->err) {
            fprintf(stderr, "Ошибка получения: %s\n",
                    rd_kafka_err2str(rkm->err));
        } else {
            printf("Получено: partition=%d, offset=%lld, "
                   "key=%.*s, value=%.*s\n",
                   rkm->partition, (long long)rkm->offset,
                   (int)rkm->key_len, (char*)rkm->key,
                   (int)rkm->len, (char*)rkm->payload);

            // Ручной commit
            rd_kafka_commit_message(rk, rkm, 0);
        }
        rd_kafka_message_destroy(rkm);
    }

    rd_kafka_consumer_close(rk);
    rd_kafka_destroy(rk);
    return 0;
}
```

---

## 5. Community и неофициальные клиенты

Помимо официально поддерживаемых Confluent клиентов, существует множество community-библиотек для других языков:

| Язык | Библиотека | Статус | Функциональность |
|---|---|---|---|
| **Scala** | `kafka-clients` (Java) + функциональные обёртки (fs2-kafka, zio-kafka) | ✅ Production | Полная (через Java interop) |
| **Elixir/Erlang** | `brod` (kafka4beam), `kafka_ex` | ✅ Production | Producer, Consumer, Admin |
| **Ruby** | `rdkafka-ruby` (обёртка над librdkafka), `ruby-kafka` | ✅ Production | Producer, Consumer |
| **PHP** | `php-rdkafka` (обёртка над librdkafka) | ✅ Production | Producer, Consumer |
| **Perl** | `Net::Kafka` (pure Perl, ограниченный) | ⚠️ Limited | Базовый producer |
| **Haskell** | `hw-kafka-client` (обёртка над librdkafka) | ⚠️ Limited | Producer, Consumer |
| **Zig** | Экспериментальные проекты | 🔬 Experimental | Базовый протокол |

> **Важно:** Для Scala не существует отдельного «Scala Kafka клиента» — используется Java `kafka-clients` напрямую. Функциональные обёртки (`fs2-kafka`, `zio-kafka`) добавляют поддержку стримовых абстракций Cats Effect/ZIO, но под капотом тот же `KafkaConsumer`/`KafkaProducer`.

---

## 6. Ключевые конфигурационные параметры (единые для всех клиентов)

Большинство параметров, описанных ниже, работают одинаково во всех клиентах (через librdkafka или нативные реализации):

### 6.1. Producer-параметры первой необходимости

| Параметр | Рекомендация | Зачем |
|---|---|---|
| `acks` | `all` для критичных данных, `1` для баланса | Гарантия сохранности: `all` — ждём все ISR |
| `enable.idempotence` | `true` для production | Предотвращает дубликаты при retry |
| `compression.type` | `snappy` или `zstd` | Баланс CPU vs bandwidth: snappy — быстро, zstd — сильно |
| `linger.ms` | `5-10` для низкой латентности, `50-100` для throughput | Задержка перед отправкой батча: больше → лучше батчинг |
| `batch.size` | `16384` (по умолчанию), увеличить до `65536` для high-throughput | Размер батча в байтах |
| `max.in.flight.requests.per.connection` | `5` (по умолчанию), `1` если без idempotence | Контроль параллельных неподтверждённых запросов |
| `delivery.timeout.ms` | `120000` (2 минуты) | Верхняя граница времени доставки |

### 6.2. Consumer-параметры первой необходимости

| Параметр | Рекомендация | Зачем |
|---|---|---|
| `group.id` | Уникальное имя группы | Координация консьюмеров |
| `enable.auto.commit` | `false` для production | Ручной контроль offset-ов после обработки |
| `auto.offset.reset` | `earliest` для новых групп, `latest` для recovery | Поведение при отсутствии offset-а |
| `max.poll.records` | `500` (по умолчанию), уменьшить для низкой латентности | Сколько сообщений за раз |
| `max.poll.interval.ms` | `300000` (5 мин), увеличить для долгой обработки | Таймаут между poll → иначе rebalance |
| `session.timeout.ms` | `45000` (по умолчанию) | Обнаружение падения консьюмера |
| `heartbeat.interval.ms` | `3000` (1/3 от session.timeout) | Частота heartbeat'ов |
| `isolation.level` | `read_committed` для транзакционных продюсеров | Читать только committed или все сообщения |

---

## 7. Дерево выбора: какой клиент использовать?

```
Нужен клиент для Kafka?
│
├─ Язык: Java / JVM (Scala, Kotlin)?
│  └─ Используйте kafka-clients (org.apache.kafka)
│     ├─ Spring-приложение? → Spring Kafka (KafkaTemplate + @KafkaListener)
│     ├─ Нужна потоковая обработка? → Kafka Streams
│     └─ Нужна интеграция с внешними системами? → Kafka Connect
│
├─ Язык: Python?
│  ├─ Production / высокая производительность? → confluent-kafka-python
│  ├─ Asyncio (FastAPI, aiohttp)? → aiokafka
│  └─ Прототип / обучение? → confluent-kafka-python (всё равно)
│     ⚠️ kafka-python — не использовать для production!
│
├─ Язык: Go?
│  ├─ Чистый Go (без CGO, простая кросскомпиляция)? → franz-go
│  └─ Максимальная производительность (CGO ок)? → confluent-kafka-go
│
├─ Язык: .NET / C#?
│  └─ Используйте Confluent.Kafka (v2.x, официальный клиент)
│     ⚠️ v1.x — legacy, мигрируйте на v2.x
│
├─ Язык: Node.js / TypeScript?
│  ├─ Production / высокие нагрузки? → confluent-kafka-javascript
│  └─ Прототип / простые сценарии? → kafka.js
│
├─ Язык: Rust?
│  └─ Используйте rdkafka (rust-rdkafka)
│
├─ Язык: C/C++?
│  └─ Используйте librdkafka напрямую
│
└─ Другие языки?
   ├─ Есть binding к librdkafka? → Используйте его
   │  (Elixir/brod, Ruby/rdkafka-ruby, PHP/php-rdkafka)
   └─ Нет? → Ищите community pure-language реализацию
      ⚠️ Проверяйте зрелость и поддержку!
```

---

## 8. Антипаттерны при работе с клиентскими библиотеками

### 😱 Антипаттерн 1: Создание нового Producer/Consumer на каждый запрос

```python
# Так делать НЕЛЬЗЯ:
def send_message(msg):
    producer = Producer({'bootstrap.servers': 'localhost:9092'})  # ❌
    producer.produce('topic', msg)
    producer.flush()
    # создаётся новый producer на каждый вызов — TCP-соединения не закрываются,
    # метаданные запрашиваются заново, память утекает
```

**✅ Правильно:** Один экземпляр Producer и Consumer на всё приложение. Kafka-клиенты потоко-безопасны и спроектированы для долгоживущих экземпляров.

---

### 😱 Антипаттерн 2: Auto-commit при критичных данных

```python
# Так — рискуете потерять данные:
consumer_config = {
    'enable.auto.commit': True,   # ❌ commit может произойти до обработки!
    'auto.commit.interval.ms': 5000,
}
```

Если auto-commit сработал, а обработчик упал — offset переместился, сообщение потеряно.

**✅ Правильно:** `enable.auto.commit=False`, коммитить вручную **после** успешной обработки каждого батча.

---

### 😱 Антипаттерн 3: Игнорирование ошибок callback'а

```python
# Так — тихо теряете сообщения:
producer.produce('topic', value=msg)  # ❌ без callback
producer.poll(0)  # ❌ poll без обработки ошибок
```

**✅ Правильно:** Всегда проверяйте callback и логируйте ошибки:

```python
def callback(err, msg):
    if err:
        logger.error(f'Ошибка доставки: {err}')
        # Отправьте метрику в Prometheus / Datadog
    else:
        logger.debug(f'Доставлено: partition={msg.partition()}')
```

---

### 😱 Антипаттерн 4: Слишком долгая обработка в Consumer

```java
// Так — получите rebalance и дубликаты:
while (true) {
    var records = consumer.poll(Duration.ofMillis(1000));
    for (var record : records) {
        Thread.sleep(600_000);  // ❌ 10 минут обработки на одно сообщение
        // max.poll.interval.ms истечёт → rebalance
    }
}
```

**✅ Правильно:** Обработка не должна превышать `max.poll.interval.ms` (по умолчанию 5 минут). Для длительных операций — offload в отдельный тред/очередь.

---

### 😱 Антипаттерн 5: Неправильный `auto.offset.reset`

```java
// Новая consumer-группа с 'latest' пропустит все существующие данные:
props.put("auto.offset.reset", "latest");  // ❌ если нужна полная история
```

**✅ Правильно:** `earliest` для новых consumer-групп, которые должны обработать все существующие данные. `latest` — только когда нужны только новые сообщения.

---

## 9. Производительность клиентов: цифры для ориентира

Ниже — типичные показатели на одном продюсере/консьюмере (один топик, 6 партиций, сообщения ~1KB, localhost, современный сервер):

| Клиент | Producer throughput (msg/sec) | Consumer throughput (msg/sec) | Latency p99 |
|---|---|---|---|
| **Java** (kafka-clients) | ~800K – 1.2M | ~1M – 1.5M | 2-5ms |
| **C/C++** (librdkafka) | ~1M – 1.5M | ~1.2M – 2M | 1-3ms |
| **Rust** (rdkafka) | ~800K – 1.2M | ~1M – 1.5M | 2-5ms |
| **Go** (franz-go) | ~600K – 1M | ~800K – 1.2M | 3-7ms |
| **Go** (confluent-kafka-go) | ~800K – 1.3M | ~1M – 1.6M | 2-5ms |
| **Python** (confluent-kafka) | ~200K – 400K | ~300K – 600K | 5-15ms |
| **Node.js** (confluent-kafka) | ~300K – 500K | ~400K – 700K | 5-15ms |
| **Node.js** (kafka.js) | ~50K – 150K | ~80K – 200K | 10-50ms |
| **C#/.NET** (Confluent.Kafka v2) | ~400K – 700K | ~500K – 900K | 3-10ms |

> ⚠️ **Важно:** Это ориентировочные цифры для «идеальных условий». Реальная производительность зависит от размера сообщений, сети, конфигурации брокера, `acks`, сжатия, батчинга и ещё десятков параметров. Для конкретного сценария — всегда бенчмаркайте на своём железе.

---

## 10. Что изменилось в 2024-2026: ключевые вехи

| Дата | Событие | Влияние на клиенты |
|---|---|---|
| **Март 2025** | Kafka 4.0 (стабильный) | Удаление ZooKeeper, KIP-848 (новый consumer protocol). Java 8 больше не поддерживается. |
| **Май 2025** | Confluent.Kafka .NET v2.x стабилен | Полный async/await API, поддержка .NET 8/9 |
| **Октябрь 2025** | Kafka 4.1 | KIP-932 Share Consumer. Клиенты 2.1-2.8 — частичная совместимость |
| **Февраль 2026** | Kafka 4.2 | Дальнейшие улучшения KIP-848, JDK 25 support |
| **2025-2026** | Рост franz-go | Pure-Go клиент без CGO становится основным выбором для Go-экосистемы |
| **2025-2026** | Rust-экосистема зреет | Появляются экспериментальные pure-Rust клиенты, но rdkafka остаётся стандартом |

---

## 11. Итоговый чеклист выбора клиентской библиотеки

- [ ] **Вы используете официальный или зрелый community клиент** (а не заброшенный форк с 2 звёздами на GitHub)
- [ ] **Клиент поддерживает librdkafka**, если вы не на Java — это гарантирует производительность и совместимость
- [ ] **Версия клиента совместима с версией брокера** (см. матрицу в разделе 3)
- [ ] **Один экземпляр Producer/Consumer на приложение** — не создавайте на каждый запрос
- [ ] **`enable.auto.commit=false`** для production consumer'ов — ручной контроль offset'ов
- [ ] **`acks=all` + `enable.idempotence=true`** для критичных данных — предотвращает потерю и дубликаты
- [ ] **Настроен callback/delivery-report** — вы знаете о проблемах доставки
- [ ] **`max.poll.interval.ms` > времени обработки батча** — избегаете нежелательных rebalance'ов
- [ ] **Compression включён** (`snappy` или `zstd`) — экономия bandwidth ценой минимальных CPU затрат
- [ ] **Проведён бенчмарк** на целевом железе с реальными размерами сообщений

---

## Источники

1. [Confluent — Apache Kafka Clients Overview](https://docs.confluent.io/kafka-client/overview.html) — официальная документация Confluent (Tier 1)
2. [Apache Kafka 4.1 — Compatibility](https://kafka.apache.org/41/getting-started/compatibility/) — официальная матрица совместимости (Tier 1)
3. [Confluent — librdkafka documentation](https://docs.confluent.io/platform/current/clients/librdkafka/html/md_INTRODUCTION.html) — официальная документация librdkafka (Tier 1)
4. [GitHub: confluentinc/confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python) — официальный Python-клиент (Tier 1)
5. [GitHub: twmb/franz-go](https://github.com/twmb/franz-go) — pure-Go клиент Kafka (Tier 2)
6. [GitHub: confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go) — официальный Go-клиент (Tier 1)
7. [GitHub: confluentinc/confluent-kafka-dotnet](https://github.com/confluentinc/confluent-kafka-dotnet) — официальный .NET клиент (Tier 1)
8. [GitHub: fede1024/rust-rdkafka](https://github.com/fede1024/rust-rdkafka) — Rust-обёртка librdkafka (Tier 2)
9. [GitHub: tulios/kafkajs](https://github.com/tulios/kafkajs) — pure-JS Kafka клиент (Tier 2)
10. [KafkaJS Documentation](https://kafka.js.org/) — официальная документация kafkajs (Tier 2)
11. [AutoMQ — Apache Kafka Clients: Usage & Best Practices](https://www.automq.com/blog/kafka-clients-usage-best-practices) (Tier 3)
12. [Quix — Choosing a Python Kafka client: A comparative analysis](https://quix.io/blog/choosing-python-kafka-client-comparative-analysis) (Tier 3)
13. [Apache Kafka Wiki — Compatibility Matrix](https://cwiki.apache.org/confluence/display/KAFKA/Compatibility+Matrix) (Tier 1)
14. [Spring for Apache Kafka](https://spring.io/projects/spring-kafka) — официальный проект Spring (Tier 1)
