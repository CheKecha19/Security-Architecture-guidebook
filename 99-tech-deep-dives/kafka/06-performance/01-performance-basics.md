# Основы производительности Apache Kafka: метрики, типичные числа и методология бенчмаркинга

**Трек:** 06 — Performance & Optimization  
**Статья:** 01 из 4  
**Статус:** done  
**Слов:** ~6500  
**Источников:** 13

---

## TL;DR

Производительность Kafka измеряется тремя основными метриками: **throughput** (пропускная способность — MB/s и сообщений/с), **latency** (задержка — от p50 до p99.9) и **использование ресурсов** (CPU, память, диск, сеть). Типичный production-брокер на современном NVMe-железе выдаёт **300–600 MB/s write + read** при 3× репликации, задержка p99 укладывается в **5–20 мс** при нагрузке до 200 MB/s. Знаменитый бенчмарк LinkedIn (2014) показал **2 миллиона записей/с на трёх дешёвых машинах**, а современный Confluent-бенчмарк (2023–2024) — **605 MB/s пиковый throughput** с p99 = 5 мс на AWS i3en.2xlarge. Бенчмаркинг Kafka — это не разовая акция, а циклический процесс: определить baseline → менять один параметр → замерять → анализировать → повторять. Ключевое правило: **всегда измеряй процентили, а не только средние** — средняя задержка может быть 2 мс, а p99 — 500 мс, и именно хвостовые задержки убивают пользовательский опыт.

---

## 1. Зачем вообще измерять производительность Kafka

Kafka часто описывают как «быструю из коробки», и это правда — архитектурные решения (page cache, zero-copy, sequential I/O, batching) делают её одной из самых производительных распределённых систем. Но «быстрая» — не число. Без конкретных цифр невозможно:

- **Планировать мощность (capacity planning).** Сколько брокеров нужно, чтобы обрабатывать 1 TB данных в день? Какой retention реально выдержит железо?
- **Обнаруживать деградацию.** После обновления Kafka с 3.6 на 4.0 latency выросла на 30% — это баг или особенность твоего workload-а?
- **Доказывать бизнесу.** «Наша система обрабатывает 100 000 событий/с с p99 < 10 мс» — это язык, понятный CTO.
- **Сравнивать облачных провайдеров.** AWS MSK vs Confluent Cloud vs self-hosted — без бенчмарков ты просто гадаешь.

**Аналогия:** Ты покупаешь спорткар. Производитель говорит «быстрый». Но тебе нужны конкретные цифры: 0–100 км/ч за 3.2 с, максималка 320 км/ч, тормозной путь 35 м. С Kafka то же самое — абстрактная «производительность» бесполезна без измеримых метрик.

---

## 2. Ключевые метрики производительности

Все метрики Kafka делятся на три группы: throughput, latency и resource utilization.

### 2.1 Throughput (пропускная способность)

**Определение:** Количество данных, которое система обрабатывает за единицу времени.

Два измерения:
- **Сообщений в секунду (messages/sec или records/sec)** — важно, когда сообщения примерно одинакового размера.
- **Мегабайт в секунду (MB/s или GB/s)** — универсальная метрика, не зависящая от размера сообщения.

**Разделение по ролям:**
| Роль | Метрика | Пример |
|------|---------|--------|
| Producer throughput | `BytesInPerSec`, `MessagesInPerSec` | 150 MB/s write на брокер |
| Consumer throughput | `BytesOutPerSec` | 300 MB/s read с брокера (consumer + replication) |
| Replication throughput | `FetchFollower` request rate | Follower читает ~50 MB/s с лидера |

**Почему BytesOutPerSec > BytesInPerSec?** Каждый записанный байт может быть прочитан многократно: consumer-ами (N групп) + follower-ами (RF-1 реплик). На практике BytesOut часто в 2–5× больше BytesIn.

**Пиковая vs устойчивая пропускная способность:**
- **Peak throughput** — максимальная скорость, которую система может выдать в коротком тесте.
- **Stable/sustained throughput** — скорость, при которой consumer-ы успевают за producer-ами без бесконечно растущего consumer lag. **Именно stable throughput — это реальная production-метрика.**

> В бенчмарке Confluent 2023 года peak stable throughput составил **605 MB/s** на трёх брокерах AWS i3en.2xlarge (каждый с 2×NVMe SSD по 2.5 TB). Это примерно 200 MB/s на брокер при 3× репликации и acks=all.

### 2.2 Latency (задержка)

**Определение:** Время прохождения одного сообщения от точки А до точки Б.

Виды задержки в Kafka:

```
Producer                     Broker                     Consumer
   |                           |                           |
   |--- produce request ------>|                           |
   |                           |--- write to log --------->|
   |                           |--- replicate to ISR ---->|
   |<--- ack ------------------|                           |
   |                           |                           |
   |                           |<--- fetch request --------|
   |                           |--- return records ------->|
   |                           |                           |
   |<---------- end-to-end latency ----------------------->|
```

**Типы задержки:**

1. **Producer send latency** — от вызова `send()` до получения `ack` от брокера. Зависит от `acks`, `linger.ms`, сети.
2. **Broker processing latency** — время обработки запроса на брокере (очереди + запись на диск + репликация).
3. **End-to-end (E2E) latency** — от `producer.send()` до `consumer.poll()`. Самая важная бизнес-метрика.

**Почему процентили критичнее среднего:**

Представь систему, обрабатывающую 1000 сообщений/с:
- 990 сообщений: latency = 2 мс
- 10 сообщений: latency = 500 мс (GC pause, burst на диске)

Средняя latency = (990×2 + 10×500) / 1000 = **6.98 мс** — выглядит неплохо.

Но для 10 пользователей из 1000 задержка — полсекунды. Это 1% запросов с ужасным опытом. Именно поэтому измеряют процентили:

| Процентиль | Значение | Что означает |
|------------|----------|--------------|
| p50 (медиана) | 2 мс | Половина запросов быстрее этого |
| p95 | 5 мс | 95% запросов быстрее, 5% — медленнее |
| p99 | 15 мс | Только 1% медленнее этого порога |
| p99.9 | 50 мс | Один из тысячи запросов может быть таким медленным |
| p99.99 | 200 мс | Один из десяти тысяч |

**SLA обычно формулируется через процентили:** «p99 E2E latency < 20 мс при нагрузке до 100 MB/s».

**Типичные цифры из бенчмарка Confluent (2023):**
- При нагрузке 200 MB/s (200K msg/s по 1 KB): p99 = **5 мс**
- При максимальной нагрузке 605 MB/s: p99 растёт до **нескольких десятков мс** (начинает упираться в диск)

### 2.3 Resource Utilization (использование ресурсов)

Без этих метрик throughput и latency не имеют смысла — ты не знаешь, **почему** система ведёт себя так, а не иначе.

| Ресурс | Ключевые метрики | Что смотреть |
|--------|-----------------|--------------|
| **CPU** | `os.cpu.user`, `os.cpu.system`, `os.cpu.iowait` | iowait > 10% → диск не справляется |
| **Память** | JVM heap usage, page cache dirty pages | Heap > 80% → риск GC pause; мало page cache → reads идут с диска |
| **Диск** | `disk.read.bytes`, `disk.write.bytes`, `disk.io.utilization`, `disk.await` | utilization > 80% → диск — бутылочное горлышко |
| **Сеть** | `network.bytes.in`, `network.bytes.out` | Сравнивай с лимитами инстанса (облака часто режут bandwidth на маленьких VM) |
| **JVM GC** | GC pause time, GC frequency | Любой GC pause > 50 мс → latency spike |

**Диск — главное бутылочное горлышко Kafka.** Почти всегда, когда Kafka «медленная», проблема в дисковой подсистеме. CPU и сеть редко становятся узким местом раньше диска.

### 2.4 Kafka-специфичные метрики (JMX)

Kafka выставляет сотни JMX-метрик. Вот самые важные для performance-анализа:

| JMX метрика | Что означает | Red flag |
|-------------|-------------|----------|
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | Скорость записи на брокер | — |
| `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec` | Скорость чтения с брокера | — |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Партиции с отстающими репликами | > 0 — проблема репликации |
| `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce` | Полное время обработки Produce-запроса | Рост без изменения нагрузки = деградация |
| `kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request=Produce` | Время в очереди запросов | > 10 мс — брокер не справляется |
| `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent` | Простой сетевых потоков | < 0.2 → сетевые потоки перегружены |
| `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent` | Простой I/O потоков | < 0.2 → I/O потоки перегружены |
| `kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs` | Частота и длительность flush-а лога | Рост времени flush = медленный диск |
| `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` | Частота выпадения реплик из ISR | > 0 в steady state → проблемы с брокерами/сетью |

---

## 3. Типичные цифры производительности Kafka

Никакая теория не заменит конкретных чисел. Вот проверенные цифры из трёх ключевых источников.

### 3.1 LinkedIn-бенчмарк 2014: 2 миллиона записей/с на трёх дешёвых машинах

Это самый цитируемый бенчмарк в истории Kafka. Jay Kreps (создатель Kafka) показал:

**Конфигурация:**
- 3 брокера, каждый: 6-ядерный Xeon 2.5 GHz, 32 GB RAM, 8×7200 rpm SATA в RAID-10
- 1 producer, 1 consumer
- 1 топик, 6 партиций (по 2 на брокер)
- Размер сообщения: 100 байт
- `acks=1` (лидер подтверждает без ожидания реплик)
- Consumer читает с конца топика (fresh data, из page cache)

**Результаты:**
- **Producer throughput: 2 024 032 записей/с** (~193 MB/s с учётом batch-заголовков)
- **Consumer throughput: 2 167 330 записей/с** (~207 MB/s)
- Это ~674K записей/с на брокер

**Важно:** Это бенчмарк 11-летней давности на SATA-дисках. Современные NVMe-диски в 10–20 раз быстрее. Он показывает не абсолютный предел Kafka, а её архитектурную эффективность.

### 3.2 Confluent-бенчмарк 2023: 605 MB/s + p99 = 5 мс

Современный бенчмарк на облачном железе:

**Конфигурация:**
- 3 брокера AWS i3en.2xlarge (8 vCPU, 64 GB RAM, 2×2.5 TB NVMe SSD)
- 4 worker-инстанса для нагрузки
- 100 партиций в одном топике
- Размер сообщения: 1 KB
- `acks=all`, `min.insync.replicas=2` (репликация на ≥2 брокера)
- Producer: `batch.size=1MB`, `linger.ms=10` (throughput-режим)
- 4 producer-а, 4 consumer-а

**Результаты:**

| Метрика | Throughput-режим | Latency-режим |
|---------|-----------------|---------------|
| Нагрузка | максимальная | 200 MB/s |
| Peak stable throughput | **605 MB/s** | — |
| p99 E2E latency | десятки мс | **5 мс** |
| Disk utilization | ~100% (насыщение) | ниже предела одного диска (~65%) |
| Producer config | `linger.ms=10` | `linger.ms=1` |

**Ключевой вывод:** Kafka упирается в диск раньше, чем во что-либо ещё. 605 MB/s — это физический предел двух NVMe-дисков в инстансе i3en.2xlarge (каждый даёт ~327 MB/s). CPU и сеть не были узким местом.

### 3.3 Google Cloud Managed Service for Kafka (2025)

Бенчмарк от Google показал scaling-характеристики managed-сервиса:

- **Линейное масштабирование** write throughput при добавлении брокеров
- Для sustained нагрузки рекомендуется держать **disk utilization < 70%** чтобы оставить запас для пиков и ребалансировки
- E2E latency в managed-среде включает latency облачного storage — на 20–30% выше, чем на локальных дисках, но стабильнее

### 3.4 Сводная таблица типичных цифр

| Сценарий | Per Broker Write | Per Broker Read | E2E Latency (p99) | Hardware |
|----------|-----------------|-----------------|---------------------|----------|
| Low-end (dev) | 50–100 MB/s | 100–200 MB/s | 10–50 мс | SATA SSD, 4 cores, 16 GB |
| Mid-range (prod) | 150–300 MB/s | 300–600 MB/s | 5–20 мс | NVMe SSD, 8 cores, 32 GB |
| High-end (extreme) | 300–600 MB/s | 600+ MB/s | 2–10 мс | Multi-NVMe, 16+ cores, 64+ GB |
| LinkedIn-scale (2014) | 65 MB/s | 69 MB/s | не измерялась | 8×SATA RAID-10 |
| Confluent-bench (2023) | 200 MB/s | ~400 MB/s | 5 мс (p99) | AWS i3en.2xlarge |

**Правило большого пальца:**
- **Write throughput брокера ≈ суммарная скорость последовательной записи дисков / replication factor**
- **Read throughput брокера ≈ скорость чтения из page cache** (если consumer-ы читают свежие данные) или скорость дисков (если читают старые данные)
- **Сеть нужна с запасом 2×** от ожидаемого throughput: Kafka генерирует много cross-DC replication и consumer read-трафика

---

## 4. Факторы, влияющие на производительность

Производительность Kafka — это функция от многих переменных. Вот главные:

### 4.1 Размер сообщения (record size)

**Чем меньше сообщения, тем меньше throughput в MB/s при том же количестве msg/s.** Оверхед на заголовки и batch-метаданные делает мелкие сообщения неэффективными:

| Размер сообщения | Throughput (MB/s) | Сообщений/с | Эффективность |
|-----------------|-------------------|-------------|---------------|
| 100 байт | ~50 MB/s | 500 000 | Низкая: много оверхеда |
| 1 KB | ~200 MB/s | 200 000 | Хорошая |
| 10 KB | ~400 MB/s | 40 000 | Отличная |
| 100 KB | ~500 MB/s | 5 000 | Отличная |

**Почему так:** Каждое сообщение несёт оверхед: batch-заголовок (~61 байт), record-заголовок (key, value, headers, timestamp), offset-метаданные. Для 100-байтовых сообщений оверхед может составлять 40–50%. Для 10 KB сообщений — менее 1%.

### 4.2 Количество партиций

Больше партиций = больше параллелизма, но и больше оверхеда:
- Каждая партиция — это набор файлов (`.log`, `.index`, `.timeindex`)
- Каждая партиция требует file handles (лимит ОС: `ulimit -n`)
- Каждая партиция требует map areas (лимит: `vm.max_map_count`)
- Recovery после сбоя пропорционален количеству партиций
- Больше партиций = больше consumer rebalance-ов

**Эмпирическое правило:** 6–12 партиций на брокер на топик — хороший старт. Суммарно на брокере не рекомендуется превышать **4000–6000 партиций** без тщательного тестирования.

### 4.3 Replication factor и acks

| Конфигурация | Throughput | Latency | Durability |
|-------------|-----------|---------|------------|
| `acks=0` | Максимальный | Минимальная | Никакой (fire-and-forget) |
| `acks=1` | Высокий | Низкая | Потеря при сбое лидера |
| `acks=all`, `min.insync.replicas=2` | Средний | Средняя | Production-grade |
| `acks=all`, `min.insync.replicas=3` (RF=3) | Ниже среднего | Выше средней | Максимальная |

**На практике:** `acks=all` с `min.insync.replicas=2` при RF=3 — золотой стандарт. Ты получаешь durability без катастрофического падения throughput.

### 4.4 Сжатие (compression)

Kafka поддерживает: `gzip`, `snappy`, `lz4`, `zstd`.

| Кодек | Коэффициент сжатия | CPU overhead | Типичное применение |
|-------|-------------------|--------------|---------------------|
| `none` | 1.0× | 0 | Уже сжатые данные (изображения, видео) |
| `snappy` | 1.5–2× | Низкий | Баланс speed/размер |
| `lz4` | 1.5–2.5× | Низкий-средний | Лучше snappy по speed |
| `gzip` | 2–4× | Высокий | Максимальное сжатие, не latency-sensitive |
| `zstd` | 2–4× | Средний | Современная альтернатива gzip, лучше баланс |

**Важно:** Сжатие на producer-е (а не на брокере) — это стандарт. Брокер хранит сжатые batch-и как есть, не перепаковывая. Consumer получает сжатые данные и распаковывает сам.

### 4.5 Batching (пакетирование)

Это главный рычаг производительности в Kafka:

| Параметр | Throughput-режим | Latency-режим |
|----------|-----------------|---------------|
| `batch.size` | 512 KB — 1 MB | 16–64 KB |
| `linger.ms` | 5–100 мс | 0–5 мс |
| Результат | Максимум MB/s | Минимум задержки |

**Как работает batching:** Producer накапливает записи в буфер (`batch.size`), потом отправляет пачку брокеру либо когда буфер заполнен, либо когда прошло `linger.ms`. На брокере batch пишется на диск одной операцией — это даёт sequential I/O, который в 100× быстрее random I/O.

---

## 5. Методология бенчмаркинга Kafka

Бенчмаркинг без методологии — это гадание. Вот пошаговый фреймворк.

### 5.1 Фаза 0: Определи цели

До запуска любого теста ответь на вопросы:
- **Что измеряем?** Peak write throughput? E2E latency при фиксированной нагрузке? Влияние количества партиций на latency?
- **Какой workload?** Размер сообщений, ключей, процент уникальных ключей, паттерн нагрузки (равномерный / bursty)?
- **Какой SLA?** Например: «Устойчивая запись 100 MB/s при p99 E2E < 20 мс»
- **Что будем менять?** Конфигурацию producer? Параметры брокера? Количество партиций? Версию Kafka?

**Антипаттерн:** Запускать тест «просто чтобы посмотреть, сколько выжмет Kafka». Без гипотезы ты не интерпретируешь результаты.

### 5.2 Фаза 1: Подготовка окружения

**Изоляция:**
- **Никогда не бенчмаркай на production-кластере.** Даже «лёгкий» тест может повлиять на реальный трафик.
- Создай выделенное окружение, максимально идентичное production: те же типы дисков, та же сеть, те же версии ПО.
- Убедись, что тестовые клиенты (producer/consumer) **не являются бутылочным горлышком** — выдели достаточно CPU и памяти.

**OS-tuning (из официальной документации Kafka):**
```bash
# Лимит файловых дескрипторов
ulimit -n 100000

# Размер socket buffer (для cross-DC репликации)
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728

# Максимальное количество memory map areas (если много партиций)
sysctl -w vm.max_map_count=262144

# Монтирование дисков с noatime
mount -o noatime /dev/nvme0n1 /var/lib/kafka/data
```

**Выбор файловой системы:** И XFS, и EXT4 работают хорошо. Confluent рекомендует **XFS** — их тесты показали лучшую производительность (160 мс vs 250 мс+ Request Local Time по сравнению с лучшей конфигурацией EXT4).

### 5.3 Фаза 2: Прогрев (warm-up)

Kafka (и Linux page cache) требуют прогрева для показа реальной производительности:

- **Первые минуты теста — мусор.** Page cache пуст, JVM не разогрета, JIT-компилятор не отработал.
- **Длительность прогрева:** 5–15 минут под нагрузкой, близкой к тестовой.
- **Отбрасывай данные прогрева** из финальных результатов.

> В бенчмарке Google Cloud Managed Service for Kafka (2025) авторы делали warm-up 30 минут перед измерениями.

### 5.4 Фаза 3: Проведение тестов

**Золотые правила:**
1. **Меняй одну переменную за раз.** Если ты поменял `linger.ms`, `batch.size` и `compression.type` одновременно — ты не узнаешь, что именно дало эффект.
2. **Повторяй тесты 3–5 раз.** Одиночный прогон может попасть на GC pause или сетевой всплеск. Результат должен быть воспроизводимым.
3. **Гони тест достаточно долго.** 5-минутный тест показывает «пиковую» производительность, а не устойчивую. Для production-прогнозов нужно 30–60 минут.
4. **Измеряй steady state.** После начального переходного периода система должна стабилизироваться. Если throughput продолжает падать или latency расти — ты превысил устойчивый предел.

**Сбор метрик:**
```
Клиенты (producer/consumer):
├── Throughput (msg/s, MB/s)
├── Latency (p50, p95, p99, p99.9, max)
├── Error rate
└── Client-side CPU/Memory

Брокеры (Kafka JMX):
├── BytesInPerSec, BytesOutPerSec
├── TotalTimeMs, RequestQueueTimeMs, ResponseSendTimeMs
├── NetworkProcessorAvgIdlePercent
├── RequestHandlerAvgIdlePercent
├── UnderReplicatedPartitions, IsrShrinksPerSec
└── LogFlushRateAndTimeMs

OS (broker nodes):
├── CPU: user, system, iowait
├── Disk: utilization, await, read/write throughput
├── Memory: page cache dirty/clean, swap usage
└── Network: bytes in/out, packet errors
```

### 5.5 Фаза 4: Анализ и интерпретация

После получения цифр ищи бутылочное горлышко:

| Симптом | Вероятная причина | Что проверять |
|---------|------------------|---------------|
| Producer latency высокий, брокер CPU/disk низкий | Проблема на стороне producer | Batching, serialization, client-side resources |
| Брокер CPU на 100% | Broker processing bottleneck | SSL, compression, too many partitions, network threads |
| Disk iowait > 20% | Диск не справляется | Слишком высокий fsync, медленные диски, конкуренция за диск с другими процессами |
| Network throughput плоский несмотря на рост нагрузки | Сеть — узкое место | Проверь bandwidth limits облачного инстанса |
| E2E latency высокий, producer/broker latency низкий | Consumer-проблема | Медленная обработка в consumer, rebalance, slow network to consumer |
| UnderReplicatedPartitions > 0 | Проблема репликации | Перегруженный брокер, network partition, медленный follower |

### 5.6 Фаза 5: Итерация

Бенчмаркинг — это цикл, а не разовое мероприятие:

```
Определить baseline (default config)
        ↓
Изменить ОДИН параметр
        ↓
Прогнать тест → Собрать метрики
        ↓
Сравнить с baseline
        ↓
Достигнут SLA? ──No──→ Повторить с новым параметром
        ↓ Yes
Задокументировать конфигурацию
        ↓
Периодически перепроверять (после апгрейдов, изменения нагрузки)
```

---

## 6. Инструменты для бенчмаркинга

### 6.1 Встроенные утилиты Kafka

Самый простой способ начать:

```bash
# Producer performance test: пишет 10M записей по 1 KB с throttling 50K msg/s
bin/kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 10000000 \
  --record-size 1024 \
  --throughput 50000 \
  --producer-props \
    bootstrap.servers=broker1:9092,broker2:9092 \
    acks=all \
    linger.ms=10 \
    compression.type=lz4

# Consumer performance test
bin/kafka-consumer-perf-test.sh \
  --topic perf-test \
  --broker-list broker1:9092,broker2:9092 \
  --messages 10000000 \
  --group perf-test-group \
  --show-detailed-stats
```

**Плюсы:** Идут в комплекте, простые в использовании.  
**Минусы:** Нет гистограмм latency по процентилям, нет multi-topic сценариев, нет координации producer+consumer.

### 6.2 OpenMessaging Benchmark Framework (OMBF)

Стандартизированный фреймворк для бенчмаркинга messaging-систем. Используется Confluent для публикуемых бенчмарков.

**Возможности:**
- Деплой через Kubernetes или bare-metal
- Конфигурируемые workload-ы (размер сообщения, паттерн ключей, throttling)
- Автоматический сбор метрик в Prometheus
- Сравнение разных систем (Kafka, Pulsar, RabbitMQ) на одном workload-е

**Запуск:**
```bash
git clone https://github.com/confluentinc/openmessaging-benchmark
cd openmessaging-benchmark
# Конфигурируешь workload в YAML, запускаешь через provided scripts
```

### 6.3 Trogdor

Фреймворк тестирования, разработанный командой Kafka для internal system testing:

- Запуск агентов на каждом брокере
- Создание task-ов: produce records, consume records, fault injection
- Поддержка coordinated failure testing
- Сложнее в настройке, но мощнее для комплексных сценариев

### 6.4 Кастомные клиенты

Для специфических сценариев (special key distribution, non-standard message sizes, custom serialization) пишут кастомные producer/consumer на Java, Python, Go:

```java
// Пример кастомного latency-aware producer на Java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092");
props.put("acks", "all");
props.put("linger.ms", 5);
props.put("batch.size", 65536);

try (KafkaProducer<String, byte[]> producer = new KafkaProducer<>(props)) {
    for (int i = 0; i < TOTAL_MESSAGES; i++) {
        long sendStart = System.nanoTime();
        producer.send(new ProducerRecord<>("perf-topic", generatePayload(1024)),
            (metadata, exception) -> {
                long e2eStart = /* stored from send */;
                long latency = System.nanoTime() - e2eStart;
                latencyHistogram.record(latency); // HdrHistogram или t-digest
            });
    }
}
```

**Рекомендация для продакшена:** Используй HdrHistogram или t-digest для latency-гистограмм — они дают точные процентили без хранения всех значений.

---

## 7. Типичные ошибки бенчмаркинга (и как их избежать)

### Ошибка 1: Измерять только среднюю latency

**Пример из жизни:** Средняя E2E latency = 3 мс. Внешне система быстрая. Но p99.9 = 2 секунды. Причина: GC pause в 2 секунды раз в минуту. Среднее этого не показывает.

**Исправление:** Всегда собирай p50, p95, p99, p99.9. HdrHistogram или t-digest — стандарт.

### Ошибка 2: Не делать warm-up

**Симптом:** Первый тест показывает 100 MB/s, второй (на той же конфигурации) — 350 MB/s. Разница — page cache разогрелся.

**Исправление:** 15 минут прогрева, данные прогрева — в мусор.

### Ошибка 3: Тестировать с нереалистичными данными

**Пример:** Ты тестируешь сжатие gzip на сообщениях из одних нулей. Gzip сожмёт их в 1000 раз. В production — сжатие 2×. Ты завысил ожидания.

**Исправление:** Генерируй реалистичные данные. Если production-данные — JSON, используй JSON-подобный payload.

### Ошибка 4: Клиент — бутылочное горлышко

**Симптом:** Брокер загружен на 30%, но throughput не растёт при добавлении producer-ов. А клиентская машина загружена на 100% CPU.

**Исправление:** Мониторь клиентские ресурсы так же тщательно, как и брокеры. При необходимости разноси producer/consumer на отдельные машины.

### Ошибка 5: Короткие тесты

**Симптом:** 2-минутный тест показывает 500 MB/s. 30-минутный тест — 250 MB/s. Page cache заполнился, брокер начал ждать fsync.

**Исправление:** Минимум 15 минут теста, лучше 30–60 для production-прогнозов.

### Ошибка 6: Менять несколько переменных одновременно

**Симптом:** Ты поменял `acks` с 1 на all, `linger.ms` с 0 на 10 и `compression.type` с none на lz4. Throughput вырос на 20%. Что дало эффект? Batching? Сжатие? Неизвестно.

**Исправление:** Одна переменная — один тест. Это скучно, но это единственный способ понять систему.

---

## 8. Связь с другими статьями цикла

Эта статья дала тебе метрический фундамент. Дальше по треку:

- **[02-tuning.md](../06-performance/02-tuning.md)** — как конкретно крутить параметры брокера, producer и consumer для достижения throughput/latency целей. Практические рецепты для throughput-режима и latency-режима.
- **[03-scalability.md](../06-performance/03-scalability.md)** — горизонтальное масштабирование Kafka: добавление брокеров, увеличение партиций, rebalance, cross-DC replication.
- **[04-bottlenecks.md](../06-performance/04-bottlenecks.md)** — диагностика реальных production-проблем: профилирование, flame graphs, case studies инцидентов.

А также:
- **[../05-development-api/04-testing.md](../05-development-api/04-testing.md)** — тестирование Kafka-приложений (integration testing, Testcontainers, Trogdor), которое дополняет бенчмаркинг.
- **[../07-operations/01-monitoring.md](../07-operations/01-monitoring.md)** — production-мониторинг метрик, которые мы здесь определили, через Prometheus + Grafana.

---

## Источники

1. **Confluent Developer — Apache Kafka Performance (2023).** Официальный бенчмарк: throughput 605 MB/s, p99 = 5 мс. URL: https://developer.confluent.io/learn/kafka-performance/
2. **Confluent — Kafka Performance Testing: Best Practices, Tools, and Metrics.** Гайд по тестированию производительности. URL: https://confluent.io/learn/kafka-performance-testing
3. **LinkedIn Engineering (Jay Kreps, 2014).** «Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)». Классический бенчмарк. URL: https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines
4. **Apache Kafka Documentation — Hardware and OS (v3.9).** Официальные рекомендации по железу, XFS vs EXT4, OS-тюнингу. URL: https://kafka.apache.org/39/operations/hardware-and-os/
5. **ActiveWizards — Kafka Benchmarking: Methodologies & Tools for Performance.** Методология бенчмаркинга, сравнение инструментов. URL: https://activewizards.com/blog/kafka-benchmarking-methodologies-and-tools-for-performance
6. **AutoMQ — Kafka Latency: Optimization & Benchmark & Best Practices (2024).** Компоненты задержки, бенчмарк-инструменты, конфигурация. URL: https://automq.com/blog/kafka-latency-optimization-strategies-best-practices
7. **Google Cloud Blog (2025).** «Managed Service for Kafka benchmarking and scaling guidance». Бенчмаркинг managed Kafka. URL: https://cloud.google.com/blog/products/data-analytics/managed-service-for-kafka-benchmarking-and-scaling-guidance/
8. **Habr — Оптимизация настроек Kafka кластера. Часть 3. Сравнительное тестирование, мониторинг и тонкая настройка (2024).** Перевод руководства Confluent: метрики, бенчмарк-методология. URL: https://habr.com/ru/articles/819677/
9. **arXiv (Mohammad, 2025).** «Analysis of Design Patterns and Benchmark Practices in Apache Kafka Event‑Streaming Systems». Академический обзор: TPCx-Kafka, Yahoo Streaming Benchmark, reproducibility issues. URL: https://arxiv.org/html/2512.16146v1
10. **DataFlow Academy — Kafka Sizing & Scaling: Hardware Requirements and Scaling Strategies.** Формулы расчёта размера кластера. URL: https://dataflow.academy/en/knowledge-hub/kafka-sizing-scaling
11. **Instaclustr — How to size Apache Kafka clusters for Tiered Storage (2024).** Модель производительности для SSDs, network, I/O. URL: https://instaclustr.com/blog/how-to-size-apache-kafka-clusters-for-tiered-storage-part-1
12. **Azure Blog — Processing trillions of events per day with Apache Kafka on Azure (2018).** Production-кейс масштаба. URL: https://azure.microsoft.com/en-us/blog/processing-trillions-of-events-per-day-with-apache-kafka-on-azure/
13. **Confluent — OpenMessaging Benchmark Framework.** Исходный код референсного бенчмарк-фреймворка. URL: https://github.com/confluentinc/openmessaging-benchmark

---

*Статья написана на основе официальной документации Kafka, бенчмарков Confluent/LinkedIn/Google Cloud, профильных блогов (ActiveWizards, AutoMQ, DataFlow Academy) и Habr-переводов. Цифры актуальны на 2024–2025 гг. Для абсолютной точности — всегда проводи собственные бенчмарки на своём железе со своим workload-ом.*
