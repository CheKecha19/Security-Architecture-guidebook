# Реальные кейсы: кто и как использует Apache Kafka в production

> **Следующая статья:** [Миграция на Kafka](02-migration.md)

**Главный вывод:** Apache Kafka — это не просто middleware для обмена сообщениями. Крупнейшие технологические компании превратили его в центральную нервную систему, обрабатывающую триллионы событий в день. Отсюда ключевое правило: если у вас поток данных вышел за пределы одного приложения — вам нужна стратегия event streaming, и Kafka в ней почти наверняка будет фундаментом.

---

## 1. LinkedIn: место рождения и крупнейший потребитель

**Масштаб:** 7 триллионов сообщений в день, более 100 кластеров, 4 000+ брокеров

LinkedIn не просто создал Kafka в 2010 году — он остаётся его крупнейшим пользователем. Сегодня через кластеры LinkedIn проходит поток данных, сопоставимый с сетевым трафиком небольшой страны.

### Архитектурные решения

- **Brooklin MirrorMaker** — собственный инструмент репликации между ЦОДами, заменяющий MirrorMaker 2 для балансировки нагрузки на межкластерном уровне.
- **Сотни consumer groups** — каждая команда подписывается на нужные топики независимо. Это модель data mesh на практике: владельцы данных публикуют события, потребители сами решают, что им нужно.
- **Kafka как source of truth** — данные о профилях, активности, рекрутинге и аналитике проходят через одни и те же топики, гарантируя консистентность.

### Ключевой урок

**Изоляция потребления от производства.** Когда у вас сотни команд-потребителей, каждая со своим темпом чтения, единственный способ не устроить каскадный сбой — модель «dumb broker, smart consumer», где брокер не знает о логике потребителей.

_Источники: [LinkedIn Engineering Blog](https://engineering.linkedin.com/kafka/running-kafka-scale), [LinkedIn — 7 trillion messages/day](https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages)_

---

## 2. Netflix: потоковая обработка 2 триллионов событий в день

**Масштаб:** 2 триллиона событий/день, пиковая нагрузка — 38 млн событий/сек во время live‑трансляций

Netflix использует Kafka в связке с Apache Flink, Mantis и Druid для того, что они называют **Keystone Pipeline** — центральный конвейер событий платформы.

### Сценарии использования

| Сценарий | Поток событий | Критичность |
|----------|-------------|-------------|
| Мониторинг качества воспроизведения | Clickstream, буферизация, смена разрешения | Критично (38M событий/с, SLA — секунды) |
| Персонализация рекомендаций | История просмотров, паузы, пропуски | Высокая |
| A/B-тестирование | Атрибуты экспериментов, метрики конверсии | Высокая |
| Операционная аналитика | Логи CDN, ошибки кодирования | Средняя |

### Архитектурный паттерн

Netflix строит конвейер по принципу **fan-out**: одно событие от CDN или клиентского устройства попадает в топик, после чего независимо читается Flink-джобами мониторинга, персонализации и аналитики. Это позволяет командам развивать потребителей независимо, не затрагивая источник данных.

### Ключевой урок

**Разделение критичных и некритичных потоков.** Netflix изолирует мониторинговые события (38 млн/с, low‑latency SLA) в отдельные кластеры, чтобы batch-аналитика не влияла на операционные метрики. Это плата за гарантии: нельзя смешивать потоки с разными требованиями к latency.

_Источники: [Factor House — Netflix Kafka Architecture](https://factorhouse.io/articles/netflix-kafka-architecture), [Netflix Tech Blog](https://netflixtechblog.com/)_

---

## 3. Uber: транспортная платформа в реальном времени

**Масштаб:** триллионы событий в день, миллионы GPS-сообщений в секунду

Uber — классический пример компании, где **вся бизнес-логика построена поверх event streaming**. Без Kafka Uber не смог бы существовать: каждая поездка генерирует сотни событий от запроса до оплаты.

### Сценарии использования

1. **Динамическое ценообразование (surge pricing)**
   - Миллионы GPS-событий от водителей и пассажиров → Kafka → Flink → Redis
   - Flink анализирует баланс спроса/предложения в реальном времени за секунды
   - Коэффициенты обновляются и возвращаются в приложение

2. **Geo‑spatial индексация**
   - Kafka хранит поток координат, Flink строит гео-индекс (H3 от Uber)
   - Система матчинга водитель-пассажир читает индекс напрямую из топиков

3. **Dead Letter Queue на Kafka**
   - Сообщения, которые потребитель не может обработать после N попыток, попадают в DLQ-топик
   - Отдельная команда мониторит DLQ и чинит интеграции

### Ключевой урок

**Event streaming — не роскошь, а фундамент.** Когда бизнес-логика не может ждать batch-обработку (пассажир не будет ждать 5 минут расчёта цены), Kafka становится единственным возможным выбором. И Uber доказывает это масштабом.

_Источники: [Factor House — Uber Kafka Architecture](https://factorhouse.io/articles/uber-kafka-architecture), [Uber Engineering Blog](https://eng.uber.com/)_

---

## 4. Walmart: триллионы сообщений с отказоустойчивостью 99.99%

**Масштаб:** триллионы сообщений в день, 25 000+ consumer-инстансов, мультиоблачная архитектура (private + public cloud)

Walmart использует Kafka для трёх критичных сценариев:
- **Перемещение данных** между унаследованными системами и облаком
- **Event-driven микросервисы** для онлайн-торговли
- **Потоковая аналитика** для управления запасами в реальном времени

### Архитектурные решения

- **Ручная балансировка consumer groups** — при 25K потребителей автоматическая ребалансировка создаёт каскадные сбои. Команда Walmart разработала собственный механизм распределения партиций.
- **Throttling на уровне продюсеров** — чтобы защитить брокеры от всплесков трафика в Black Friday, каждый продюсер ограничен квотой.
- **Мультикластерная архитектура** — отдельные кластеры для staging/production и для разных регионов, с асинхронной репликацией.

### Ключевой урок

**Не доверяйте автоматическому — на триллионном масштабе.** Rebalance, retention, compaction — всё, что Kafka делает автоматически, на масштабе Walmart требует ручного контроля. Команда активно мониторит и вмешивается в процессы, которые на меньшем масштабе работают «сами».

_Источники: [Confluent — Walmart real-time replenishment](https://www.confluent.io/blog/how-walmart-uses-kafka-for-real-time-omnichannel-replenishment/), [ByteByteGo — Walmart Kafka Setup](https://blog.bytebytego.com/p/the-trillion-message-kafka-setup)_

---

## 5. Spotify: доставка событий с гарантией 99.999%

**Масштаб:** 4.6 млн событий/сек на пике, 99.999% SLA на доставку

Spotify прошёл путь от self-hosted Kafka 0.8 до современной архитектуры с Kafka 3.7, Flink 1.19 и tiered storage.

### Эволюция Event Delivery Infrastructure (EDI)

1. **Kafka 0.8 (2015):** прототип на bare-metal, 2 млн событий/с, проблемы с ребалансировкой
2. **Переход на GCP Pub/Sub (2018–2020):** временный переход на managed-решение Google для упрощения операций
3. **Возврат к Kafka 3.7 (2024–2026):** с tiered storage, zstd-компрессией и idempotent-продюсерами

### Что заставило вернуться

- **Tiered storage (KIP-405):** снизил стоимость хранения в 3–9 раз при росте retention до 30 дней
- **Zstd-компрессия:** 30–40% экономии сетевого трафика
- **Exactly-once семантика:** критично для биллинга и аналитики прослушиваний
- **Kubernetes-native операторы (Strimzi):** автоматизация развёртывания

### Ключевой урок

**Managed-решение ≠ навсегда.** Spotify перешёл на Pub/Sub, а затем вернулся на Kafka, потому что экосистема Connectors, Streams и Schema Registry перевесила удобство managed-сервиса. Это урок тем, кто думает, что «облачный сервис решит все проблемы» — экосистема и сообщество важнее.

_Источники: [Spotify Engineering — EDI Migration](https://engineering.atspotify.com/2021/10/changing-the-wheels-on-a-moving-bus-spotify-event-delivery-migration), [Johal.in — Spotify Kafka Teardown](https://johal.in/architecture-teardown-spotifys-2026-music-streaming-uses-kafka/)_

---

## 6. JD.com, Grab, Tencent: Kafka в Азии на предельных нагрузках

### JD.com — e-commerce с эластичностью на секунды

JD.com использует diskless Kafka (AutoMQ) на Kubernetes для e-commerce core:
- **Проблема:** shared-nothing Kafka дублирует durability: репликация брокеров поверх репликации облачного хранилища
- **Решение:** stateless-брокеры, persistent storage вынесен в S3-совместимое объектное хранилище
- **Результат:** эластичность на уровне секунд при пиках трафика (китайские распродажи вроде Singles' Day)

### Grab — data engineering платформа

Grab (юго-восточный Uber) столкнулся с тем, что reassignment партиций требовал физического перемещения данных между брокерами, создавая простои:
- **Решение:** Diskless Kafka, где перебалансировка становится операцией над метаданными, а не копированием данных
- **Результат:** улучшенная производительность брокеров, сниженное потребление сети и хранилища

### Tencent Cloud EMR — стриминг как часть data-платформы

Tencent интегрировал AutoMQ в EMR (Elastic MapReduce):
- **Суть:** Kafka перестаёт быть островом stateful-инфраструктуры в окружении эластичного compute
- **Результат:** пользователи EMR обрабатывают крупномасштабные потоки через эластичную архитектуру

### Ключевой урок

**Diskless Kafka решает реальную боль облачных инсталляций.** Когда у вас уже есть S3-совместимое хранилище с durability 99.999999999%, зачем Kafka дублирует эту работу через ISR-репликацию? Архитектурно правильное решение — вынести persistence на уровень хранилища, а брокерам оставить compute и протокол.

_Источники: [AutoMQ — Kafka in Production: Grab, JD.com, Tencent](https://www.automq.com/blog/kafka-in-production-grab-jd-tencent-case-studies)_

---

## 7. Банки и финтех: fraud detection в реальном времени

Kafka в финансовом секторе — это не «удобно», а «обязательно». Регуляторы требуют мониторинга транзакций в реальном времени (особенно с приходом PSD2/PSD3), и batch-анализ post-factum уже недостаточен.

### Типовая архитектура

```
Транзакции (POS, ATM, переводы)
    │
    ▼
Kafka (idempotent producer, acks=all, compression=zstd)
    │
    ├──► Flink/Spark Streaming (ML-модели: anomaly detection, rule engine)
    │       │
    │       ├── normal → пропустить
    │       └── suspicious → Kafka alert topic
    │
    └──► Kafka Connect → S3/Iceberg → long-term audit trail
```

### Кто использует

- **ING:** real-time fraud scoring на Kafka + Flink для платежей по картам
- **Goldman Sachs:** транзакционный мониторинг для compliance (KYC/AML)
- **PayPal:** event-driven риск-скоринг с latency < 100ms

### Ключевой урок

**Exactly-once — не опция, а требование.** Потеря или дублирование сообщения о мошеннической транзакции может стоить миллионы. Финансовые инсталляции используют транзакционных продюсеров (transactional.id) и читателей с isolation.level=read_committed.

_Источники: [Confluent — Real-Time Streaming Prevents Fraud](https://www.confluent.io/blog/real-time-streaming-prevents-fraud/)_

---

## 8. Автомобильная индустрия: connected cars и умное производство

Kafka — стандарт де-факто для connected vehicle архитектур. BMW, Audi, Porsche и Tesla используют его для телеметрии в реальном времени.

### Сценарии

- **Connected Cars:** 10–100 ГБ телеметрии в день с одного автомобиля → Kafka → real-time аналитика/алерты
- **Predictive Maintenance:** датчики станков → Kafka → ML-модели → предсказание отказов
- **Smart Manufacturing (Industry 4.0):** PLC, MES-системы → Kafka → цифровой двойник производства
- **Customer 360:** все точки касания клиента (сайт, дилер, приложение, авто) → Kafka → единый профиль

### Масштаб

Современный автомобиль генерирует 25 ГБ данных в час (по оценкам Intel). При парке в 1 млн connected cars это 600 ТБ в день. Kafka — единственная система, способная принять и обработать такой поток с гарантией доставки.

### Ключевой урок

**IoT без event streaming не масштабируется.** MQTT хорош для устройства↔брокер, но для агрегации, аналитики и интеграции со множеством систем нужен event streaming backbone. Типовой паттерн: MQTT → Kafka Bridge → Kafka.

_Источники: [Kai Waehner — Kafka Automotive](https://www.kai-waehner.de/blog/2021/07/19/kafka-automotive-industry-use-cases-examples-bmw-porsche-tesla-audi-connected-cars-industrial-iot-manufacturing-customer-360-mobility-services/), [DZone — Kafka for Automotive and Manufacturing](https://dzone.com/articles/apache-kafka-landscape-for-automotive-and-manufact)_

---

## Общие паттерны и уроки

### Что объединяет всех

| Паттерн | Примеры | Почему это работает |
|---------|---------|-------------------|
| **Fan-out потребление** | Netflix, LinkedIn, Walmart | Одно событие → множество независимых потребителей, каждая команда развивается автономно |
| **Tier-разделение кластеров** | Netflix (critical vs analytical), Walmart (staging vs prod) | Нельзя смешивать потоки с разными SLA — latency-чувствительный мониторинг умрёт под batch-аналитикой |
| **Idempotent + transactional producers** | Все финансовые кейсы, Spotify | Exactly-once — единственная приемлемая гарантия для биллинга, fraud detection, аудита |
| **Ручной контроль на экстремальном масштабе** | Walmart (25K consumers), LinkedIn (7T msg/day) | Автоматика Kafka работает только до определённого порога, дальше — ручное управление |
| **Разделение compute и storage** | AutoMQ/diskless (JD.com, Grab, Tencent), tiered storage (Spotify) | Зрелый кластер почти всегда упирается в стоимость хранения и сложность ребалансировки |

### Когда Kafka НЕ нужна

- **Объём < 10 000 сообщений в день** — RabbitMQ или даже Redis Pub/Sub проще в эксплуатации
- **Один потребитель, простой pipeline** — Cron + batch-скрипт дешевле и надёжнее
- **Нет требования к ordering гарантиям** — SQS, Pub/Sub, NATS дают меньше церемоний
- **Команда из 1–2 человек** — операционные издержки Kafka съедят всё время

### Дерево принятия решения

```
У вас > 100 000 событий/день?
├── Нет → RabbitMQ / Redis Pub/Sub / NATS
└── Да → Нужна ли replayability и ordering?
    ├── Нет → Cloud Pub/Sub / SQS
    └── Да → Критична ли latency < 100ms?
        ├── Нет → Kafka Connect + batch
        └── Да → Нужен exactly-once?
            ├── Нет → Kafka (acks=1, idempotent)
            └── Да → Kafka (acks=all, transactional)
```

---

## Сводная таблица кейсов

| Компания | Масштаб | Ключевой сценарий | Уникальное решение | SLA |
|----------|---------|-------------------|-------------------|-----|
| LinkedIn | 7T msg/day, 4K+ brokers | Data movement, source of truth | Brooklin MirrorMaker | 99.99% |
| Netflix | 2T events/day, 38M/s peak | Quality monitoring, Keystone Pipeline | Tier-изоляция кластеров | секунды |
| Uber | Trillions/day, M GPS/s | Surge pricing, geo-spatial | H3-индекс на Kafka | < 1с |
| Walmart | Trillions/day, 25K consumers | Inventory, микросервисы | Ручная балансировка | 99.99% |
| Spotify | 4.6M events/s, 99.999% SLA | Event Delivery (EDI) | Tiered storage, zstd | 99.999% |
| JD.com | E-commerce core, Kubernetes | Cloud-native e-commerce | Stateless brokers | секунды эластичность |
| Grab | Data engineering platform | Data pipeline | Metadata-level reassignment | высокая |
| Tencent EMR | Cloud provider integration | Stream+Analytics | S3-backed persistence | Cloud SLA |
| Финансы | M транзакций/день | Fraud detection, AML/KYC | Exactly-once гарантии | < 100ms |
| Автопром | 25 ГБ/ч/авто, M автомобилей | Predictive maintenance, connected cars | MQTT → Kafka Bridge | real-time |

---

## Следующие шаги в изучении

- [Миграция на Kafka: стратегии, pitfalls и rollback-планы](02-migration.md) — как компании переходят на Kafka с унаследованных систем
- [Антипаттерны Kafka](03-anti-patterns.md) — что делают неправильно и как это чинить
- [Бенчмарки и сравнения](04-benchmarks-comparison.md) — реальные цифры, а не маркетинг

---

## Источники и ссылки

1. LinkedIn Engineering Blog — Running Kafka At Scale (2015)
2. LinkedIn Engineering Blog — How LinkedIn Customizes Apache Kafka for 7 Trillion Messages Per Day (2022)
3. Factor House — How Netflix uses Apache Kafka in production (2026)
4. Factor House — How Uber uses Apache Kafka in production (2026)
5. Factor House — How LinkedIn uses Apache Kafka in production (2026)
6. Confluent Blog — How Walmart Uses Kafka for Real-Time Omnichannel Replenishment
7. ByteByteGo — The Trillion Message Kafka Setup at Walmart (2024)
8. Spotify Engineering — Changing the Wheels on a Moving Bus: Spotify's Event Delivery Migration (2021)
9. Johal.in — Spotify 2026 Streaming: Kafka 3.7 & Flink 1.19 Teardown (2026)
10. AutoMQ Blog — Kafka in Production: Grab, JD.com, Tencent Case Studies (2026)
11. Kai Waehner — Apache Kafka in the Automotive Industry (2021)
12. Confluent Blog — How Real-Time Streaming Prevents Fraud in Banking & Payments
