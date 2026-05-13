# Бенчмарки и Сравнение Производительности: Kafka против Аналогов

**Нижняя строка:** Независимые бенчмарки 2024–2026 годов показывают, что Kafka сохраняет лидерство по пропускной способности на одном брокере (605 MB/s пиковый throughput, 200+ MB/s stable) и зрелости экосистемы, однако Redpanda выигрывает по latency на малых нагрузках, Pulsar — по изоляции многопользовательских окружений, а RabbitMQ Streams приближается к Kafka-классу throughput с существенно меньшим потреблением CPU. Главный вывод: **цифры вендорских бенчмарков требуют обязательной верификации** — методология и конфигурация тестов определяют результат сильнее, чем архитектура брокера.

---

## 1. Почему бенчмарки Kafka — это минное поле

Производительность систем потоковой обработки варьируется в зависимости от десятков переменных. Два бенчмарка на одинаковом железе могут показать противоположные результаты, если различается хотя бы одна из них:

| Переменная | Влияние | Пример разброса |
|-----------|---------|----------------|
| Размер сообщения | Маленькие сообщения (100 B) — overhead протокола; большие (1 MB) — упираются в диск | 100 B: 200K msg/s → 1 KB: 80K msg/s (тот же MB/s!) |
| Количество партиций | Больше партиций = больше параллелизма, но больше метаданных | 1 партиция: 50K → 100 партиций: 500K msg/s |
| Режим подтверждения (acks) | acks=0: throughput max; acks=all: durability max | Разница до 3-10× между acks=0 и acks=all |
| Коэффициент репликации | RF=1 vs RF=3: дополнительный сетевой трафик + дисковая запись | RF=3 съедает ~30-40% throughput против RF=1 |
| Сжатие (compression) | snappy/lz4/gzip/zstd снижают нагрузку на диск и сеть | Сжатие может удвоить эффективный throughput |
| Файловая система | XFS vs ext4, опции монтирования (noatime) | Разница 10-20% на запись |
| Версия Kafka и JDK | Kafka 4.x (KRaft) vs 3.x (ZK), JDK 21 vs JDK 17 | До 15% разницы на latency |
| Размер page cache и RAM | Если данные помещаются в page cache — скорость memory; иначе — disk | Падение throughput в 2-5× при уходе в disk I/O |
| Сетевой лимит инстанса | AWS i3en.2xlarge = 25 Gbps; t3.medium = 5 Gbps | 655 MB/s vs ~250 MB/s потолок |

**Аналогия:** Бенчмаркинг Kafka похож на сравнение спорткаров на разных трассах — один тест на автобане, другой на горном серпантине. Цифры несопоставимы, пока не выровнены условия.

**Главные ловушки вендорских бенчмарков:**

1. **Разное железо.** Один брокер тестируется на AWS i3en.6xlarge (24 vCPU, 192 GB RAM), другой — на c5.4xlarge (16 vCPU, 32 GB). Результаты несопоставимы.
2. **Разные режимы durability.** «Kafka быстрее Pulsar» на acks=0 — это не вывод для финансового приложения, где нужен acks=all.
3. **Сжатие включено у одного, выключено у другого.** Разница в эффективном throughput может быть двукратной.
4. **Default-конфигурация vs тюнинг.** «Из коробки» Kafka работает на 30-50% медленнее, чем после базового тюнинга (размер батча, linger.ms, количество потоков).
5. **Разные версии клиентов.** Старый клиент на blocking I/O vs новый на async — разница в throughput может быть 5-10x.

---

## 2. Методология честного бенчмаркинга

### 2.1 Стандарт OpenMessaging Benchmark (OMB)

Индустриальным стандартом для сравнения месседжинг-систем стал **OpenMessaging Benchmark Framework** — открытый инструмент, поддерживаемый совместно Confluent, StreamNative (Pulsar), Redpanda и другими вендорами. OMB решает главную проблему: обеспечивает воспроизводимые, apples-to-apples сравнения на идентичном железе.

**Архитектура OMB:**

```
┌─────────────┐     ┌──────────────┐     ┌───────────────────┐
│  Benchmark  │────▶│   Workers     │────▶│  Kafka / Pulsar /  │
│     CLI     │     │  (producers + │     │  Redpanda / etc.   │
│  (driver)   │◀────│   consumers)  │◀────│                   │
└─────────────┘     └──────────────┘     └───────────────────┘
                            │
                    ┌───────▼───────┐
                    │   Prometheus  │
                    │  + Grafana    │
                    └───────────────┘
```

**Два режима работы:**
- **Distributed mode** — workers на отдельных инстансах, максимально приближено к production
- **Local mode** — единый процесс, подходит для быстрой проверки гипотез

**Стандартная процедура тестирования через OMB:**
1. Определить **Driver** (параметры подключения: bootstrap-серверы, TLS/SASL, replication factor)
2. Определить **Workload** (количество топиков/партиций, producer/consumer, размер сообщений, длительность)
3. Запустить бенчмарк с **warmup-периодом** (минимум 60 секунд) — JIT-компиляция и прогрев page cache
4. Собрать метрики через Prometheus (брокерные + клиентские)
5. Повторить минимум **3 раза**, взять медиану

### 2.2 Минимальные требования к честному сравнению

| Требование | Почему важно |
|-----------|-------------|
| Идентичное железо | Исключает hardware bias |
| Единый инструмент измерения | Разные клиенты = разные bottleneck'и |
| Warmup ≥ 60 с | JVM JIT + page cache прогрев |
| Тест ≥ 10 минут | Исключает transient-эффекты |
| Изоляция (нет соседних нагрузок) | Cloud — noisy neighbor, on-prem — фоновые процессы |
| Фиксация p50/p95/p99/p99.9 | Средняя latency бесполезна без хвостов |
| OS-тюнинг (tuned-adm, I/O scheduler) | Стандартизирует низкоуровневое поведение |
| Отключение swap | Swap убивает latency предсказуемость |
| Контроль версий (Kafka + JDK) | Разные версии = разные оптимизации |

---

## 3. Ключевые бенчмарки (2023–2026)

### 3.1 Confluent: Kafka на AWS i3en.2xlarge (2023–2024)

**Самый цитируемый открытый бенчмарк.** Confluent опубликовал полную методологию, код (open source) и результаты.

**Конфигурация:**
- 3 брокера на AWS i3en.2xlarge (8 vCPU, 64 GB RAM, 2×2.5 TB NVMe)
- 3 ZK-ноды (переход на KRaft в поздних тестах)
- 4 worker-инстанса (c5n.9xlarge) для producer/consumer
- 100 партиций, 3× репликация, acks=all
- OS-тюнинг: `tuned-adm latency-performance`, deadline I/O scheduler

**Результаты:**

| Метрика | Значение | Контекст |
|---------|----------|----------|
| Пиковый stable throughput | **605 MB/s** (на 3 брокера) | ~200 MB/s на брокер |
| p99 latency при 200 MB/s | **5 мс** | Включая репликацию |
| Disk utilization | ~92% от максимума (655 MB/s) | Kafka близка к hardware ceiling |
| CPU utilization | ~20% на брокер | I/O-bound, не CPU-bound |

**Вывод:** Kafka на современном NVMe-железе упирается в пропускную способность дисков, а не в CPU или сеть. Дополнительный тюнинг (увеличение буферов, количества I/O-потоков) может дать ещё 10-15%.

---

### 3.2 Kafka vs Redpanda: независимый бенчмарк (апрель 2026)

**Самый свежий apples-to-apples тест** — Kafka 4.2.0 (KRaft) против Redpanda 26.1.2 на идентичных Proxmox VM (4 vCPU, 8 GB RAM, NVMe).

**Producer Throughput (1M сообщений):**

| Тест | Kafka rec/s | Kafka avg lat | Redpanda rec/s | Redpanda avg lat |
|------|-------------|---------------|----------------|------------------|
| 100B, 1 partition, acks=1 | 203 915 | 89 мс | **296 559** | 33 мс |
| 1KB, 1 partition, acks=1 | 81 426 | 50 мс | **100 593** | 91 мс |
| 100B, 6 partition, acks=1 | **479 386** | 17 мс | 291 205 | 34 мс |
| 1KB, 6 partition, acks=1 | **165 782** | 57 мс | 61 839 | 460 мс |
| 100B, 1 partition, acks=all | **326 797** | 573 мс | 39 577 | 6 275 мс |

**Sustained Throughput (5M сообщений, 1KB, 6 partition, acks=1):**

| Метрика | Kafka 4.2 | Redpanda 26.1 |
|---------|----------|---------------|
| Throughput | **211 282 rec/s** (206 MB/s) | 89 455 rec/s (87 MB/s) |
| Avg latency | **2.53 мс** | 315 мс |
| p99 latency | **41 мс** | 514 мс |

**End-to-End Latency (10K сообщений):**

| Тест | Kafka avg | Kafka p99 | Redpanda avg | Redpanda p99 |
|------|----------|-----------|-------------|-------------|
| 100B, acks=1 | 1.05 мс | 4 мс | **0.79 мс** | 3 мс |
| 1KB, acks=1 | 0.88 мс | 3 мс | **0.83 мс** | 3 мс |
| 100B, acks=all | **0.92 мс** | **3 мс** | 2.09 мс | 6 мс |

**Интерпретация:**
- **Redpanda выигрывает на малых нагрузках:** однопартиционные тесты с acks=1 показывают преимущество C++/Seastar thread-per-core архитектуры на 45% по throughput и в 2-3× по latency
- **Kafka доминирует на высоком параллелизме:** 6 партиций — 2.4× sustained throughput (211K vs 89K rec/s) и в 7-12× ниже latency
- **acks=all — катастрофа для Redpanda:** на одном узле community edition Raft fsync даёт 6+ секунд задержки, тогда как Kafka KRaft держит 573 мс. Причина: Redpanda community edition делает синхронный fsync на каждый запрос при acks=all на одном узле
- **Sustained load:** Kafka удерживает стабильный throughput в 2.4× выше и latency в 100× ниже под продолжительной нагрузкой

> **⚠️ Важно:** Это бенчмарк на 1 брокере. Redpanda позиционируется для multi-broker production-кластеров. Результаты не стоит экстраполировать на production без собственного тестирования.

---

### 3.3 Kafka vs Pulsar: бенчмарки устойчивости и изоляции

Поскольку и Kafka, и Pulsar — distribution-first системы, бенчмарки между ними фокусируются не на «raw throughput», а на поведении под смешанными нагрузками.

**Ключевые наблюдения из независимых тестов (2023–2025):**

| Аспект | Kafka | Pulsar |
|--------|-------|--------|
| Raw throughput (одинаковое железо) | Сопоставимо (5-10% разброс) | Сопоставимо |
| Изоляция tenant'ов | Общий кластер, ACL/quotas | Нативная multi-tenancy (тенанты, namespace'ы) |
| Влияние «шумного соседа» | Партиции конкурируют за I/O | Сегментированное хранилище (bookie), меньше interference |
| Поведение при отказе брокера | Выборы лидера (секунды) | Бесшовное переключение bookie (миллисекунды) |
| Производительность при geo-replication | MirrorMaker 2 (async) | Встроенная geo-replication (лучше на 15-20% в тестах) |

**Академический бенчмарк (University of Helsinki, 2024):**

Магистерская диссертация, сравнивающая Kafka и Pulsar через systematic literature review + controlled experiments, показала:

- По **throughput** системы сопоставимы (±5-10% в зависимости от конфигурации)
- По **latency** Pulsar показал на 15-25% лучшие результаты на низких процентилях (p50), но на p99.9 разрыв сокращается
- По **resource utilization** Pulsar потребляет на 20-30% больше CPU из-за разделения брокеров и bookie, но обеспечивает лучшую изоляцию
- По **fault tolerance** Pulsar восстанавливается быстрее (сегментированное хранилище vs единый commit log)

**StreamNative OpenMessaging бенчмарк (2024):**

StreamNative (коммерческий вендор Pulsar) провёл собственный OMB-бенчмарк с «исправленной методологией» против тестов Confluent:

- Ключевое исправление: выравнивание уровней durability (в исходном тесте Confluent Kafka использовала `acks=all` с репликацией, а Pulsar — с меньшими гарантиями)
- При одинаковых гарантиях durability разрыв сократился до **5-8%** в пользу Kafka на throughput
- Pulsar показал лучшие результаты в тестах с большим количеством consumer-групп (50+) — преимущество сегментированного хранения

**Вывод:** Для 95% use-case'ов Kafka и Pulsar сопоставимы по raw производительности. Выбор сводится к архитектурным предпочтениям: multi-tenancy и geo-replication (Pulsar) vs экосистема и зрелость (Kafka).

---

### 3.4 Kafka vs RabbitMQ Streams: неожиданный конкурент

RabbitMQ 4.x (2024+) представил **Streams** — новую структуру данных с replay, retention и Kafka-классом throughput, встроенную в RabbitMQ. Это стирает главное различие «Kafka для стриминга, RabbitMQ для очередей».

**Бенчмарк Confluent (2024) — Kafka vs RabbitMQ vs Pulsar на одинаковом железе:**

```
Пиковый stable throughput (одинаковые AWS i3en.2xlarge):

Kafka 4.x:     ████████████████████████████ 605 MB/s
Pulsar 3.x:    ██████████████████████████   550 MB/s
RabbitMQ Str:  ██████████████████████       480 MB/s
RabbitMQ Cls:  ██████                        50 MB/s  ← classic queues
```

| Система | Stable Throughput | p99 Latency | CPU Usage |
|---------|-------------------|-------------|-----------|
| Kafka 4.x | **605 MB/s** | 5 мс | 20% |
| Pulsar 3.x | 550 MB/s | 8 мс | 35% |
| RabbitMQ Streams 4.x | 480 MB/s | 12 мс | 40% |
| RabbitMQ Classic Queues | 50 MB/s | <1 мс | 40% |

**Что это значит:** RabbitMQ Streams сократил разрыв с Kafka до 20% по throughput, оставаясь в рамках единого брокера (не нужно разворачивать отдельный Kafka). Для команд, уже использующих RabbitMQ для очередей, Streams — жизнеспособная альтернатива Kafka для 80% стриминг-сценариев.

---

## 4. Реальные цифры vs маркетинговые заявления

### 4.1 Миф: «Kafka обрабатывает миллионы сообщений в секунду»

**Правда:** Знаменитый бенчмарк LinkedIn 2014 года — 2 миллиона записей/секунду на трёх дешёвых машинах — по-прежнему цитируется. Но это **2 миллиона записей по 100 байт**, что составляет жалкие **200 MB/s**. На современном железе Kafka достигает **500-600 MB/s**, что при 100-байтовых сообщениях действительно составляет ~5 миллионов записей/с. Но при размере сообщений 1 KB это уже 500 000 записей/с, а при 10 KB — 50 000 записей/с.

**Мораль:** Всегда переводите «сообщений/с» в MB/s. Это единая валюта производительности.

### 4.2 Миф: «Redpanda в 10× быстрее Kafka»

**Правда:** Маркетинговые 10× относятся к специфическому сценарию: p99 latency на малых сообщениях при низкой нагрузке (одна партиция, acks=1). В тестах апреля 2026 года на sustained нагрузке Kafka оказалась в 2.4× быстрее. Цифры «10×» — реальны только в изолированном микробенчмарке, не репрезентативном для production.

### 4.3 Миф: «RabbitMQ медленный»

**Правда:** Classic queues RabbitMQ действительно на порядок медленнее Kafka на throughput (50K vs 1M msg/s). Но RabbitMQ Streams достигает 480 MB/s — 80% от Kafka-производительности — при этом сохраняя преимущества RabbitMQ: sub-millisecond latency для отдельных сообщений, гибкую маршрутизацию и простоту операций. Медленный — classic queues, а не платформа в целом.

---

## 5. Бенчмарки в облаке: Managed Services

Отдельная категория — сравнение управляемых сервисов, где добавляется фактор цены:

| Сервис | Max Throughput (на кластер) | p99 Latency | Стоимость / GB/мес | SLA |
|--------|---------------------------|-------------|-------------------|-----|
| Confluent Cloud | ~1 GB/s (dedicated) | 5-10 мс | $0.10-0.35/GB | 99.95% |
| Amazon MSK | ~800 MB/s | 5-15 мс | $0.10-0.20/GB | 99.9% |
| Redpanda Cloud | ~600 MB/s | 3-8 мс | $0.08-0.15/GB | 99.95% |
| WarpStream | ~400 MB/s | 50-200 мс | **$0.02-0.05/GB** | 99.9% |
| Self-hosted Kafka | ~600 MB/s (на 3 брокера) | 3-10 мс | Только инфраструктура | Зависит от вас |

**DoubleCloud бенчмарк performance-per-price (2024):**

DoubleCloud провёл сравнение производительности на доллар для различных конфигураций:

- Self-hosted Kafka на reserved instances выигрывает по TCO при sustained нагрузке >500 MB/s
- Confluent Cloud оптимален для нагрузок 50-300 MB/s с пиковыми всплесками
- WarpStream предлагает лучшую цену за GB, но ценой значительно более высокой latency (50-200 мс vs 5-10 мс)

---

## 6. Сравнительная сводка: Когда какой инструмент показывает лучшие результаты

| Сценарий | Победитель по бенчмаркам | Ключевая метрика |
|----------|------------------------|-----------------|
| Максимальный throughput (>500 MB/s) | **Kafka** | 605 MB/s пиковый |
| Минимальная latency на малых нагрузках | **Redpanda** | 0.79 мс avg (acks=1) |
| Минимальная latency с durability (acks=all) | **Kafka** | 0.92 мс avg |
| Изоляция tenant'ов (multi-tenancy) | **Pulsar** | Меньше interference от «шумных соседей» |
| Geo-replication | **Pulsar** | Встроенная, на 15-20% эффективнее MirrorMaker 2 |
| Единый брокер для очередей + стриминга | **RabbitMQ Streams** | 480 MB/s + classic queues в одном |
| Минимальный TCO на высоких объёмах | **Self-hosted Kafka** | Нет наценки за управление |
| Минимальная стоимость за GB (терпимость к latency) | **WarpStream** | $0.02/GB |
| Экосистема (Connect, Streams, ksqlDB, Debezium) | **Kafka** | 200+ коннекторов, клиенты на 20+ языках |

---

## 7. Как интерпретировать бенчмарки: чеклист

Перед тем как принять решение на основе бенчмарка, проверь:

- [ ] **Железо идентично** у всех участников? (инстансы, диски, сеть)
- [ ] **Версии ПО** зафиксированы? (Kafka 4.2 vs 3.6 — разный мир)
- [ ] **Методология описана** достаточно детально для воспроизведения?
- [ ] **Durability-гарантии выровнены?** (acks=all у всех участников)
- [ ] **Warmup-период** был? (≥60 секунд)
- [ ] **Длительность теста** достаточна? (≥10 минут)
- [ ] **Показаны процентили**, а не только средние?
- [ ] **Sustained throughput**, а не пиковый burst?
- [ ] **Кто автор бенчмарка?** Вендорский тест — не фальшивка, но требует независимой верификации
- [ ] **Соответствует workload моему production-сценарию?** (размер сообщений, количество consumer-групп, паттерн чтения/записи)

---

## 8. Рекомендации

1. **Не верьте одному бенчмарку.** Никогда. Всегда ищите минимум 2 независимых источника.
2. **Запустите собственный тест.** OMB делает это проще, чем кажется — базовый тест на одном инстансе занимает час.
3. **Бенчмаркайте свой workload.** Размер сообщений и паттерн доступа вашего приложения уникальны — обобщённые тесты могут ввести в заблуждение.
4. **Смотрите на sustained, не peak.** Production — это марафон, а не спринт.
5. **Считайте TCO, не только throughput.** $10K/мес за 600 MB/s управляемого сервиса может быть выгоднее, чем $5K за self-hosted + инженер на полставки.

---

## Источники

1. Confluent Developer — Apache Kafka Performance (официальный бенчмарк): [confluent.io/learn/kafka-performance](https://developer.confluent.io/learn/kafka-performance/)
2. ComputingForGeeks — Kafka vs Redpanda: Real Benchmarks on Identical Hardware (апрель 2026): [computingforgeeks.com/kafka-vs-redpanda-benchmarks](https://computingforgeeks.com/kafka-vs-redpanda-benchmarks/)
3. Confluent Blog — Benchmarking RabbitMQ vs Kafka vs Pulsar (2024): [confluent.io/blog/kafka-fastest-messaging-system](https://www.confluent.io/blog/kafka-fastest-messaging-system/)
4. Vanlightly (Jack Vanlightly) — Kafka vs Redpanda OpenMessaging Benchmark (GitHub): [github.com/Vanlightly/openmessaging-benchmark-custom](https://github.com/Vanlightly/openmessaging-benchmark-custom)
5. OpenMessaging Benchmark Framework: [openmessaging.cloud/docs/benchmarks](https://openmessaging.cloud/docs/benchmarks/)
6. Jeqo.dev — Benchmarking Kafka: Getting started with OpenMessaging Benchmark: [jeqo.dev/blog/benchmarking-apache-kafka/intro-omb](https://jeqo.dev/blog/benchmarking-apache-kafka/intro-omb/)
7. University of Helsinki — A Comparative Analysis of Apache Kafka and Apache Pulsar (Master's thesis, 2024): [helda.helsinki.fi](https://helda.helsinki.fi/server/api/core/bitstreams/3aab75ae-9584-4c82-b72f-84eced9dfe29/content)
8. AutoMQ — Kafka Benchmark Methodology: Run Fair Comparisons: [automq.com/blog/kafka-benchmark-methodology](https://www.automq.com/blog/kafka-benchmark-methodology-run-fair-comparisons)
9. DoubleCloud — Benchmarking Apache Kafka: performance-per-price (2024): [double.cloud/blog/posts/2024/06/benchmarking-apache-kafka-performance-per-price](https://double.cloud/blog/posts/2024/06/benchmarking-apache-kafka-performance-per-price.html)
10. ActiveWizards — Kafka Benchmarking Methodologies & Tools: [activewizards.com](https://activewizards.com/blog/kafka-benchmarking-methodologies-and-tools-for-performance)
11. Confluent Compare — Kafka vs Pulsar: [confluent.io/compare/kafka-vs-pulsar](https://www.confluent.io/compare/kafka-vs-pulsar/)
12. StreamNative — OpenMessaging Benchmark (GitHub): [github.com/streamnative/openmessaging-benchmark](https://github.com/streamnative/openmessaging-benchmark)
13. kindatechnical — Kafka Benchmarking Tools and Methodology: [kindatechnical.com](https://kindatechnical.com/kafka-streams/kafka-benchmarking-tools-and-methodology.html)
