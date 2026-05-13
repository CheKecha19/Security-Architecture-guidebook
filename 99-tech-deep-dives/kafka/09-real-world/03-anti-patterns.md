# Анти-паттерны Apache Kafka: как не надо строить event-driven архитектуру

> **Предыдущая статья:** [Миграция на Kafka](02-migration.md) · **Следующая статья:** [Бенчмарки и сравнения](04-benchmarks-comparison.md)

**Главный вывод:** Большинство production-инцидентов с Apache Kafka вызваны не багами платформы, а повторяющимися архитектурными ошибками. Kafka — распределённый commit-log, и ровно здесь лежит корень проблем: к нему применяют паттерны, уместные для message-очередей, реляционных БД или stateless-сервисов, и получают потерю данных, неконтролируемый рост стоимости и нестабильные пайплайны. В этой статье — 12 наиболее разрушительных анти-паттернов с разбором симптомов, причин и конкретными шагами исправления.

---

## Анти-паттерн №1: Использование Kafka как постоянного хранилища (Database)

**Аналогия:** хранить семейный фотоархив в оперативной памяти телефона вместо облачного диска. Работает до первой нехватки места.

### Симптомы

- Retention выкручен в `-1` (бесконечное хранение) на всех топиках
- Отсутствует стратегия offloading-а в S3/HDFS/data lake
- Диски брокеров заполнены на 85%+, мониторинг постоянно в orange
- Запросы вида «дай мне запись по ID» реализованы через полный скан лога

### Почему это проблема

Kafka — это распределённый журнал (distributed commit log), а не база данных. У него нет индексов для точечных запросов, нет SQL-подобного языка, он не ACID-совместим. Попытка хранить в Kafka терабайты данных «навсегда» приводит к:

- **Взрывному росту стоимости хранения.** При факторе репликации 3 каждый байт хранится трижды на дорогих EBS-дисках ($0.08/GiB против $0.023/GB в S3). Кластер на 300 MB/s throughput с retention в 7 дней порождает ~228 TB реплицированного хранилища — это $36K+/месяц только на диски.
- **Деградации производительности.** Операции compaction и очистки сегментов конкурируют с клиентскими I/O.
- **Нечитаемости данных.** Извлечь «запись по customer_id = 42» из Kafka означает просканировать весь топик — операция, занимающая часы.

### Как исправить

1. **Установите разумный retention по времени:** `retention.ms=259200000` (72 часа) для операционных топиков, `retention.ms=86400000` (24 часа) для транзитных.
2. **Используйте tiered storage (KIP-405):** начиная с Kafka 3.9, remote-сегменты могут автоматически перекладываться в S3, сохраняя доступность через broker без удержания данных на EBS.
3. **Выносите холодные данные через Kafka Connect:** S3 Sink Connector, Elasticsearch Sink Connector или custom-джобы, читающие топики и пишущие в data lake.
4. **Если данные нужны в реальном времени и для истории — разделите топики:** один с коротким retention для stream processing, второй с долгим для batch-аналитики (читается Spark/Flink по расписанию).

```bash
# Аудит retention-политик всех топиков
kafka-topics.sh --bootstrap-server localhost:9092 --describe \
  | awk '{print $1, $2}' | grep -v "^$"
```

> **Источники:** [InfoWorld — Don't make Kafka your database](https://www.infoworld.com/article/2335427/dont-make-apache-kafka-your-database.html), [AutoMQ — Top 10 Kafka Mistakes](https://www.automq.com/blog/top-10-kafka-mistakes-that-cost-you-50k-per-month)

---

## Анти-паттерн №2: Слишком мало партиций — бутылочное горлышко с первого дня

**Аналогия:** построить шестиполосный автобан и оставить однополосный въезд.

### Симптомы

- Топики созданы с 1–3 партициями «на пробу», так и живут
- Consumer lag растёт, хотя сообщения маленькие, а брокеры недогружены
- Попытка добавить потребителей не даёт прироста throughput — лишние экземпляры висят idle

### Почему это проблема

Партиция — единица параллелизма в Kafka. Внутри одной consumer group партиция назначается ровно одному потребителю. Если топик из 3 партиций читается consumer group из 10 экземпляров — 7 будут простаивать. Рост трафика упирается в throughput одной партиции, а не в мощности кластера.

Кроме того, партиция определяет ceiling масштабирования: нельзя добавить больше потребителей, чем партиций.

### Как исправить

1. **Планируйте партиции по формуле:** `Partitions = max(целевой throughput / throughput одной партиции, количество потребителей × запас 2×)`.
2. **Типичный throughput одной партиции:** 10–50 MB/s write, 20–100 MB/s read (зависит от размера сообщений и конфигурации).
3. **Заложите запас на рост:** увеличить число партиций можно, но это меняет key-to-partition mapping и ломает порядок сообщений для существующих ключей. Уменьшить — практически невозможно.
4. **Для высоконагруженных топиков:** минимум 12–24 партиции, даже если сегодня потребителей — 3.

```python
# Проверка числа партиций и consumer lag (Python + kafka-python)
from kafka import KafkaAdminClient, KafkaConsumer

admin = KafkaAdminClient(bootstrap_servers='localhost:9092')
consumer = KafkaConsumer('orders', group_id='order-processor',
                         bootstrap_servers='localhost:9092')

for topic, partitions in admin.describe_topics():
    for p in partitions['partitions']:
        end_offset = consumer.end_offsets([p])[p]
        committed = consumer.committed(p)
        lag = end_offset - (committed or 0)
        if lag > 10000:
            print(f"⚠️ {topic['topic']}/{p.partition}: lag={lag}")
```

> **Источники:** [Confluent — How to choose number of topics/partitions](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster), [Conduktor — 5 Kafka Config Mistakes](https://www.conduktor.io/blog/top-5-tips-to-build-more-robust-and-performant-kafka-applications)

---

## Анти-паттерн №3: Слишком много топиков и партиций — смерть от метаданных

**Аналогия:** открыть 10 000 вкладок в браузере и удивляться, почему компьютер тормозит.

### Симптомы

- Сотни (или тысячи) топиков, каждый с десятками партиций
- Брокеры потребляют гигабайты heap только на метаданные
- Контроллер (особенно ZK-based) захлёбывается на leader election
- Операции `--describe` или перезапуск брокера занимают минуты

### Почему это проблема

Каждая партиция — это:
- Отдельный набор файлов на диске: `.log`, `.index`, `.timeindex`
- Отдельный file handle (ограничение ОС — ulimit)
- Metadata-запись в ZooKeeper или KRaft-логе
- Участник leader election при каждом broker restart

При 10 000 партиций перезапуск брокера создаёт лавину leader election — тысячи партиций одновременно переключаются, порождая шторм ISR-синхронизации.

Типичный паттерн, ведущий к этому — создание отдельного топика «на каждый микро-сервис и каждый формат сообщений»: `user.created.v1`, `user.created.v2`, `user.updated.v1`, `order.placed.internal`, `order.placed.external`...

### Как исправить

1. **Объединяйте топики по домену:** вместо `user.created`, `user.updated`, `user.deleted` — один топик `users.events` с полем `eventType` внутри сообщения.
2. **Разделяйте топики только по разным SLA:** latency, retention, ordering (одно «окно упорядоченности» = одна партиция).
3. **Используйте Schema Registry** для версионирования схем внутри топика (backward-compatible evolution), а не создавайте новый топик на каждую версию.
4. **Мониторьте общее число партиций:** бенчмарк-лимит для ZK-based кластера — ~200K партиций, для KRaft — 2M+. Но на практике лучше держать <10K на брокер.

```yaml
# Анти-паттерн: версионирование через имена топиков
topics:
  - user.created.v1
  - user.created.v2
  - user.updated.v1

# Правильно: версионирование через Schema Registry
topics:
  - users.events  # schema: UserEvent v1→v2, backward compatible
```

> **Источники:** общие практики эксплуатации Kafka; [Confluent — Partition count guide](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)

---

## Анти-паттерн №4: null-ключ и потеря порядка сообщений

**Аналогия:** раздать пассажирам одного рейса билеты в разные самолёты и надеяться, что они прилетят одновременно.

### Симптомы

- Сообщения от одного пользователя/заказа приходят в неправильном порядке
- Потребители вынуждены писать компенсирующую логику: буферизацию, сортировку по timestamp, повторные запросы в БД для восстановления порядка
- Загадочные «невозможные» переходы состояний: `SHIPPED` приходит раньше `PAID`

### Почему это проблема

Kafka гарантирует порядок сообщений **только внутри партиции**. Без ключа (или со значением `null`) продюсер распределяет сообщения round-robin/sticky — они попадают в разные партиции, где порядок не определён.

Распространённая ошибка: разработчик не задаёт `key` при `send()`, считая что «Kafka сама разберётся». Вторая ошибка: выбор неправильного ключа — например, `customer_region` вместо `order_id`, если требуется порядок операций над одним заказом.

### Как исправить

1. **Ключ сообщения = доменная единица упорядоченности.** Если нужен порядок событий в рамках заказа — ключ = `orderId`. В рамках сессии пользователя — `sessionId`.
2. **Не используйте null-ключ, если в системе есть хоть малейшая потребность в порядке.**
3. **Проверяйте ключ на стороне продюсера:**

```java
// ❌ Анти-паттерн: null-ключ — потеря гарантий порядка
producer.send(new ProducerRecord<>("orders", null, orderEvent));

// ✅ Правильно: ключ = идентификатор агрегата
producer.send(new ProducerRecord<>("orders", order.getOrderId(), orderEvent));
```

```python
# Python: проверка на null-ключ на уровне инфраструктуры
class SafeProducer:
    def send(self, topic, key, value):
        if key is None:
            raise ValueError(f"Topic {topic}: key must not be None — ordering not guaranteed")
        self.producer.send(topic, key=key, value=value)
```

> **Источники:** [DEV Community — Kafka partitioning design](https://dev.to/andrewlegacci/kafka-partitioning-the-part-that-decides-whether-your-design-works-2ba2), [Factor House — Partition key best practices](https://factorhouse.io/articles/kafka-partition-key-best-practices)

---

## Анти-паттерн №5: «Горячая» партиция (Hot Partition / Partition Skew)

**Аналогия:** один кассир в супермаркете обслуживает очередь из 100 человек, остальные 9 касс пустуют.

### Симптомы

- Одна партиция получает 70%+ трафика, остальные почти пусты
- Один брокер загружен на 90% CPU/disk, соседние — на 20%
- Consumer lag на «горячей» партиции растёт, хотя общий throughput кластера низкий
- Потребитель, которому назначена горячая партиция, падает по OOM или timeout

### Почему это проблема

Неравномерное распределение ключей: например, топик заказов, ключ = `merchant_id`, а один мерчант (Amazon) генерирует 60% всех событий. Все его сообщения попадают в одну партицию, создавая бутылочное горлышко, которое нельзя обойти добавлением потребителей (партиция назначается только одному).

### Как исправить

1. **Проанализируйте распределение ключей до выбора стратегии.** Если один ключ доминирует — текущий partition key неработоспособен.
2. **Смените ключ на более гранулярный:** `order_id` вместо `merchant_id`, если порядок нужен на уровне заказа, а не мерчанта.
3. **Шардируйте горячий ключ:** для «супер-мерчанта» используйте compound key `merchant-123#shard0`, `merchant-123#shard1`… Это распределит нагрузку, но ослабит гарантии порядка с «per-merchant» на «per-shard».
4. **Разделите топики по профилю нагрузки:** транзакционный топик (строгий порядок) vs аналитический (слабый порядок, высокая пропускная способность).

```java
// Шардирование горячего ключа через custom partitioner
public class ShardedPartitioner implements Partitioner {
    private static final Set<String> HOT_MERCHANTS = Set.of("amazon", "walmart");
    private static final int SHARDS_PER_MERCHANT = 4;

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String keyStr = (String) key;
        int numPartitions = cluster.partitionCountForTopic(topic);

        if (HOT_MERCHANTS.contains(keyStr)) {
            int shard = ThreadLocalRandom.current().nextInt(SHARDS_PER_MERCHANT);
            return Math.abs((keyStr + "#" + shard).hashCode()) % numPartitions;
        }
        return Math.abs(keyStr.hashCode()) % numPartitions;
    }
}
```

> **Источники:** [DEV Community — Kafka partitioning](https://dev.to/andrewlegacci/kafka-partitioning-the-part-that-decides-whether-your-design-works-2ba2), [Confluent — Consumer group partition strategy](https://www.confluent.io/ja-jp/blog/kafka-consumer-group-partition-strategy/)

---

## Анти-паттерн №6: acks=0 или acks=1 в production — молчаливая потеря данных

**Аналогия:** отправить важный договор курьером без уведомления о вручении и без копии.

### Симптомы

- Периодическая пропажа сообщений без ошибок на продюсере
- Пайплайны «ломаются» непредсказуемо — часть данных есть, часть нет
- После failover-а брокера часть последних сообщений потеряна безвозвратно
- CTO получает звонки в 3 часа ночи

### Почему это проблема

**acks=0:** продюсер не ждёт подтверждения вообще. Сообщение может потеряться на этапе отправки (сетевой сбой), и продюсер об этом не узнает.

**acks=1:** продюсер ждёт подтверждения только от leader-брокера. Если leader падает до того, как реплики синхронизировались — сообщение потеряно. При следующем leader election новый лидер (бывший follower) не имеет этой записи.

Только **acks=all** (или `acks=-1`) в сочетании с `min.insync.replicas >= 2` даёт гарантию: сообщение считается записанным, только когда его подтвердили все in-sync реплики.

### Как исправить

```properties
# producer.properties — production-конфигурация
acks=all                            # Ждать подтверждения от всех ISR
retries=2147483647                  # Бесконечные повторы (контролируются delivery.timeout.ms)
delivery.timeout.ms=120000          # Общий таймаут на доставку (включая повторы)
enable.idempotence=true             # Идемпотентный продюсер (KIP-98): исключает дубликаты при повторах
max.in.flight.requests.per.connection=5  # Допустимо с idempotent-продюсером
```

```properties
# server.properties — брокер должен подтверждать репликам
min.insync.replicas=2               # Минимум 2 реплики должны подтвердить запись
default.replication.factor=3        # 3 копии данных
unclean.leader.election.enable=false # Не выбирать лидера из несинхронизированных реплик
```

**Важно:** `min.insync.replicas=2` + `replication.factor=3` означает, что кластер переживает падение одного брокера без остановки записи. Это минимальный production-стандарт.

> **Источники:** [Ksolves — Top 5 Kafka Pitfalls](https://www.ksolves.com/blog/big-data/kafka-pitfalls-every-developer-should-know), [Confluent — 5 Common Pitfalls](https://www.confluent.io/blog/5-common-pitfalls-when-using-apache-kafka/)

---

## Анти-паттерн №7: enable.auto.commit=true — дубликаты и потерянные оффсеты

**Аналогия:** автомат, который каждые 5 секунд фотографирует ваш прогресс в чтении книги, независимо от того, дочитали ли вы страницу.

### Симптомы

- После перезапуска потребитель перечитывает уже обработанные сообщения (дубликаты)
- После сбоя часть сообщений пропускается (потерянные оффсеты)
- Во время ребалансировки consumer group ведёт себя «непредсказуемо»
- При высокой нагрузке consumer lag накапливается, хотя данные обрабатываются

### Почему это проблема

`enable.auto.commit=true` фиксирует оффсет по таймеру (`auto.commit.interval.ms`, default = 5000 мс), а не после успешной обработки. Если consumer получил batch из 100 сообщений, обработал 60 и упал — оффсет мог быть закоммичен для всех 100. После перезапуска 40 сообщений будут потеряны. Или наоборот: обработаны все 100, но коммит ещё не произошёл — после рестарта все 100 перечитаются заново.

### Как исправить

```properties
enable.auto.commit=false            # Ручное управление оффсетами
```

```java
// Java: коммит после успешной обработки каждого сообщения
consumer.poll(Duration.ofMillis(1000)).forEach(record -> {
    try {
        processRecord(record);
        // Коммит только после успешной обработки
        consumer.commitSync(Collections.singletonMap(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)
        ));
    } catch (Exception e) {
        // Отправляем в DLQ, НЕ коммитим оффсет
        deadLetterProducer.send(record);
    }
});
```

```python
# Python: асинхронный коммит с коллбеком для мониторинга ошибок
def commit_callback(offsets, exception):
    if exception:
        logger.error(f"Offset commit failed: {exception}")

consumer.commit_async(callback=commit_callback)
```

> **Источники:** [Ksolves — Kafka Pitfalls](https://www.ksolves.com/blog/big-data/kafka-pitfalls-every-developer-should-know), [Confluent — 7 Common Mistakes](https://www.confluent.io/resources/online-talk/common-kafka-mistakes/)

---

## Анти-паттерн №8: Request-Response через Kafka (синхронный вызов через асинхронный брокер)

**Аналогия:** использовать почтовых голубей для видеозвонка.

### Симптомы

- Сервис A отправляет запрос в Kafka и блокируется в ожидании ответа от сервиса B
- Ответ приходит через отдельный топик, продюсер держит открытым consumer для correlation ID
- При падении сервиса B — сервис A висит до timeout
- Для каждого запроса создаются временные топики (или очереди), которые никогда не удаляются
- Тысячи «мусорных» топиков, деградация производительности кластера

### Почему это проблема

Kafka спроектирована для асинхронной, событийно-ориентированной коммуникации. Паттерн request-response ломает модель:
- Блокирующий продюсер теряет все преимущества batching и асинхронной отправки
- Correlation ID и временные reply-топики — костыли, имитирующие синхронность поверх асинхронной шины
- Настоящая цена — потерянное время разработки на поддержку этой конструкции плюс деградация кластера

### Как исправить

1. **Для request-response используйте HTTP/gRPC.** Это ровно то, для чего они созданы.
2. **Замените request-response на CQRS + Event Sourcing.** Сервис A публикует команду, сервис B обрабатывает её асинхронно и публикует событие-результат. Сервис A читает результат из своей проекции (KTable/materialized view).
3. **Если request-response абсолютно необходим** — используйте механизмы вне Kafka: HTTP callback, WebSocket, Server-Sent Events для доставки ответа обратно вызывающей стороне.

```
Request-Response (НЕПРАВИЛЬНО):
  Service A ──[request]──▶ Kafka ──[request]──▶ Service B
  Service A ◀──[reply]──── Kafka ◀──[reply]──── Service B
  ❌ Блокировка, костыли, потеря throughput

CQRS + Event Sourcing (ПРАВИЛЬНО):
  Service A ──[Command]──▶ Kafka ──[Command]──▶ Service B
  Service A ◀──читает KTable── Service B ──[Event]──▶ Kafka
  ✅ Асинхронно, масштабируемо, replayable
```

> **Источники:** [Kai Waehner — Request-Response with Kafka](https://www.kai-waehner.de/blog/2022/06/03/apache-kafka-request-response-vs-cqrs-event-sourcing/), [Confluent — Dead Letter Queue alternatives in Kafka](https://www.kai-waehner.de/blog/2022/05/30/error-handling-via-dead-letter-queue-in-apache-kafka/)

---

## Анти-паттерн №9: Игнорирование cross-AZ-трафика в облаке

**Аналогия:** платить за доставку каждого письма курьером между городами, не замечая, что почтовые расходы — самая большая строка бюджета.

### Симптомы

- Счёт AWS/GCP/Azure включает огромную строку «EC2 Data Transfer», не привязанную ни к одному сервису
- Кластер Kafka на 300 MB/s генерирует $60K+/месяц только на cross-AZ-трафик — это больше, чем compute и storage вместе
- Никто не знает, почему — межзональный трафик «спрятан» внутри общей категории

### Почему это проблема

Каждое сообщение пересекает границы Availability Zone несколько раз:
1. Producer → Leader (исходящий)
2. Leader → Follower × 2 (репликация)
3. Leader → Consumer (исходящий)

При 3 AZ × replication factor 3 каждое сообщение «путешествует» минимум 4 раза через межзональный барьер (по $0.01/GB в каждом направлении).

### Как исправить

1. **Включите Follower Fetching (KIP-392):**
   ```properties
   # broker.properties
   replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
   ```
   И настройте `client.rack` на потребителях. Потребители будут читать с ближайшей реплики, а не только с лидера — снижение потребительского cross-AZ-трафика на ~30%.

2. **Размещайте продюсеров в той же AZ, что и leader-ов их топиков** (rack-aware placement).
3. **Рассмотрите stretch-кластер** (один кластер на несколько AZ) vs multi-cluster (по кластеру на AZ + MirrorMaker) — зависит от требований к latency.
4. **Мониторьте cross-AZ-трафик как отдельную метрику.**

```python
# Prometheus alert: cross-AZ traffic > бюджет
- alert: HighCrossAZTraffic
  expr: sum(rate(kafka_network_bytes_total{direction="out"}[5m])) > 5e9
  annotations:
    summary: "Cross-AZ трафик превышает 5 GB/s — проверьте follower fetching"
```

> **Источники:** [AutoMQ — Kafka Mistakes: Cross-AZ Traffic](https://www.automq.com/blog/top-10-kafka-mistakes-that-cost-you-50k-per-month)

---

## Анти-паттерн №10: Зоопарк Kafka-коннекторов без оркестрации

**Аналогия:** сотня конвейерных лент без пульта управления — каждая живёт своей жизнью.

### Симптомы

- Десятки Kafka Connect connectors запущены через `connect-standalone`, каждый — отдельный процесс
- Нет централизованного мониторинга: никто не знает, какой коннектор упал, пока потребители downstream не начинают жаловаться
- При падении коннектора никто его не рестартует (нет supervisor-а)
- Разные коннекторы конфликтуют за одни и те же consumer groups или топики оффсетов

### Почему это проблема

Connect Standalone — инструмент разработки. В production он означает ручное управление жизненным циклом, отсутствие fault tolerance и нулевую видимость состояния.

### Как исправить

1. **Всегда используйте Distributed Mode:** `connect-distributed.sh` (или Kubernetes operator — Strimzi).
2. **Разверните мониторинг:** Prometheus metrics через JMX → Grafana dashboard для Kafka Connect: task status, throughput, error rate.
3. **Настройте автоматический restart failed tasks** (через REST API или K8s operator).
4. **Разделите Connect-кластеры по зонам ответственности:** CDC-коннекторы (Debezium) отдельно от S3-sink коннекторов. Разные SLA → разные кластеры.

```bash
# Мониторинг состояния коннекторов через REST API
curl -s http://connect:8083/connectors | jq '.[]' | while read conn; do
  STATUS=$(curl -s "http://connect:8083/connectors/$conn/status" | jq -r '.connector.state')
  [ "$STATUS" != "RUNNING" ] && echo "⚠️ $conn: $STATUS"
done
```

---

## Анти-паттерн №11: Отсутствие Dead Letter Queue (DLQ) — pipeline без предохранителя

**Аналогия:** конвейер без корзины для брака — одна испорченная деталь останавливает всю линию.

### Симптомы

- Одно «плохое» сообщение (невалидный JSON, нарушение схемы, бизнес-ошибка) вызывает бесконечные retry
- Consumer падает, рестартует, снова падает на том же offset — бесконечный цикл
- Партиция «застревает»: сообщение нельзя обработать, но и пропустить нельзя — lag копится
- Единственный «выход» — ручное смещение оффсета с потерей данных

### Почему это проблема

Kafka не удаляет сообщения после ошибки. Сообщение остаётся в партиции, consumer перечитывает его снова и снова. Без механизма изоляции проблемных сообщений весь pipeline стопорится на одном «ядовитом» сообщении (poison pill).

### Как исправить

1. **Реализуйте DLQ-паттерн:** при N повторных попытках обработать сообщение — отправляем его в отдельный dead letter topic (`orders.dlq`), коммитим основной оффсет и продолжаем.
2. **DLQ должен иметь retention (месяц/неделю) и мониторинг:** если DLQ растёт — проблема системная, а не точечная.
3. **Настройте алертинг на рост DLQ**, инструмент репроцессинга (перекладывание обратно в основной топик после исправления причины).

```java
// Java: Dead Letter Queue с настраиваемым числом попыток
int maxRetries = 3;
Map<TopicPartition, AtomicInteger> retryCount = new ConcurrentHashMap<>();

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);
            consumer.commitSync();
        } catch (NonRetryableException e) {
            // Неповторяемая ошибка → сразу в DLQ
            dlqProducer.send(new ProducerRecord<>("orders.dlq", record.key(), record.value()));
            consumer.commitSync();
        } catch (RetryableException e) {
            int retries = retryCount.getOrDefault(tp, new AtomicInteger(0)).incrementAndGet();
            if (retries > maxRetries) {
                logger.error("Max retries exceeded for offset {}", record.offset());
                dlqProducer.send(new ProducerRecord<>("orders.dlq", record.key(), record.value()));
                consumer.commitSync();
                retryCount.get(tp).set(0);
            } else {
                consumer.seek(tp, record.offset()); // Перечитать
            }
        }
    }
}
```

> **Источники:** [Confluent — Dead Letter Queue Alternatives](https://www.kai-waehner.de/blog/2022/05/30/error-handling-via-dead-letter-queue-in-apache-kafka/), community best practices

---

## Анти-паттерн №12: «Set and forget» — отсутствие эксплуатационной дисциплины

**Аналогия:** купить гоночный болид, залить бензин и никогда не заглядывать под капот.

### Симптомы

- Kafka развёрнута с дефолтными настройками, никто их не трогал с первого дня
- Нет мониторинга consumer lag, under-replicated partitions, ISR churn
- Диски заполняются неожиданно, retention никто не планировал
- Брокеры периодически падают «сами по себе», никто не знает почему
- JVM heap не тюнингован, GC-паузы затягиваются на секунды

### Почему это проблема

Kafka — сложная распределённая система, а не «поставил и забыл». Её поведение определяется:
- Дисковой подсистемой (IOPS, latency, размер)
- Сетевым bandwidth (особенно inter-AZ в облаке)
- JVM-тюнингом (GC-алгоритм, heap-размер)
- Retention, compaction, segment-rotation политиками
- Балансом партиций между брокерами

Без эксплуатационного контура эти параметры деградируют постепенно и незаметно — до первого крупного инцидента.

### Как исправить

1. **Мониторьте минимум:**
   - `kafka_consumer_group_lag` (rate-based, а не абсолютное)
   - `kafka_server_UnderReplicatedPartitions` (>0 = alert)
   - `kafka_server_ActiveControllerCount` (≠1 = alert)
   - `kafka_server_BrokerTopicMetrics_MessagesInPerSec`
   - Disk usage %, network throughput, ISR shrink/expand events

2. **Автоматизируйте рутину:**
   - Cruise Control для автоматической ребалансировки партиций
   - KEDA для автоскейлинга потребителей по lag
   - Инфраструктура как код (Terraform/Ansible/Strimzi)

3. **Проводите регулярный аудит:** retention audit, partition count audit, consumer group health-check — ежемесячно.

4. **Если нет компетенций Kafka ops** — используйте managed-платформы: Confluent Cloud, Amazon MSK, Aiven for Kafka. Их цена часто ниже скрытых затрат на эксплуатацию self-managed кластера.

> **Источники:** [Ksolves — Operational Complexity](https://www.ksolves.com/blog/big-data/kafka-pitfalls-every-developer-should-know), [OpenLogic — Kafka Performance Anti-Patterns](https://www.openlogic.com/blog/kafka-performance-tuning-guide)

---

## Сводная таблица анти-паттернов: симптомы → диагноз

| Что вы видите | Возможный анти-паттерн |
|---------------|----------------------|
| Диски заполнены на 85%+ | №1 (Kafka как БД) или №12 (set-and-forget) |
| Потребители idle, lag растёт | №2 (мало партиций) или №5 (hot partition) |
| Долгий перезапуск брокеров, GC-паузы | №3 (слишком много партиций) |
| Неправильный порядок сообщений | №4 (null-ключ) |
| Один брокер перегружен | №5 (hot partition) или №9 (cross-AZ) |
| Пропадают сообщения без ошибок | №6 (acks=0/1) |
| Дубликаты после рестарта | №7 (auto-commit) |
| Сервисы виснут на ожидании ответа | №8 (request-response) |
| Огромный счёт за сеть | №9 (cross-AZ трафик) |
| Одно плохое сообщение останавливает всё | №11 (нет DLQ) |

---

## Чеклист: аудит Kafka-кластера на анти-паттерны

- [ ] Retention не `-1` без tiered storage или offloading-стратегии
- [ ] Критичные топики имеют ≥12 партиций (запас на рост)
- [ ] Общее число партиций <10K на брокер
- [ ] Все продюсеры задают `key` (не null), ключ соответствует доменной единице упорядоченности
- [ ] `acks=all`, `enable.idempotence=true` для всех production-продюсеров
- [ ] `min.insync.replicas >= 2`, `replication.factor >= 3`
- [ ] `enable.auto.commit=false` на всех потребителях
- [ ] Нет синхронного request-response через Kafka
- [ ] Настроен Follower Fetching (KIP-392) + мониторинг cross-AZ-трафика
- [ ] Kafka Connect в Distributed Mode, мониторинг task status
- [ ] DLQ для всех consumer groups, алертинг на рост DLQ
- [ ] Prometheus + Grafana: consumer lag, URP, ISR churn, disk usage
- [ ] Cruise Control / ручная ребалансировка партиций не реже раза в месяц

---

**Заключение:** Каждый из описанных анти-паттернов по отдельности не выглядит фатальным — система продолжает работать, просто «странно». Но в совокупности они создают хрупкую архитектуру, которая ломается в самый неподходящий момент. Главное правило: проектируйте Kafka-топологию вокруг доменных границ (ordering boundaries), а не вокруг технических удобств (версии, команды, микросервисы). Второе правило: durability и observability не опциональны — они вшиваются в конфигурацию с первого дня.

---

> **Источники (12):**
> 1. [InfoWorld — Don't make Kafka your database](https://www.infoworld.com/article/2335427/dont-make-apache-kafka-your-database.html)
> 2. [AutoMQ — Top 10 Kafka Mistakes](https://www.automq.com/blog/top-10-kafka-mistakes-that-cost-you-50k-per-month)
> 3. [Confluent — 5 Common Pitfalls](https://www.confluent.io/blog/5-common-pitfalls-when-using-apache-kafka/)
> 4. [Ksolves — Top 5 Kafka Pitfalls (2026)](https://www.ksolves.com/blog/big-data/kafka-pitfalls-every-developer-should-know)
> 5. [Conduktor — 5 Kafka Config Mistakes](https://www.conduktor.io/blog/top-5-tips-to-build-more-robust-and-performant-kafka-applications)
> 6. [DEV Community — Kafka partitioning design](https://dev.to/andrewlegacci/kafka-partitioning-the-part-that-decides-whether-your-design-works-2ba2)
> 7. [Factor House — Partition key best practices](https://factorhouse.io/articles/kafka-partition-key-best-practices)
> 8. [Kai Waehner — Request-Response with Kafka](https://www.kai-waehner.de/blog/2022/06/03/apache-kafka-request-response-vs-cqrs-event-sourcing/)
> 9. [Kai Waehner — Dead Letter Queue Alternatives](https://www.kai-waehner.de/blog/2022/05/30/error-handling-via-dead-letter-queue-in-apache-kafka/)
> 10. [Confluent — Choosing number of topics/partitions](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)
> 11. [OpenLogic — Kafka Performance Anti-Patterns](https://www.openlogic.com/blog/kafka-performance-tuning-guide)
> 12. [Confluent — Consumer group partition strategy](https://www.confluent.io/ja-jp/blog/kafka-consumer-group-partition-strategy/)
