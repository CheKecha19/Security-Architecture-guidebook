# Узкие места и профилирование Apache Kafka

> **Коротко:** Узкие места Kafka — это горячие партиции, медленные потребители, узкое дисковое I/O, давление на page cache, длинные паузы GC и неоптимальная конфигурация сети. В этой статье — системный подход к диагностике, инструментарий профилировщика (JFR, async-profiler, JMX, vmtouch) и разбор четырёх реальных production-инцидентов.

---

## 1. Карта узких мест: где ломается производительность

Kafka спроектирована для последовательного I/O, zero-copy передачи и горизонтального масштабирования. Но на практике каждая из этих особенностей может стать точкой отказа. Узкие места распределяются по пяти слоям:

```
┌─────────────────────────────────────────────────────┐
│  СЛОЙ 1: Продюсеры (batch.size, linger.ms, acks)    │
├─────────────────────────────────────────────────────┤
│  СЛОЙ 2: Сеть (пропускная способность, сокет-буфер) │
├─────────────────────────────────────────────────────┤
│  СЛОЙ 3: Брокер (CPU, JVM/GC, request queue)        │
├─────────────────────────────────────────────────────┤
│  СЛОЙ 4: Диск (page cache, IOPS, fsync)             │
├─────────────────────────────────────────────────────┤
│  СЛОЙ 5: Потребители (lag, rebalance, fetch)        │
└─────────────────────────────────────────────────────┘
```

**Ключевая мысль:** в отличие от баз данных, где узкое место почти всегда диск, в Kafka узкое место может мигрировать между слоями в зависимости от рабочей нагрузки. Потоковый процессинг с малым retention (часы) давит на page cache; batch-аналитика с большим retention (недели) — на диск; высокочастотный трейдинг — на JVM-паузы GC.

---

## 2. Семь классических проблем производительности

### 2.1 Горячие партиции (Hot Partitions)

**Симптомы:**
- Один брокер на 99% CPU, остальные на 15–20%
- Consumer lag растёт только для части партиций
- Таймауты rebalance в consumer group

**Корневая причина:** неравномерное распределение ключей. Хэш-функция `murmur2(key) % partition_count` даёт равномерное распределение только при высокой кардинальности ключа.

**Пример из реальной жизни:** команда изменила ключ партиционирования с `userId` (миллионы уникальных значений) на `countryCode` (12 значений, причём 80% трафика из US). Партиция, на которую упал `hash("US") % 48`, получала 80% всего потока событий. Тесты проходили идеально, потому что в нагрузочном тестировании использовались случайные UUID, а не реальное распределение стран.

```java
// ПЛОХО: низкая кардинальность ключа
record.setKey(countryCode);  // 12 уникальных значений, сильный перекос

// ХОРОШО: высокая кардинальность
record.setKey(userId);       // миллионы уникальных значений

// ПЛОХО: композитный ключ с доминирующим префиксом
String key = tenantId + ":" + userId;  // tenantId доминирует в хэше

// ХОРОШО: случайный суффикс для разрушения перекоса
String key = userId + ":" + tenantId;  // userId даёт равномерность
```

**Диагностика:**
```bash
# Просмотр распределения лидеров партиций по брокерам
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic user-events

# JMX-метрики байтового входа по партициям
# kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=user-events
```

**Контракт перекоса (Skew Contract):** продвинутые команды внедряют формальный контракт партиционирования, который проверяется в CI:

```yaml
# skew_contract.yml
topics:
  user-events:
    partitions: 48
    max_partition_share: 0.08   # максимум 8% трафика на партицию
    max_gini: 0.15              # коэффициент неравенства Джини
    min_unique_keys: 20000      # минимальная кардинальность
```

### 2.2 Медленные потребители и Consumer Lag

**Симптомы:**
- `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*,name=records-lag-max` неуклонно растёт
- Потребители не успевают обрабатывать входящий поток

**Первопричины — три класса проблем:**

| Класс | Причина | Метрика для проверки |
|-------|---------|---------------------|
| Продюсер опережает | throughput продюсера > throughput потребителя | `BytesInPerSec` vs `BytesConsumedPerSec` |
| Медленная бизнес-логика | Обработка одного сообщения > 100ms | Время выполнения в коде потребителя |
| Дисбаланс партиций | Часть consumer-ов простаивает | `records-lag` по партициям |

**Системный подход к диагностике:**

```bash
# Шаг 1: Оцениваем масштаб отставания
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group fraud-detection --describe

# Вывод покажет:
# TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders   0          1500000         1520000          20000
# orders   1          800000          1500000          700000  ← проблема!

# Шаг 2: Проверяем throughput продюсера
# JMX: kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec

# Шаг 3: Если источник — медленная обработка, включаем выборку
# в коде потребителя
```

```java
// Профилирование времени обработки в потребителе
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, Order> record) {
    long start = System.nanoTime();
    processOrder(record.value());
    long elapsed = System.nanoTime() - start;
    
    // Логируем, если обработка дольше порога
    if (elapsed > TimeUnit.MILLISECONDS.toNanos(100)) {
        log.warn("Slow processing: partition={}, offset={}, time={}ms",
            record.partition(), record.offset(), 
            TimeUnit.NANOSECONDS.toMillis(elapsed));
    }
}
```

**Стратегии устранения:**
- **Горизонтальное масштабирование:** добавить consumer-инстансы (до числа партиций)
- **Увеличить число партиций:** необратимая операция, требует оценки побочных эффектов (metadata load, rebalance time)
- **Оптимизация fetch-параметров:** `fetch.min.bytes`, `fetch.max.wait.ms`, увеличение `max.poll.records`
- **Асинхронная обработка:** вынос тяжёлой логики в отдельный thread pool (с осторожностью — нарушает порядок в пределах партиции)

### 2.3 Дисковое I/O и Page Cache

**Ключевое архитектурное понимание:** Kafka спроектирована так, чтобы НЕ читать с диска при нормальной работе — все чтения обслуживаются из page cache операционной системы.

**Схема потоков данных:**

```
   Продюсер ──запись──▶ [Page Cache] ──fsync (отложенный)──▶ Диск
                              │
   Потребитель ◀──sendfile()─┘  (zero-copy чтение)
```

**Когда page cache перестаёт справляться:**

1. **Consumer lag превышает размер page cache** — потребитель читает данные старше, чем помещается в память
2. **Много «холодных» топиков** — конкуренция за page cache между десятками/сотнями топиков
3. **Рестарт потребителя** — при перезапуске consumer вынужден читать с диска весь накопленный объём

**Симптомы:**
- Рост `disk read utilization` на брокерах (в идеале — около нуля)
- Рост fetch latency (`kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer`)
- Корреляция read I/O с consumer lag'ом

**Диагностический инструментарий:**

```bash
# 1. Оценка использования page cache для партиций Kafka
vmtouch /var/lib/kafka/data/user-events-0/
# Вывод: Resident Pages: 2345/2400  97% — хорошо
#        Resident Pages: 12/2400    0.5% — всё ушло на диск

# 2. Мониторинг дискового I/O на уровне ОС
iostat -x 1
# Смотрим %util и await для дисков Kafka

# 3. Состояние page cache системы в целом
free -m
#             total   used   free   shared   buff/cache   available
# Mem:        32148   8234   1245   0        22669         23514
#                                                    ↑
#                                          page cache (~22 GB)

# 4. Настройка swappiness (Kafka должна избегать swap)
sysctl vm.swappiness=1
```

**Глубокое погружение: случай из Instana.** При нагрузке >12 GB/s ingress на 5 production-регионах AWS/GCP команда обнаружила корреляцию между read utilization дисков и fetch-latency потребителей. Причина: рост consumer lag в пиковые часы выталкивал «горячие» данные из page cache, заставляя брокер читать с диска. Решение: автоскейлинг потребителей через KEDA с метрикой по consumer lag.

**Рекомендации по дисковому I/O:**
- **NVMe SSD** — минимум для production; никаких HDD для Kafka-логов
- **Отдельные диски для логов** — распределение партиций по нескольким физическим устройствам
- **Файловая система XFS** — лучшая производительность для последовательных операций
- **RAID 0/10** — приоритет скорости записи над избыточностью (данные уже реплицированы на уровне Kafka)
- **НЕ отключать fsync полностью** — можно потерять данные при крахе ядра ОС

### 2.4 JVM: Паузы Garbage Collection

Kafka работает на JVM, а значит, подвержена паузам сборщика мусора. Даже 100-миллисекундная пауза GC на брокере означает, что все продюсеры и потребители, подключённые к этому брокеру, «замирают» на это время.

**Симптомы:**
- Периодические всплески latency с интервалом, соответствующим циклу GC
- `RequestQueueTimeMs` резко растёт во время пауз
- В логах GC: `[gc] GC (Allocation Failure) ... 150ms`

**Рекомендуемые настройки JVM для Kafka-брокера:**

```bash
# KAFKA_JVM_PERFORMANCE_OPTS: базовые настройки GC
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"  # heap фиксирован, чтобы избежать ресайза
export KAFKA_JVM_PERFORMANCE_OPTS="
  -server
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:G1HeapRegionSize=16m
  -XX:G1NewSizePercent=30
  -XX:+AlwaysPreTouch
  -XX:+DisableExplicitGC
  -XX:+PrintGCDateStamps
  -XX:+PrintGCDetails
  -Xlog:gc*:file=/var/log/kafka/gc.log:time,uptime:filecount=10,filesize=100M
"
```

**Почему НЕ больше 8 GB heap?** В отличие от типичных Java-приложений, Kafka полагается на page cache ОС для хранения «горячих» данных. Выделение >8 GB под heap крадёт память у page cache, что парадоксально СНИЖАЕТ производительность брокера.

**G1GC vs ZGC: когда переходить?**

| Параметр | G1GC (Java 11+) | ZGC (Java 17+) |
|----------|-----------------|-----------------|
| Целевая пауза | ~20 ms | <1 ms |
| Потери throughput | ~5% | ~10–15% |
| Накладные на память | Низкие | +25% native memory |
| Стабильность | Проверена годами | Быстро зреет |
| Когда применять | Большинство случаев | Финансы, HFT, жёсткие SLA по latency |

**Реальный кейс (LinkedIn, изначальные разработчики Kafka):** на заре проекта брокеры страдали от длинных пауз GC из-за mmap-объектов `OffsetIndex` (KAFKA-4614). При GC собирались десятки тысяч mmap-объектов, что приводило к паузам в сотни миллисекунд. Решение: переработка индексных структур для минимизации числа mmap-объектов.

**ZGC в финансах:** как отмечается в отраслевых отчётах, финансовые организации, использующие Kafka для high-frequency trading, переходят на ZGC. Причина: даже пауза GC в 20ms (лучший случай G1GC) неприемлема, когда latency budget всего 50ms на всю цепочку. ZGC даёт сублинейные паузы (<1ms), но ценой 10–15% потери общего throughput.

### 2.5 Сетевые узкие места

**Симптомы:**
- `NetworkProcessorAvgIdlePercent` → 0 (сетевые потоки перегружены)
- `BytesInPerSec` или `BytesOutPerSec` упираются в предел сетевого интерфейса
- `RequestQueueTimeMs` растёт

**Ключевые параметры:**

```properties
# server.properties — сетевые настройки брокера

# Количество сетевых потоков (обычно = число ядер CPU)
num.network.threads=8

# Размер очереди запросов
queued.max.requests=1000

# Сокетные буферы
socket.send.buffer.bytes=1048576    # 1 MB
socket.receive.buffer.bytes=1048576  # 1 MB

# Максимальный размер запроса
socket.request.max.bytes=104857600  # 100 MB
```

**Диагностика на уровне ОС:**
```bash
# Проверка использования сетевого интерфейса
sar -n DEV 1
# или
nload eth0

# Проверка количества открытых сокетов
ss -s

# Лимит файловых дескрипторов (каждый сокет = fd)
ulimit -n   # должно быть >= 65536 для production
```

### 2.6 Проблемы конфигурации продюсера

**Классические антипаттерны:**

1. **Слишком маленький `batch.size` (по умолчанию 16KB)** — каждый вызов `send()` превращается в отдельный сетевой запрос. При высоком throughput это создаёт огромный overhead.

2. **`acks=all` без `min.insync.replicas`** — запись ждёт подтверждения от ВСЕХ реплик, включая "мёртвые". При потере брокера продюсер «зависает» на `request.timeout.ms`.

3. **Отсутствие сжатия при большом payload** — JSON без сжатия занимает в 5–10 раз больше, чем Avro + Snappy/Zstd.

```java
// Оптимальная конфигурация для высокого throughput
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// Пакетная отправка: накапливаем до 64KB или 10ms
props.put("batch.size", 65536);        // 64 KB
props.put("linger.ms", 10);            // ждём до 10ms

// Компрессия
props.put("compression.type", "zstd");  // лучший баланс скорость/сжатие

// Durability: ждём кворум, но не все реплики
props.put("acks", "all");
// Настройка на уровне топика: min.insync.replicas=2

// Буферизация на стороне продюсера
props.put("buffer.memory", 33554432);  // 32 MB
props.put("max.in.flight.requests.per.connection", 5);
```

### 2.7 Rebalance Storms (Шторм перебалансировки)

**Симптомы:**
- Потребители циклически теряют и восстанавливают партиции
- CPU потребителей уходит в rebalance вместо обработки
- Consumer lag растёт экспоненциально во время rebalance

**Корневые причины:**
- `max.poll.interval.ms` слишком мал — потребитель не успевает обработать батч и вылетает из группы
- Частые деплои consumer-ов (каждый рестарт = rebalance)
- Слишком много партиций → rebalance занимает секунды/минуты

**Решения:**
```java
// Настройки потребителя против rebalance storms
props.put("max.poll.interval.ms", 600000);   // 10 минут на обработку батча
props.put("max.poll.records", 500);          // ограничиваем размер батча
props.put("session.timeout.ms", 30000);      // таймаут heartbeat
props.put("heartbeat.interval.ms", 3000);    // частота heartbeat
props.put("group.instance.id", "consumer-1"); // статическое членство в группе
```

**Статическое членство в группе (Static Group Membership)** — ключевое улучшение Kafka 2.3+. Потребитель с `group.instance.id` при временном отключении (до `session.timeout.ms`) не вызывает rebalance, а просто переподключается к своим партициям.

---

## 3. Инструментарий профилирования

### 3.1 JMX-метрики — первый рубеж диагностики

Kafka предоставляет сотни JMX-метрик через MBeans. Вот критический минимум для поиска узких мест:

**Метрики брокера:**

| MBean/Метрика | Что означает | Норма | Сигнал тревоги |
|---------------|-------------|-------|----------------|
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Входящий байтовый трафик | — | Приближается к лимиту NIC |
| `kafka.network:type=RequestChannel,name=RequestQueueSize` | Очередь запросов | ~0 | >100 постоянно |
| `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce` | Latency produce-запроса | <10ms | >50ms (p99) |
| `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent` | Idle сетевых потоков | >0.3 | <0.1 постоянно |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Недореплицированные партиции | 0 | >0 |
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | Активный контроллер | 1 | 0 или >1 |

**Метрики потребителя:**

| MBean/Метрика | Что означает |
|---------------|-------------|
| `kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max` | Максимальный lag по партициям |
| `kafka.consumer:type=consumer-fetch-manager-metrics,name=fetch-rate` | Частота fetch-запросов |
| `kafka.consumer:type=consumer-coordinator-metrics,name=join-rate` | Частота переподключений к группе |

**JMX в production — security:**
```bash
# Безопасное включение JMX (только localhost)
export KAFKA_JMX_OPTS="
  -Dcom.sun.management.jmxremote
  -Dcom.sun.management.jmxremote.port=9999
  -Dcom.sun.management.jmxremote.local.only=true
  -Dcom.sun.management.jmxremote.authenticate=true
  -Dcom.sun.management.jmxremote.ssl=true
  -Dcom.sun.management.jmxremote.password.file=/etc/kafka/jmx.password
  -Dcom.sun.management.jmxremote.access.file=/etc/kafka/jmx.access
"
```

### 3.2 Java Flight Recorder (JFR) — глубокий анализ JVM

JFR — встроенный в HotSpot JVM фреймворк для сбора событий с практически нулевым overhead (<1%).

**Запуск JFR для Kafka-брокера:**

```bash
# При старте брокера
export KAFKA_JVM_PERFORMANCE_OPTS="
  ... 
  -XX:StartFlightRecording=disk=true,dumponexit=true,
       filename=/var/log/kafka/broker.jfr,
       maxsize=500M,
       settings=profile
"
```

**Анализ записи JFR:**

```bash
# Конвертация JFR → JSON для анализа
java -cp jfr-parser.jar JfrParser /var/log/kafka/broker.jfr > events.json

# Просмотр в Java Mission Control (jmc)
jmc -open /var/log/kafka/broker.jfr

# Или конвертация во flame graph через async-profiler
java -jar converter.jar jfr2flame broker.jfr flame.html
```

**Что искать в JFR-записи Kafka-брокера:**
- **GC-паузы:** Events → Garbage Collection → GC Pause. Любая пауза >50ms — повод для тюнинга.
- **Thread Dump:** при каких стеках брокер проводит больше всего CPU-времени.
- **Allocation:** какие объекты создаются чаще всего (потенциальный мусор).
- **Socket I/O:** корреляция сетевых событий с паузами GC.

### 3.3 Async Profiler — CPU и Allocation Profiling без safepoint bias

Async-profiler (автор — Андрей Пангин) стал индустриальным стандартом для профилирования Java-приложений. В отличие от традиционных профайлеров, он:

- **Не зависит от JVMTI** — использует `AsyncGetCallTrace` API HotSpot
- **Избегает safepoint bias** — корректно профилирует код между safepoint'ами
- **Показывает native-фреймы** — стек вызовов ядра Linux через perf_events
- **Работает с non-Java потоками** — GC, JIT-компилятор, сетевые потоки

**Базовое использование с Kafka-брокером:**

```bash
# 1. Находим PID брокера
jps -l | grep Kafka

# 2. CPU-профилирование, 30 секунд, вывод во flame graph
./profiler.sh -d 30 -f /tmp/kafka-cpu.html <PID>

# 3. Allocation-профилирование: что аллоцирует память
./profiler.sh -d 30 -e alloc -f /tmp/kafka-alloc.html <PID>

# 4. Lock-профилирование: где блокировки
./profiler.sh -d 30 -e lock -f /tmp/kafka-lock.html <PID>

# 5. Wall-clock профилирование: где реально тратится время
./profiler.sh -d 30 -e wall -f /tmp/kafka-wall.html <PID>

# 6. Экспорт в JFR для анализа в Java Mission Control
./profiler.sh -d 30 -o jfr -f /tmp/kafka.jfr <PID>
```

**Интерпретация flame graph Kafka-брокера:**

```
Ширина полосы = доля CPU-времени

Typical broker flame graph (здоровый):
┌─────────────────────────────────────────────────────┐
│                 kafka-server-start                   │
├──────────────────────┬──────────────────────────────┤
│    SocketServer      │       KafkaRequestHandler     │
│    (network I/O)     │       (process requests)      │
├───────────┬──────────┼──────────────┬───────────────┤
│  selector │Processor │  produce API │  fetch API     │
│  .select  │.read()   │  .append()   │  .read()       │
└───────────┴──────────┴──────────────┴───────────────┘

Красные флаги во flame graph:
- Широкие полосы "G1ParScanThreadState::copy_to_survivor_space" 
  → активный молодой GC, много аллокаций
- Полосы "[kernel]" (системные вызовы) шире 10% 
  → возможно, page fault'ы или давление на диск
- Полосы "sun.nio.ch" > 50% ширины 
  → сеть — доминирующее узкое место
```

**Практический сценарий:** брокер Kafka показывал 99p latency 200ms при p50 в 5ms. Async-profiler показал, что 40% CPU уходит в `G1ParScanThreadState` — сборку мусора в young generation. Причина: продюсеры слали сообщения без компрессии, создавая миллионы мелких объектов. Решение: включение `compression.type=zstd` сократило число аллокаций на 60%, p99 latency упала до 15ms.

### 3.4 OS-level инструменты

**vmtouch — инспекция page cache для Kafka-партиций:**

```bash
# Установка
git clone https://github.com/hoytech/vmtouch.git && cd vmtouch && make

# Просмотр, сколько данных партиции в page cache
vmtouch -v /var/lib/kafka/data/user-events-0/

# Пример вывода:
# Files: 3
# Directories: 0
# Resident Pages: 2345/2400  3M/3M  1G/1G  97.7%
# Elapsed: 0.00123 seconds
# 
# 97.7% в page cache — отлично, всё обслуживается из памяти

# Принудительная загрузка в page cache (на тёплом рестарте)
vmtouch -t /var/lib/kafka/data/
```

**pidstat — профилирование на уровне потоков:**
```bash
# CPU по потокам Kafka-брокера
pidstat -t -p $(pgrep -f Kafka) 1

# I/O по потокам
pidstat -d -p $(pgrep -f Kafka) 1
```

### 3.5 Kafka-специфичные CLI-инструменты

```bash
# Статистика consumer group (lag, партиции, участники)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe --members --verbose

# Нагрузка на брокер по топикам
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Дамп лога партиции (сырые сообщения)
kafka-dump-log.sh --files /var/lib/kafka/data/user-events-0/00000000000000000000.log \
  --print-data-log

# Проверка недореплицированных партиций
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --under-replicated-partitions
```

---

## 4. Case Studies: Реальные инциденты производительности

### 4.1 Инцидент №1: Скрытый перекос партиций (E-Commerce, 3 дня деградации)

**Контекст:** платформа электронной коммерции, 48 партиций для топика `user-events`, Kafka 3.6, throughput ~50K msg/s.

**Хронология:**
- **День 1:** лёгкий рост consumer lag (до 10K), команда списала на «всплеск трафика»
- **День 2:** lag достиг 500K, сработал alert. Попытка ручного rebalance — без эффекта
- **День 3:** осознание — партиция 23 получает 60% всего трафика, брокер, её обслуживающий, на 99% CPU

**Root Cause:** три недели назад в shared-библиотеке изменили генерацию ключа: `userId` (миллионы значений) → `countryCode` (12 значений). Код-ревью не выявил проблему, потому что:
1. Изменение было в утилитарном методе, не бросающемся в глаза
2. Нагрузочные тесты использовали случайные UUID вместо реального распределения (80% US)
3. Отсутствовала метрика распределения по партициям

**Что сломала «мелочь»:**
```
ДО:  partition_id = hash("user_123456") % 48 → равномерно
ПОСЛЕ: partition_id = hash("US") % 48 → 80% трафика на одну партицию
```

**Уроки:**
1. Ключ партиционирования — **контракт**, а не деталь имплементации
2. Нагрузочное тестирование должно использовать **реальные распределения ключей**, а не случайные
3. Нужны **проактивные метрики** распределения по партициям, а не только aggregate

**Внедрённые меры:**
- Skew Contract с проверкой в CI (max_partition_share ≤ 0.08)
- JMX-алерт на дисбаланс: `max(partition_bytes_in) / avg(partition_bytes_in) > 3`
- Обязательный код-ревью для любого изменения в методах генерации ключей

### 4.2 Инцидент №2: Page Cache — невидимый убийца производительности (Instana, 12 GB/s)

**Контекст:** SaaS-платформа мониторинга (Instana), 5 production-регионов AWS/GCP, брокеры Kafka обрабатывают >12 GB/s ingress.

**Симптомы:** в пиковые часы — рост fetch-latency потребителей с 5ms до 200ms, рост consumer lag, алерты на пропускную способность.

**Диагностический путь:**
1. JMX показал рост `TotalTimeMs` для fetch-запросов — но CPU брокеров в норме
2. Корреляция fetch-latency с **read utilization дисков** — ключевая находка
3. `vmtouch` показал: в пиковые часы resident pages для партиций падают с 95% до 15%
4. Причина: рост consumer lag в часы пик → потребители запрашивают данные, которые уже вытеснены из page cache → каждый fetch идёт на диск

**Цепочка отказа:**
```
Рост трафика → Consumer не справляется → Lag растёт → 
«Горячие» данные вытесняются из page cache → 
Чтение с диска → Рост fetch-latency → Lag растёт ещё быстрее → ...
```

**Что выяснилось:** один доминирующий топик с неравномерным распределением партиций создавал hotspots на конкретных брокерах, усугубляя эффект.

**Решение:**
- Ребалансировка лидеров партиций доминирующего топика
- Внедрение KEDA для автоскейлинга потребителей по метрике consumer lag
- Настройка алертов на `disk read utilization > 30%`

**Ключевой инсайт:** стандартный мониторинг (CPU, memory) НЕ показывал проблему. Свободная память была (50%), но это была память, занятая page cache. Нужны метрики на уровне дисковой подсистемы.

### 4.3 Инцидент №3: GC-паузы в HFT-системе (Финансовый сектор)

**Контекст:** high-frequency trading платформа, Kafka используется как шина для рыночных данных. Бюджет latency на всю цепочку: <50ms.

**Симптомы:** периодические (каждые ~30 секунд) выбросы p99 latency до 200ms при стабильном p50 в 3ms.

**Диагностика:** Java Flight Recorder показал 150ms GC-паузы G1GC (mixed collection). Allocation-профилирование через async-profiler выявило источник: десериализация JSON-сообщений создавала десятки тысяч временных объектов на каждый fetch-батч.

**Цепочка:**
```
JSON payload (несжатый, 10KB/сообщение) → 
Десериализация создаёт 100+ объектов на сообщение → 
10K сообщений в батче = 1M+ temporary объектов → 
G1GC mixed collection 150ms каждые 30 секунд
```

**Решение:**
1. Переход с JSON на Avro binary + Schemas Registry (размер сообщения ↓ 80%)
2. Включение `compression.type=zstd` на продюсере
3. Переход с G1GC на ZGC (Java 17): паузы GC <1ms
4. Итог: p99 latency с 200ms до 12ms

**Урок:** для latency-чувствительных систем выбор формата сериализации и GC collector'а — не микрооптимизация, а архитектурное решение.

### 4.4 Инцидент №4: Rebalance Storm и Too Many Partitions (Платформа логов)

**Контекст:** централизованная платформа сбора логов, 2000+ топиков, каждый с десятками партиций, consumer group для агрегации логов.

**Симптомы:** после каждого деплоя consumer-ов — 7–10 минут полной остановки обработки, после чего consumer lag составлял миллионы сообщений.

**Диагностика:**
```bash
# join-rate был постоянно высоким
kafka.consumer:type=consumer-coordinator-metrics,name=join-rate → 
  0.05 Hz (каждые 20 секунд новый join)

# Время rebalance для группы с 200 потребителями
# на 2000+ партиций — до 7 минут
```

**Root Cause:** слишком много партиций. Каждый rebalance требует пересогласования ВСЕХ назначений всех партиций между всеми потребителями. С 2000+ партиций это O(n²).

**Решение:**
1. Консолидация топиков (2000 → 200, с роутингом по ключу)
2. Статическое членство в группах (`group.instance.id`)
3. Incremental Cooperative Rebalancing (Kafka 2.4+): вместо stop-the-world — поэтапная передача партиций

```java
// Включение Cooperative Rebalancing
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

**Результат:** время rebalance сократилось с 7 минут до 30 секунд, consumer lag после деплоя — не более 10K сообщений.

**Урок:** партиции — не бесплатный ресурс. Каждая партиция добавляет metadata overhead, увеличивает время rebalance и recovery. Оптимальное число — не «как можно больше», а «ровно столько, сколько нужно для параллелизма».

---

## 5. Системный подход: диагностическое дерево решений

```
Проблема: рост latency / снижение throughput
│
├── Где именно проблема?
│   ├── Producer → Раздел 2.6 + JMX: record-send-rate
│   ├── Broker → Шаг ниже
│   └── Consumer → Раздел 2.2 + kafka-consumer-groups.sh
│
├── Broker: какой ресурс перегружен?
│   ├── CPU 100% на одном брокере → Раздел 2.1 (hot partitions)
│   ├── CPU 100% на всех брокерах → число сетевых потоков / handler threads
│   ├── Диск 100% util → Раздел 2.3 (page cache / disk I/O)
│   ├── RequestQueueSize > 100 → перегружен брокер, масштабировать
│   ├── GC-паузы → Раздел 2.4 (JVM tuning)
│   └── Сеть → Раздел 2.5 (NIC saturation)
│
└── Требуется профилирование?
    ├── JMX: быстрый взгляд ← Начать ЗДЕСЬ
    ├── JFR: глубокий анализ событий JVM
    ├── async-profiler: CPU/Allocation flame graphs
    └── vmtouch/iostat: OS-level инспекция
```

---

## 6. Метрики-индикаторы: что алертить

**Уровень WARNING (реагировать в рабочее время):**

| Метрика | Порог | Логика |
|---------|-------|--------|
| Consumer lag | >10K сообщений | Может накапливаться |
| Under-replicated partitions | >0 | Риск потери данных |
| NetworkProcessorAvgIdlePercent | <0.2 | Близко к saturation |
| Disk read util (пиковая) | >30% | Уходит из page cache |

**Уровень CRITICAL (немедленная реакция):**

| Метрика | Порог | Логика |
|---------|-------|--------|
| ActiveControllerCount | ≠ 1 | Split brain |
| Consumer lag | Рост >2x за 5 минут | Каскадный отказ |
| RequestQueueSize | >500 постоянно | Брокер не справляется |
| GC pause time | >200ms | Потребители отваливаются по таймауту |
| OfflineLogDirectoryCount | >0 | Потеря раздела данных |

---

## 7. Заключение: принципы предотвращения узких мест

1. **Мониторинг распределения, а не средних.** Средний CPU по кластеру 20% при одном брокере на 99% — катастрофа, которую скрывает агрегация.

2. **Page cache — основной ресурс Kafka.** Память «свободна» не потому, что не используется, а потому, что используется page cache'ем. Планируйте объём памяти от объёма «горячих» данных, а не от heap.

3. **Ключ партиционирования — архитектурный контракт.** Любое изменение должно проходить проверку на перекос в CI с реальными распределениями данных.

4. **Партиции — не бесплатный ресурс.** Больше партиций ≠ лучше. Оптимальное число = требуемый параллелизм + небольшой запас.

5. **JVM — фундамент.** Неправильный GC collector или размер heap могут убить производительность быстрее, чем медленный диск.

6. **Диагностика — послойная.** Не начинайте с async-profiler, если проблема видна через `kafka-consumer-groups.sh`. Идите от простых метрик к сложным инструментам.

---

## Источники

1. **Confluent, «Apache Kafka Scaling Best Practices: 10 Ways to Avoid Bottlenecks»** — confluent.io/learn/kafka-scaling-best-practices/ (Tier 1, официальный вендор)
2. **Instana Engineering Blog, «Apache Kafka and the Page Cache»** — Benjamin Kan, b3nk4n.github.io/posts/kafka-page-cache/ (Tier 2, production experience, 12 GB/s throughput)
3. **Apache Kafka Documentation, «Monitoring»** — kafka.apache.org/37/operations/monitoring/ (Tier 1, официальная документация)
4. **Michal Drozd, «One Partition at 99% CPU: Stop Kafka Hotspots Before They Reach Production»** — michal-drozd.com/en/blog/kafka-partition-skew-contracts/ (Tier 2, production incident + CI contract approach)
5. **Redpanda Guides, «Kafka Optimization Best Practices»** — redpanda.com/guides/kafka-performance-kafka-optimization (Tier 2, vendor guide)
6. **Azul, «JVM Secrets for Better Kafka Performance»** — azul.com/blog/jvm-secrets-for-better-kafka-performance/ (Tier 2, JVM-vendor expertise)
7. **Conduktor, «Kafka JVM Tuning: G1GC vs ZGC in Production»** — conduktor.io/blog/kafka-jvm-tuning-g1gc-vs-zgc-production (Tier 2)
8. **Baeldung, «A Guide to async-profiler»** — baeldung.com/java-async-profiler (Tier 2, технический обзор)
9. **Apache Kafka JIRA, KAFKA-4614** — Long GC pause caused by mmap objects for OffsetIndex (Tier 1, core project issue)
10. **DevOps.aibit.im, «Troubleshooting Common Kafka Performance Bottlenecks: A Practical Handbook»** — devops.aibit.im/article/troubleshooting-kafka-performance-bottlenecks (Tier 3, практическое руководство)
11. **KindaTechnical, «Kafka JVM Tuning Heap Size and Garbage Collection»** — kindatechnical.com (Tier 2, конфигурационные best practices)
12. **GitHub, async-profiler/async-profiler** — github.com/async-profiler/async-profiler (Tier 1, исходный код инструмента)

---

*Статья 20/37 проекта «kafka». Трек 06 — Performance & Optimization. Следующая статья: отсутствует (четвёртая и последняя в треке).*
