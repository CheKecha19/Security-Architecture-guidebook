# Тюнинг производительности Apache Kafka: конфигурационные параметры, trade-off-ы и профильная настройка

**Трек:** 06 — Performance & Optimization  
**Статья:** 02 из 4  
**Статус:** done  
**Слов:** ~7600  
**Источников:** 15

---

## TL;DR

Тюнинг Kafka — это не список «10 магических параметров», а система компромиссов между **скоростью**, **долговечностью** (durability) и **консистентностью**. Главный вывод современных исследований (2025): **наибольший эффект даёт одна брокерная настройка — log.flush.interval.messages**, которую почти никто не проверяет. При значении 1 (fsync на каждое сообщение) throughput падает в 50×. Правильный дефолт (Long.MAX_VALUE) + acks=all + min.insync.replicas=2 даёт 45–70 MB/s на скромном железе. Продюсерные настройки (batch.size=256KB, linger.ms=20, max.in.flight=5) доводят этот базовый уровень до максимума. Потребительский тюнинг (fetch.min.bytes=100KB, max.poll.records=2000–5000, отказ от auto-commit) устраняет потребительский lag. Итоговая стратегия: **сначала проверь брокер → потом тюнь продюсер → потом масштабируй потребителя**.

---

## 1. Введение: почему тюнинг — это про trade-off-ы, а не про «лучшие параметры»

### 1.1 Три оси компромисса

Любое изменение конфигурационного параметра Kafka перемещает систему в пространстве трёх осей:

| Ось | Что измеряется | Пример цены |
|-----|---------------|-------------|
| **Скорость** (throughput + latency) | MB/s, records/s, p50–p99.9 latency | Потеря durability ради +30% throughput |
| **Долговечность** (durability) | Гарантия, что сообщение не потеряно при сбое | acks=all добавляет ~10–50 мс к latency per batch |
| **Консистентность** (consistency/ordering) | Гарантия порядка сообщений, exactly-once | max.in.flight=1 снижает throughput в 7.4× |

**Аналогия:** Это как настройка гоночного автомобиля. Ты можешь: а) увеличить мощность (throughput), пожертвовав надёжностью двигателя (durability); б) настроить подвеску на максимальное сцепление (consistency), потеряв в максимальной скорости. Нельзя получить всё сразу — каждая настройка двигает машину по треугольнику «скорость–надёжность–управляемость».

### 1.2 Порядок проверки: ошибка, которую совершают все

Начинающие администраторы Kafka обычно начинают тюнинг с продюсерных параметров: `batch.size`, `linger.ms`, `compression.type`. Это ошибка. Современные исследования (2025, pairwise-тестирование 186 624 конфигураций на Kafka 4.2.0) показывают: **доминирующий фактор — брокерная настройка `log.flush.interval.messages`**. При патологическом значении (1) никакой продюсерный тюнинг не поможет.

**Правильный порядок проверки:**

1. **Брокер** — проверь flush-интервал, thread pools, page cache, OS-настройки
2. **Продюсер** — настрой batching, сжатие, acks, in-flight requests
3. **Потребитель** — настрой fetch-батчи, параллелизм, offset commits
4. **OS и JVM** — оптимизируй swappiness, файловую систему, GC

---

## 2. Брокерный тюнинг: параметры, которые определяют потолок производительности

Брокер — фундамент. Если он настроен неправильно, всё остальное бессмысленно.

### 2.1 log.flush.interval.messages — убийца производительности №1

**Параметр:** `log.flush.interval.messages`
**Дефолт:** `Long.MAX_VALUE` (9223372036854775807)  
**Смысл:** Количество сообщений, после которого Kafka выполняет fsync на диск.  
**Где настраивается:** `server.properties` брокера или per-topic override через `kafka-configs.sh`.

**Реальные цифры (бенчмарк 2025, 3-брокерный KRaft-кластер, acks=all, RF=3, min.insync.replicas=2):**

| log.flush.interval.messages | Средний throughput | Средняя p99 latency |
|----------------------------|-------------------|---------------------|
| 10 000 (≈ дефолт) | 59.5 MB/s | 339 ms |
| 1 000 | 26.5 MB/s | 1 776 ms |
| 1 | 1.2 MB/s | 48 421 ms (48 секунд!) |

При `log.flush.interval.messages=1` Kafka делает fsync на каждое сообщение — и на каждую реплику. При RF=3 и acks=all это **три fsync-а на одно сообщение** перед тем, как продюсер получит подтверждение. Результат: throughput с 59.5 MB/s падает до 1.2 MB/s — **падение в 50 раз**.

**Ключевой инсайт:** Дефолтное значение `Long.MAX_VALUE` означает, что Kafka **не делает fsync явно**, полагаясь на page cache операционной системы и репликацию для durability. При `acks=all` + `min.insync.replicas=2` сообщение переживёт потерю любого одного брокера **даже без fsync** — в этом весь смысл репликации.

**Как проверить текущее значение:**

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --entity-type brokers --entity-default \
  | grep flush
```

Если кто-то в команде выставил это значение в 1 «для надёжности» — он создал бутылочное горлышко в 50×, которое никакой продюсерный тюнинг не исправит.

### 2.2 Thread Pools: num.network.threads и num.io.threads

**num.network.threads** (дефолт: 3) — потоки, принимающие сетевые запросы от клиентов (produce, fetch) и других брокеров.

**num.io.threads** (дефолт: 8) — потоки, обрабатывающие запросы (чтение/запись на диск).

**Рекомендации:**

| Параметр | Стартовое значение | Высоконагруженный кластер | Как определить, что мало |
|----------|-------------------|--------------------------|--------------------------|
| `num.network.threads` | = кол-ву CPU cores / 4 | 8–12 | `NetworkProcessorAvgIdlePercent` ≈ 0% |
| `num.io.threads` | 8 × кол-во дисков | 16–24 | Растёт `RequestQueueSize` |

**Метрика для мониторинга:** `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent`. Если этот показатель равен 0% — все сетевые потоки заняты на 100%, нужно увеличивать.

**Антипаттерн:** Слишком много потоков на слабом CPU. Бенчмарк на 2-CPU машинах показал, что 8 network threads работают хуже, чем 4 — начинается contention и context switching overhead.

**queued.max.requests** (дефолт: 500) — максимальный размер очереди запросов. Если очередь заполнена, сетевые потоки блокируются. Увеличивать при пиковых нагрузках до 1000–2000.

### 2.3 Сетевые буферы: socket.send.buffer.bytes и socket.receive.buffer.bytes

**Дефолт:** 102400 байт (100 KB) — слишком мало для production.

**Оптимальный расчёт** через bandwidth-delay product:

```
buffer_size = bandwidth (бит/с) × round-trip-time (секунды)
```

Для 10 Gbps сети и RTT = 1 мс:
```
buffer = 10 × 10^9 × 0.001 = 10 × 10^6 бит = 1.25 MB
```

**Рекомендация:** Установить 1–16 MB (1048576–16777216):

```properties
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
```

### 2.4 Репликационный тюнинг: num.replica.fetchers

**Параметр:** `num.replica.fetchers` (дефолт: 1)  
**Смысл:** Количество потоков, которыми follower брокер читает данные с лидера.

**Рекомендации:**
- При высокой партиционности (>1000 partitions на брокер): **4–8**
- При кросс-DC репликации: **8–16**
- При стандартной нагрузке: **2–4**

Увеличение этого параметра напрямую ускоряет репликацию, снижая риск ISR shrink-а (выпадения реплики из in-sync replicas) при пиковых нагрузках.

### 2.5 Лог-сегменты: log.segment.bytes и log.roll.ms

**log.segment.bytes** (дефолт: 1 GB = 1073741824):

| Размер сегмента | Плюсы | Минусы | Сценарий |
|----------------|-------|--------|----------|
| 256–512 MB | Быстрая очистка, компакция | Больше file handles | log compaction, частая ротация |
| 1 GB (дефолт) | Баланс | — | Общий случай |
| 2 GB | Меньше файлов, меньше file handles | Задержка очистки (retention) | Высокая партиционность |

Для high-throughput топиков со скоростью записи >100 MB/s рекомендуется **2 GB** — меньше накладных расходов на создание новых файлов.

**log.roll.ms** (дефолт: 604800000 = 7 дней) — максимальное время до принудительного «переворота» активного сегмента. В высоконагруженных кластерах сегмент заполняется по размеру (log.segment.bytes) задолго до истечения этого времени, поэтому параметр редко требует изменения.

### 2.6 Политики очистки и retention

**log.cleanup.policy** (дефолт: `delete`):

| Политика | Когда использовать | Влияние на производительность |
|----------|-------------------|------------------------------|
| `delete` | Временные данные, логи, метрики | Минимальный overhead |
| `compact` | Журнал событий с ключами, CDC, материализованные представления | Дополнительные CPU + I/O на фоновую компакцию |
| `compact,delete` | Компакция + временной лимит | Наибольший overhead, но контролируемый размер |

**log.retention.bytes** и **log.retention.ms** управляют тем, как долго и сколько данных хранится. Меньше retention → меньше дискового пространства → быстрее очистка, но риск потери данных при длительном потребительском lag-е.

---

## 3. Trade-off-ы: скорость vs durability, скорость vs consistency

### 3.1 Durability vs Throughput: битва acks

Параметр `acks` в конфигурации продюсера — классический пример trade-off-а:

| acks | Durability | Throughput | Когда использовать |
|------|-----------|------------|-------------------|
| `acks=0` | Никакой — продюсер не ждёт ответа | Максимальный | Метрики, логи, где потеря нескольких сообщений допустима |
| `acks=1` | Лидер подтвердил запись в свой лог | Высокий (≈90–95% от acks=0) | Аналитика, неаудируемые потоки |
| `acks=all` (или `acks=-1`) | Все ISR подтвердили запись | База (≈70–85% от acks=1) | Финансовые транзакции, аудит, CDC |

**Acks=all требует min.insync.replicas!** Без этого параметра (или при значении 1) `acks=all` деградирует до поведения `acks=1` при первом же ISR shrink-е. Правильная конфигурация durable-топика:

```properties
# Топик
min.insync.replicas=2  # RF=3 → допустимо падение 1 брокера
# Продюсер
acks=all
enable.idempotence=true  # дефолт с Kafka 3.0+
```

### 3.2 Consistency vs Throughput: max.in.flight.requests.per.connection

**max.in.flight.requests.per.connection** (дефолт: 5) — количество параллельных неподтверждённых запросов.

**Историческая ловушка:** Многие руководства рекомендуют `max.in.flight.requests.per.connection=1` для сохранения порядка сообщений. Этот совет устарел в 2017 году с выходом Kafka 0.11.

С идемпотентным продюсером (`enable.idempotence=true`, дефолт с Kafka 3.0) Kafka гарантирует порядок внутри партиции с **до 5 in-flight запросов** за счёт sequence numbers. Установка inflight=1 превращает протокол в stop-and-wait: отправь батч → жди подтверждения от ВСЕХ реплик → отправь следующий.

**Бенчмарк (acks=all, 2025):**

| max.in.flight | Средний throughput | vs худшего |
|--------------|-------------------|-----------|
| 5 (дефолт) | 45.5 MB/s | 7.4× |
| 2 | 25.9 MB/s | 4.2× |
| 1 | 6.1 MB/s | 1× |

**Вывод:** Не ставь `max.in.flight.requests.per.connection=1` без крайней необходимости. Ты теряешь 7.4× throughput.

### 3.3 Балансировка partition count и replication factor

| Параметр | Больше → | Меньше → |
|----------|---------|---------|
| **Количество партиций** | + параллелизм потребления, + throughput | − overhead на брокере, − file handles, − memory |
| **Replication Factor** | + durability, + availability | + throughput (меньше сетевого трафика), − disk usage |

**Правило:** Не делай больше партиций, чем количество consumer-ов в группе. Исключение: нужно больше throughput от продюсера через параллельную запись. Типичный production RF = 3 при min.insync.replicas = 2.

---

## 4. Профильный тюнинг: high throughput vs low latency

Kafka-кластеры редко оптимизируются под всё сразу. Обычно выбирается **один профиль**, и настройки выставляются под него.

### 4.1 Профиль «High Throughput» (максимальная пропускная способность)

**Цель:** Максимальный MB/s или records/s при допустимой задержке p99 до 500–1000 мс.

**Сценарии:** Пакетная загрузка данных, ETL, data lake ingestion, логирование.

**Конфигурация:**

```properties
# === BATCHING (продюсер) ===
batch.size=262144          # 256 KB (можно до 1 MB)
linger.ms=10-100           # Ждать заполнения батча до 100 мс
buffer.memory=134217728    # 128 MB — чтобы не блокироваться при пиках

# === СЖАТИЕ ===
compression.type=zstd      # Лучшее сжатие (20-30% лучше lz4)
# Альтернатива: lz4 — быстрее, но меньше сжатие

# === ПОДТВЕРЖДЕНИЯ ===
acks=1                     # Только лидер — не ждём репликацию
# Или acks=all если данные важны, но с max.in.flight=5

# === IN-FLIGHT ===
max.in.flight.requests.per.connection=5

# === БРОКЕР ===
num.replica.fetchers=8     # Быстрая репликация
num.io.threads=16-24       # Много I/O потоков
log.segment.bytes=2147483648  # 2 GB сегменты — меньше файловых операций
```

**Ожидаемые результаты:**
- Throughput: 300–600 MB/s на брокер (NVMe, 25 GbE)
- p99 latency: 200–1000 мс (компромисс ради батчинга)

### 4.2 Профиль «Low Latency» (минимальная задержка)

**Цель:** p99 < 20 мс, p95 < 10 мс.

**Сценарии:** Трейдинг, real-time аналитика, fraud detection, игровые серверы.

**Конфигурация:**

```properties
# === BATCHING (продюсер) ===
batch.size=32768-65536      # 32-64 KB — маленькие батчи
linger.ms=0-5               # Минимальная задержка на накопление
# Kafka 4.0+ дефолт linger.ms=5 (KIP-1030)

# === СЖАТИЕ ===
compression.type=lz4        # Минимальные накладные расходы CPU
# Или snappy — тоже быстро. zstd добавляет ~2-5 мс на сжатие/разжатие

# === ПОДТВЕРЖДЕНИЯ ===
acks=1                      # Только лидер — минимизация round-trip
# Для аудиторских данных можно acks=all, но latency вырастет

# === IN-FLIGHT ===
max.in.flight.requests.per.connection=3-5  # Параллельные запросы важны

# === БРОКЕР ===
num.network.threads=8-12    # Быстрая обработка запросов
queued.max.requests=1000    # Чтобы не блокироваться

# === ПОТРЕБИТЕЛЬ ===
fetch.min.bytes=1           # Не ждать накопления — забирать сразу
fetch.max.wait.ms=10-50     # Минимальное ожидание
max.poll.records=100-200    # Маленькие poll-ы

# === OS ===
vm.swappiness=1             # Page cache не должен свопиться
# см. раздел 6
```

**Ожидаемые результаты:**
- p99 latency: 5–20 мс
- Throughput: 50–150 MB/s на брокер (меньше из-за маленьких батчей)

### 4.3 Профиль «Balanced» (золотая середина)

```properties
batch.size=131072           # 128 KB
linger.ms=5-20              # Разумный компромисс
compression.type=zstd       # Хорошее сжатие без большого overhead
acks=all                    # Данные важны
max.in.flight.requests.per.connection=5
fetch.min.bytes=102400      # 100 KB для потребителя
fetch.max.wait.ms=500       # Стандартное ожидание
```

**Ожидаемые результаты:**
- Throughput: 150–300 MB/s на брокер
- p99 latency: 20–100 мс

### 4.4 Сводная таблица профилей

| Параметр | High Throughput | Low Latency | Balanced |
|----------|----------------|-------------|----------|
| `batch.size` | 256 KB – 1 MB | 32–64 KB | 128 KB |
| `linger.ms` | 10–100 | 0–5 | 5–20 |
| `compression.type` | zstd | lz4/snappy | zstd |
| `acks` | 1 (или all) | 1 | all |
| `max.in.flight` | 5 | 3–5 | 5 |
| `fetch.min.bytes` | 1048576 (1 MB) | 1 | 102400 (100 KB) |
| `fetch.max.wait.ms` | 500–1000 | 10–50 | 500 |
| `max.poll.records` | 2000–5000 | 100–200 | 500–1000 |
| `num.io.threads` | 16–24 | 8–12 | 12–16 |
| `num.replica.fetchers` | 8 | 2–4 | 4 |

---

## 5. Продюсерный тюнинг: от батчинга до idempotence

### 5.1 Batching: batch.size и linger.ms

**batch.size** (дефолт: 16384 = 16 KB):

Чем больше батч — тем меньше сетевых вызовов и тем эффективнее сжатие. Но больше latency и memory usage.

Расчёт memory для продюсера:

```
producer_memory = batch.size × num_partitions × max.in.flight.requests.per.connection
```

Пример: 256 KB × 100 партиций × 5 in-flight = **128 MB** только на батч-буферы.

**linger.ms** (дефолт: 0 до Kafka 4.0, 5 начиная с Kafka 4.0):

Значение 0 означает «отправлять немедленно» — батч может быть почти пустым. Значение 20 мс даёт +30–50% throughput при минимальном влиянии на latency (Kafka 4.0, KIP-1030).

**Аномалия:** linger.ms=100 работает **хуже** 20 мс при acks=all. Причина: каждый батч уже тратит десятки миллисекунд в репликационном pipeline — 100 мс ожидания приводит к тому, что продюсер простаивает.

### 5.2 Сжатие: compression.type

| Алгоритм | Сжатие | CPU cost (compress) | CPU cost (decompress) | Когда |
|----------|--------|--------------------|-----------------------|-------|
| **none** | 1× (нет) | 0 | 0 | Low-latency, данные уже сжаты (ProtoBuf/JSON small) |
| **snappy** | 1.5–2× | Низкий | Низкий | Баланс, legacy |
| **lz4** | 1.7–2.5× | Очень низкий | Очень низкий | Low-latency, high-throughput |
| **zstd** | 2–3.5× | Средний | Низкий | **Рекомендуемый дефолт (Kafka 3.0+)** |
| **gzip** | 2–4× | Высокий | Средний | Не рекомендуется в 2025 |

zstd даёт на 20–30% лучшее сжатие, чем lz4, при сопоставимой скорости на современном оборудовании. gzip стоит избегать — слишком высокие CPU-затраты на сжатие при минимальном выигрыше по сравнению с zstd.

### 5.3 Idempotence и транзакции

**enable.idempotence** — с Kafka 3.0+ включён по умолчанию. Предотвращает дублирование сообщений при retry, назначая каждому сообщению sequence number. Performance overhead: 1–3%.

**Транзакции** (`transactional.id`) дают exactly-once семантику для записи в несколько партиций:

```properties
enable.idempotence=true
transactional.id=my-unique-id
transaction.timeout.ms=900000  # 15 мин — дефолт
acks=all
```

Каждый транзакционный ID привязан к одному продюсеру. Если старый продюсер «завис», а новый пытается писать с тем же ID — Kafka «огораживает» (fences) старый инстанс по epoch number.

**Performance overhead транзакций:**
- Дополнительные маркерные сообщения (commit markers) в лог
- Дополнительный round-trip для commit/abort
- Суммарно: +5–15% latency, −10–20% throughput

### 5.4 retries и delivery.timeout.ms

```properties
retries=2147483647          # = Integer.MAX_VALUE = «бесконечно»
delivery.timeout.ms=120000  # 2 минуты — общий таймаут доставки
```

При включённой идемпотентности retries можно (и нужно) ставить в максимум — дублирования не будет благодаря sequence numbers. `delivery.timeout.ms` ограничивает общее время — если за это время сообщение не доставлено, продюсер выбрасывает исключение.

### 5.5 buffer.memory

**Параметр:** `buffer.memory` (дефолт: 33554432 = 32 MB) — общий объём памяти для буферов ожидающих отправки сообщений.

Когда буфер заполнен, `send()` блокируется до `max.block.ms` (дефолт: 60 000 мс). Для high-throughput продюсеров увеличить до **128–256 MB**:

```properties
buffer.memory=268435456  # 256 MB
```

---

## 6. Потребительский тюнинг: fetch-батчи, параллелизм и offset-commits

### 6.1 Fetch-батчинг: fetch.min.bytes и fetch.max.wait.ms

Эти параметры — зеркальное отражение продюсерных `batch.size` и `linger.ms`, но для потребительской стороны:

| Параметр | Дефолт | High Throughput | Low Latency |
|----------|--------|----------------|-------------|
| `fetch.min.bytes` | 1 | 100 000–1 048 576 (100 KB–1 MB) | 1 |
| `fetch.max.wait.ms` | 500 | 500–1000 | 10–50 |

**Механика:** Потребитель делает fetch-запрос к брокеру. Брокер ждёт, пока накопится минимум `fetch.min.bytes` данных, но не дольше `fetch.max.wait.ms`. Если накопилось раньше — отвечает сразу.

**Для throughput:** Большие значения сокращают количество network round-trips и CPU overhead.
**Для latency:** Маленькие значения минимизируют время ожидания на брокере.

### 6.2 Управление размером poll: max.poll.records, max.partition.fetch.bytes, fetch.max.bytes

| Параметр | Дефолт | Смысл |
|----------|--------|-------|
| `max.poll.records` | 500 | Максимум записей за один `poll()` |
| `max.partition.fetch.bytes` | 1 048 576 (1 MB) | Максимум данных с одной партиции за fetch |
| `fetch.max.bytes` | 52 428 800 (50 MB) | Максимум данных суммарно за один fetch |

**High throughput:** `max.poll.records=2000-5000` — большие батчи для пакетной обработки.

**Low latency:** `max.poll.records=100-200` — маленькие poll-ы, минимальная задержка между получением и обработкой.

**Важно про память:**

```
memory_upper_bound = NUMBER_OF_BROKERS × fetch.max.bytes
                   + NUMBER_OF_PARTITIONS × max.partition.fetch.bytes
```

Пример: 3 брокера × 50 MB + 100 партиций × 1 MB = **250 MB** потенциального потребления памяти потребителем при worst-case сценарии.

### 6.3 Параллелизм: количество consumer-ов vs количество партиций

Партиция — атомарная единица параллелизма для потребителей. Одна партиция обрабатывается ровно одним consumer-ом в группе. Следовательно:

```
N_consumers_эффективных = min(N_consumers_в_группе, N_partitions)
```

Добавление consumer-ов сверх количества партиций бесполезно — они будут idle. Но они могут служить hot standby на случай отказа активных consumer-ов.

### 6.4 Offset commits: ручное управление vs auto-commit

**enable.auto.commit** (дефолт: `true`) — автоматический коммит оффсетов каждые `auto.commit.interval.ms` (дефолт: 5000 мс).

**Проблемы auto-commit:**

- **Data loss:** Сообщение закоммичено, но ещё не обработано → краш → сообщение потеряно при рестарте.
- **Data duplication:** Сообщения обработаны, но коммит не успел → краш → повторная обработка при рестарте.

**Для production систем с требованиями к durability:**

```properties
enable.auto.commit=false
# Коммитить вручную ПОСЛЕ завершения обработки батча:
# consumer.commitSync() — синхронно, надёжно, блокирует
# consumer.commitAsync() — асинхронно, быстрее, но может потерять коммит при сбое
```

**Паттерн для ручного коммита:**

```java
// Java-пример
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);  // обработать запись
    }
    // Коммитить после обработки ВСЕГО батча
    consumer.commitSync();
}
```

### 6.5 Rebalancing: session.timeout.ms, heartbeat.interval.ms, max.poll.interval.ms

| Параметр | Дефолт | Смысл | Рекомендация |
|----------|--------|-------|-------------|
| `session.timeout.ms` | 45 000 (45 с) | Если брокер не получил heartbeat за это время — consumer считается мёртвым | 30 000–60 000 |
| `heartbeat.interval.ms` | 3 000 (3 с) | Частота heartbeat-ов | ≈ session.timeout / 10 |
| `max.poll.interval.ms` | 300 000 (5 мин) | Максимальное время между poll-ами до отключения от группы | >= max времени обработки батча |

**Важно:** Если обработка одного `poll()` занимает больше `max.poll.interval.ms`, consumer будет исключён из группы и произойдёт **rebalance** — перераспределение партиций между оставшимися consumer-ами. Во время rebalance-а группа не потребляет — это dead time. Решение: либо увеличить `max.poll.interval.ms`, либо уменьшить `max.poll.records`.

**Kafka 4.0+** (новый consumer protocol) сокращает время rebalance-а на 50–70% по сравнению с классическим протоколом.

### 6.6 Статические группы: group.instance.id

```properties
group.instance.id=consumer-1  # постоянный ID инстанса
```

Со статическим членством consumer не покидает группу при временной недоступности — его партиции ждут его возврата. Это **предотвращает лишние rebalance-ы** при кратковременных сетевых сбоях или GC-паузах.

### 6.7 Consumer Lag — главная метрика здоровья потребления

Lag = разница между последним записанным сообщением в партиции и позицией потребителя. Растущий lag означает, что потребитель не успевает за продюсером.

**Причины и решения:**

| Причина | Решение |
|---------|---------|
| Мало consumer-ов | Добавить consumer-ов (но не больше партиций) |
| Медленная обработка | Оптимизировать код, увеличить `max.poll.records` |
| Маленькие fetch-батчи | Увеличить `fetch.min.bytes` |
| Частые rebalance-ы | Настроить `group.instance.id`, таймауты |

---

## 7. OS-уровень и JVM-тюнинг

### 7.1 Linux kernel parameters

Все настройки добавляются в `/etc/sysctl.d/99-kafka.conf`.

```ini
# === ВИРТУАЛЬНАЯ ПАМЯТЬ ===
vm.swappiness=1                     # Минимальный swap — Kafka держит данные в page cache
vm.dirty_background_ratio=5         # Начинать сброс dirty pages при 5% RAM
vm.dirty_ratio=60                   # Максимум 60% RAM под dirty pages
vm.max_map_count=262144             # Для mmap-логов (log segments)
vm.overcommit_memory=1              # Разрешить overcommit для malloc
vm.min_free_kbytes=1048576          # 1 GB резерва для ядра

# === СЕТЬ ===
net.core.rmem_max=16777216          # 16 MB — максимальный read buffer
net.core.wmem_max=16777216          # 16 MB — максимальный write buffer
net.core.netdev_max_backlog=30000   # Очередь пакетов на входе
net.core.somaxconn=32768            # Максимальная очередь TCP-соединений

net.ipv4.tcp_rmem=4096 65536 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
net.ipv4.tcp_fin_timeout=30         # Быстрый FIN timeout
net.ipv4.tcp_keepalive_time=60      # TCP keepalive через 60 с
net.ipv4.tcp_tw_reuse=1             # Переиспользование TIME_WAIT сокетов

# TCP Congestion Control — BBR (лучший для современных сетей)
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_slow_start_after_idle=0
```

**Применить:**

```bash
sudo sysctl -p /etc/sysctl.d/99-kafka.conf
```

### 7.2 Transparent Huge Pages (THP)

**Проблема:** THP (Transparent Huge Pages) вызывают многоминутные GC-паузы в JVM — критично для Kafka.

**Решение — всегда отключать:**

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

Сделать перманентным через systemd unit-файл (см. исходник конфигурации).

### 7.3 Файловая система: XFS и mount options

**XFS** рекомендуется для Kafka вместо ext4 — лучше работает с большими файлами и параллельной записью.

```bash
# В /etc/fstab:
UUID=... /data/kafka-logs xfs defaults,noatime,nodiratime,largeio,inode64,swalloc 0 0
```

- `noatime,nodiratime` — отключение обновления access time (экономит +5–10% write throughput)
- `largeio` — оптимизация для больших I/O операций
- `inode64` — 64-битные inode (для больших разделов)
- `swalloc` — оптимизация размещения stripe width

### 7.4 I/O Scheduler

Для NVMe-дисков:

```bash
echo none > /sys/block/nvme1n1/queue/scheduler  # NVMe — не нужен доп. scheduling
```

Для SATA SSD: `mq-deadline`.

### 7.5 File Descriptors

Kafka открывает минимум один file descriptor на партицию (сегмент лога + индекс). Кластер с 5000 партиций легко потребляет 30 000+ file descriptors. Дефолтный лимит (1024) катастрофически недостаточен.

```bash
# /etc/security/limits.d/kafka.conf
kafka soft nofile 1000000
kafka hard nofile 1000000
kafka soft nproc 32768
kafka hard nproc 32768
```

Для systemd — дублировать в unit-файле:
```ini
[Service]
LimitNOFILE=1000000
LimitNPROC=32768
```

### 7.6 JVM и Garbage Collection

**Heap size:** 5–8 GB. Больше — редко помогает, чаще увеличивает GC-паузы. Kafka использует page cache OS, а не heap, для хранения данных.

```bash
export KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
```

**G1GC** — дефолт для Kafka 4.0+:

```bash
export KAFKA_JVM_PERFORMANCE_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:+DisableExplicitGC
  -XX:+AlwaysPreTouch
"
```

- `MaxGCPauseMillis=20` — цель: GC не дольше 20 мс (важно для low-latency)
- `InitiatingHeapOccupancyPercent=35` — начинать concurrent marking раньше (по умолчанию 45%)
- `AlwaysPreTouch` — выделить и «потрогать» всю heap-память при старте, избегая page faults на ходу

**ZGC** — для ultra-low latency (<10 мс GC-пауз):

```bash
-XX:+UseZGC
-XX:ZCollectionInterval=10  # Собирать каждые 10 с
```

ZGC даёт GC-паузы <1 мс, но потребляет больше CPU и памяти. Рекомендуется только когда G1GC не укладывается в latency SLO.

---

## 8. Практический чеклист: порядок тюнинга Kafka-кластера

### Шаг 1: Проверь брокер (5 минут)

```bash
# 1. Flush interval
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --entity-type brokers --entity-default | grep flush

# 2. File descriptors
cat /proc/$(pgrep -f kafka.Kafka)/limits | grep "open files"

# 3. Swap
cat /proc/sys/vm/swappiness

# 4. THP
cat /sys/kernel/mm/transparent_hugepage/enabled
# Должно быть: always madvise [never]

# 5. Mount options
mount | grep kafka-logs
# Должно быть: noatime,nodiratime

# 6. I/O scheduler
cat /sys/block/$(df /data/kafka-logs | tail -1 | awk '{print $1}' | xargs basename)/queue/scheduler
```

### Шаг 2: Настрой продюсера

1. Включи/проверь `enable.idempotence=true` (скорее всего уже дефолт)
2. Выстави `batch.size` под профиль (256 KB для throughput, 32 KB для latency)
3. Выстави `linger.ms` (20 для throughput, 5 для latency — Kafka 4.0+ дефолт 5)
4. Выбери `compression.type` (zstd для большинства, lz4 для latency)
5. Определи `acks` (all для важных данных, 1 для throughput-метрик)
6. Не ставь `max.in.flight.requests.per.connection=1` без крайней нужды

### Шаг 3: Настрой потребителя

1. Отключи `enable.auto.commit=false` для production
2. Настрой `fetch.min.bytes` + `fetch.max.wait.ms` под профиль
3. Настрой `max.poll.records` (2000–5000 для throughput, 100–200 для latency)
4. Убедись, что `max.poll.interval.ms` > времени обработки одного батча
5. Используй `group.instance.id` для предотвращения лишних rebalance-ов
6. Количество consumer-ов ≤ количество партиций

### Шаг 4: OS и JVM

1. Примени `sysctl` настройки (swappiness=1, TCP buffers=16 MB, BBR)
2. Отключи THP
3. Проверь XFS + noatime на data-директории
4. Выстави file descriptor limit = 1 000 000
5. Настрой JVM: G1GC с MaxGCPauseMillis=20

### Шаг 5: Измеряй и итерируй

После каждого изменения:
1. Сними baseline-метрики (throughput, p50/p95/p99 latency, consumer lag)
2. Измени ОДИН параметр
3. Сними новые метрики
4. Сравни, запиши вывод
5. Повтори

---

## 9. Реальный пример: от 0.42 MB/s до 70.23 MB/s

Рассмотрим эволюцию одного и того же 3-брокерного KRaft-кластера (2 CPU, 2 GB RAM на брокер, Kafka 4.2.0) при разных настройках, все с **acks=all, RF=3, min.insync.replicas=2, record.size=1KB**.

### Сценарий A: Полностью неправильная конфигурация

```properties
log.flush.interval.messages=1    # fsync на КАЖДОЕ сообщение
batch.size=16384                 # 16 KB (дефолт)
linger.ms=0                      # отправлять немедленно
max.in.flight.requests.per.connection=1  # stop-and-wait
```

Результат: **0.42 MB/s, p99 = 72 609 мс (72 секунды!)**

### Сценарий B: Частично исправленный

```properties
log.flush.interval.messages=1    # всё ещё fsync на каждое
batch.size=262144                # 256 KB
linger.ms=5                      # ждать 5 мс
max.in.flight=5                  # параллелизм
```

Результат: **9.77 MB/s, p99 = 5 333 мс** — улучшение в 23× по throughput, но всё ещё ужасно.

### Сценарий C: Исправлен брокер

```properties
log.flush.interval.messages=10000  # ≈ дефолт (нет fsync)
batch.size=16384                   # 16 KB (дефолт)
linger.ms=20                       # ждать 20 мс
max.in.flight=5                    # параллелизм
```

Результат: **46.57 MB/s, p99 = 508 мс** — прорыв! Без изменения продюсерных настроек, только исправление брокера.

### Сценарий D: Полный тюнинг

```properties
log.flush.interval.messages=10000  # ≈ дефолт
batch.size=65536                   # 64 KB
linger.ms=100                      # ждать 100 мс
max.in.flight=1                    # но с 24 партициями
```

Результат: **70.23 MB/s, p99 = 81 мс** — оптимум.

**Вывод:** Исправление одной брокерной настройки дало прирост с 0.42 до 46.57 MB/s — **в 111 раз**. Весь дополнительный продюсерный тюнинг добавил ещё +50%. Порядок имеет значение: брокер → продюсер → масштабирование.

---

## 10. Распространённые ошибки и антипаттерны

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `log.flush.interval.messages=1` | Throughput < 10 MB/s при любых настройках | Вернуть дефолт (Long.MAX_VALUE) |
| `max.in.flight=1` без необходимости | Throughput в 3–7× ниже возможного | Вернуть 5 (при enable.idempotence=true) |
| `vm.swappiness=60` (дефолт) | Случайные latency-спайки до секунд | Установить 1 |
| THP не отключены | GC-паузы по несколько минут | Отключить (never) |
| ext4 вместо XFS | Просадки write throughput на больших файлах | Переформатировать в XFS |
| file descriptor limit = 1024 | «Too many open files» на пиках | Установить 1 000 000 |
| `auto.commit.enable=true` в production | Потеря или дублирование сообщений | Перейти на ручной коммит |
| Consumer-ов больше, чем партиций | Idle consumer-ы, нет прироста throughput | Не превышать N партиций |
| Сравнивать только среднюю latency | Пропуск хвостовых задержек (p99–p99.9) | Всегда измерять процентили |
| Менять больше одного параметра за раз | Невозможно понять, что сработало | Менять по одному, замерять |

---

## Ссылки по теме

- [Основы производительности Kafka: метрики и бенчмаркинг](../06-performance/01-performance-basics.md)
- [Масштабирование Kafka: партиционирование и шардирование](../06-performance/03-scalability.md) (скоро)
- [Бутылочные горлышки и диагностика производительности](../06-performance/04-bottlenecks.md) (скоро)
- [Архитектура Kafka: как это работает](../02-basics/02-how-it-works.md)
- [Ключевые концепции и гарантии](../02-basics/03-core-concepts.md)

---

## Источники

1. **Apache Kafka Documentation** — Producer/Broker/Consumer configs reference. https://kafka.apache.org/documentation/
2. **Strimzi Blog** — Optimizing Kafka brokers, PaulRMellor, 2021. https://strimzi.io/blog/2021/06/08/broker-tuning/
3. **Strimzi Blog** — Optimizing Kafka producers, PaulRMellor, 2020. https://strimzi.io/blog/2020/10/15/producer-tuning/
4. **Strimzi Blog** — Optimizing Kafka consumers, PaulRMellor, 2021. https://strimzi.io/blog/2021/01/07/consumer-tuning/
5. **Gist: sderosiaux** — "I tested 186,624 Kafka configurations with acks=all." 2025. https://gist.github.com/sderosiaux/4b99d1c579175824d3085a8c869dc778
6. **Conduktor** — Kafka Performance Tuning Guide, 2024–2025. https://www.conduktor.io/glossary/kafka-performance-tuning-guide
7. **Red Hat** — Kafka configuration tuning (Streams for Apache Kafka docs). https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/
8. **Redpanda** — Kafka performance tuning strategies & tips. https://www.redpanda.com/guides/kafka-performance-kafka-performance-tuning
9. **KLogic** — Optimize Kafka Throughput & Latency: Performance Tuning Guide. https://klogic.io/guides/optimize-kafka-throughput-latency/
10. **AutoMQ Blog** — Apache Kafka Performance Tuning: Tips & Best Practices. https://www.automq.com/blog/apache-kafka-performance-tuning-tips-best-practices
11. **IBM Community** — How to Improve Kafka Performance: A Comprehensive Guide, Devesh Singh, 2024. https://community.ibm.com/community/user/blogs/devesh-singh/2024/09/26/
12. **ActiveWizards** — Advanced Kafka Performance Tuning. https://activewizards.com/blog/advanced-kafka-performance-tuning
13. **KindaTechnical** — Kafka OS Tuning: File Descriptors and Network Settings. https://kindatechnical.com/kafka-streams/kafka-os-tuning-file-descriptors-and-network-settings.html
14. **Cloudera Documentation** — Virtual Memory Handling for Kafka. https://docs.cloudera.com/cdp-private-cloud-base/7.1.9/kafka-performance-tuning/
15. **EdgeInData** — Optimize Kafka Threads: Network, I/O, and Background Configurations. https://www.edgeindata.com/kafka/optimize-kafka-threads-network-io-and-background-configurations
