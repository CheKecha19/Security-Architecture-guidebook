# 10.1 — Roadmap Apache Kafka: KIP-процесс, планы развития 2026–2027

> **Bottom Line:** Apache Kafka вошёл в фазу зрелости, но не замедлился. Версия 4.0 (2025) завершила удаление ZooKeeper, 4.1 дала Preview для Queues for Kafka (KIP-932), а 4.2 (февраль 2026) — Production-Ready share groups. 4.3 ожидается в мае 2026, и далее по расписанию: 3 минорных релиза в год — 4.4 (сентябрь), 4.5 (январь 2027). Главные направления roadmap: полномасштабная очередь на базе Kafka, tiered storage (KIP-405), broker-driven Streams rebalancing (KIP-1071), эволюция KRaft, observability, безопасность и AI/ML-интеграции.

---

## 1. Механизм развития: что такое KIP

Apache Kafka развивается через формализованный процесс **KIP (Kafka Improvement Proposal)** — аналог Python PEP, Java JEP или Kubernetes KEP. Процесс был введён в 2015 году и с тех пор стал главным двигателем эволюции платформы.

### 1.1 Жизненный цикл KIP

Каждый KIP проходит стандартный путь:

1. **Идея и обсуждение** — автор публикует черновик на [Confluence-вики Apache Kafka](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)
2. **[DISCUSS]-тред в mailing list** — сообщество обсуждает proposal, высказывает замечания
3. **Голосование [VOTE]** — после доработки автор выносит KIP на голосование. Требуется минимум 3 binding-голоса от PMC-членов
4. **Реализация** — accepted KIP имплементируется через стандартный workflow JIRA → Pull Request → Code Review
5. **Релиз** — KIP включается в ближайший минорный релиз, соответствующий KIP Freeze-дате

По состоянию на март 2026 года в каталоге Kafka Improvement Proposals зарегистрировано **свыше 1300 KIP** (последние — KIP-1293 → KIP-1308, поданные в марте 2026).

Важные KIP-вехи:
- **KIP-1:** формализация самого KIP-процесса (2015)
- **KIP-500:** замена ZooKeeper на KRaft — самый масштабный архитектурный KIP в истории
- **KIP-932:** Queues for Kafka — добавление очередей в event-streaming платформу
- **KIP-405:** Tiered Storage — разделение hot/cold-данных

### 1.2 Календарь релизов

Apache Kafka перешёл на **Time-Based Release Plan**: 3 минорных релиза в год, примерно каждые 4 месяца.

**Фактический трек (2025–2026):**

| Версия | Дата релиза | Ключевые фичи | Статус |
|--------|-------------|---------------|--------|
| **4.0.0** | Март 2025 | KRaft default, ZK removal complete, Queues EA, consumer rebalancing v2 (KIP-848) | Supported |
| **4.0.1** | Сентябрь 2025 | Bug fixes | Supported |
| **4.0.2** | Март 2026 | 43 bugfixes, CVE fixes (log4j, lz4, jetty) | Supported |
| **4.1.0** | Июль 2025 | Queues Preview, Connect improvements, Streams enhancements | Supported |
| **4.1.2** | Март 2026 | 31 bugfixes, CVE updates | Supported |
| **4.2.0** | Февраль 2026 | Queues GA, Streams rebalance GA, 38 KIPs | Latest |
| **4.3.0** | Май 2026 (план) | Feature freeze 18 марта, code freeze 8 апреля | In progress |
| **4.2.1** | ~Апрель 2026 | Bugfix release, RM: PoAn Yang | Planned |
| **4.4.0** | ~Сентябрь 2026 (план) | — | Planned |
| **4.5.0** | ~Январь 2027 (план) | — | Planned |

> **Примечание:** Kafka 4.2.0 — текущий актуальный релиз (February 17, 2026). Поддерживаются релизы 4.0.x, 4.1.x, 4.2.x. Версия 3.9.x (последняя с ZooKeeper) больше не поддерживается с выходом 4.0.

---

## 2. Ключевые KIP 2025–2026: что уже сделано

### 2.1 KIP-932: Queues for Kafka (GA в 4.2)

**Самый значимый feature KIP со времён KIP-500.** После двух лет разработки Queues for Kafka прошли путь:

- **4.0** — Early Access (не для production)
- **4.1** — Preview (для evaluation и тестирования)
- **4.2** — General Availability (Production-Ready)

**Что такое share groups:** новый тип consumer group, где несколько consumer'ов могут одновременно обрабатывать сообщения из одной партиции. Это радикально отличается от классической модели, где партиция эксклюзивно принадлежит одному consumer'у.

Ключевые возможности:
- **Per-message acknowledgement** — можно подтверждать/отклонять индивидуальные сообщения, а не только commit'ить offset
- **Нет головной блокировки (head-of-line blocking)** — если одно сообщение «застряло», остальные продолжают обрабатываться
- **Кооперативная обработка** — число активных consumer'ов может превышать число партиций
- **Delivery attempt limit** — настраиваемое через `group.share.delivery.attempt.limit`
- **Брокерный share-partition leader** — координирует in-flight messages в пределах скользящего окна между SPSO и SPEO

Связанные KIP в 4.2:
- **KIP-1222:** renewal acquisition lock timeout для explicit-режима
- **KIP-1224:** адаптивный `append.linger.ms` для group/share coordinator
- **KIP-1226:** persistence и retrieval share partition lag (метрики лага для share groups)
- **KIP-1206:** строгий max fetch records в share fetch

### 2.2 KIP-1071: Streams Rebalance Protocol (Первая фаза)

Broker-driven ребалансировка для Kafka Streams — вместо координации через consumer group protocol, Streams-приложения теперь используют брокер как центральный координатор. Это решает давние проблемы:

- Отсутствие stop-the-world ребалансировок
- Более быстрое перераспределение задач
- Возможность incremental rebalancing

В 4.2 первая фаза с **limited feature set**, полная версия ожидается в 4.3+.

### 2.3 KIP-848: Next Generation Consumer Rebalance Protocol

Полностью новый протокол ребалансировки consumer group, вошедший в 4.0:

- **Incremental cooperative rebalancing** — consumers не отзывают партиции все разом, а перераспределяют постепенно
- **Брокерный group coordinator** — брокер, а не consumer leader, теперь координирует группу
- Основа для работы share groups (KIP-932 использует этот же протокол)

### 2.4 Другие KIP в 4.2

| KIP | Область | Суть |
|-----|---------|------|
| KIP-1034 | Streams | Dead letter queue в Kafka Streams |
| KIP-1054 | Connect | External schema support в JSONConverter |
| KIP-1146 | Streams | Anchored punctuation (привязка punctuation к wall-clock) |
| KIP-1157 | Security | Enforced KafkaPrincipalSerde implementation |
| KIP-1160 | Core | API для supported features конкретного брокера |
| KIP-1186 | Core | Update AddRaftVoterRequest для auto-join (KRaft) |
| KIP-1188 | Connect | ConnectorClientConfigOverridePolicy с allowlist |
| KIP-1190 | Core | Метрика idle controller thread |
| KIP-1216 | Streams | Rebalance listener metrics |
| KIP-1227 | Tools | Expose Rack ID в MemberDescription и ShareMemberDescription |
| KIP-1228 | Core | Transaction Version в WriteTxnMarkersRequest |
| KIP-1230 | Streams | Config для file system permissions |

---

## 3. KIP в разработке: что ждёт в 4.3 и далее

### 3.1 Мартовские KIP (1293–1308)

В марте 2026 сообщество подало 16 новых KIP. Ключевые из них:

- **KIP-1295:** Dead letter queue для GlobalKTable (расширение KIP-1270) — записи, которые не удалось обработать при загрузке GlobalKTable, можно отправлять в DLQ
- **KIP-1303:** Deprioritize Tiered Storage Followers в Leader Election — новый механизм для Kafka 4.3+, который предпочитает реплики с большим объёмом локальных данных при выборе лидера в топиках с tiered storage
- **KIP-1307:** Метрики для SerDe и Interceptors — время выполнения Serializer/Deserializer и Interceptor, error rate

### 3.2 Ожидаемые направления 4.3

На основе анализа KIP Freeze 4.3 (18 марта 2026) и открытых обсуждений:

**Tiered Storage (KIP-405) — зрелость:**

После введения Tiered Storage в 3.6 (Early Access), 4.0–4.2 принесли production-ready функционал. KIP-1023 (4.3) позволит новым репликам стартовать с earliest pending upload offset — то есть копировать только те данные, которые ещё не отправлены в remote storage. Это даёт:
- Мгновенное вхождение реплики в ISR
- Снижение стоимости хранения в 3–9× (за счёт S3/GCS вместо локальных дисков)
- Ускорение восстановления после сбоя в 115× (по сравнению с полным копированием)

KIP-1303 решает trade-off: реплика с меньшим количеством локальных данных не должна становиться лидером (иначе придётся читать из remote storage, что замедлит производительность).

**Queues for Kafka — дальнейшее развитие:**

После GA share groups в 4.2, сообщество фокусируется на:
- **Key-based ordering** — возможность сохранения порядка сообщений с одинаковым ключом внутри share group (сейчас порядок не гарантируется)
- **Follower reads для share partitions** — сейчас share-partition leader обязан быть на том же брокере, что и partition leader (нет возможности читать с follower)
- **Share group state persistence** — улучшение персистентности состояния share groups

**KRaft — продолжение эволюции:**

- **KIP-1186** (в 4.2) добавил auto-join для KRaft voter'ов — новый брокер может автоматически присоединиться к metadata quorum
- **KIP-966:** JBOD support для KRaft — полная поддержка множественных дисковых директорий в KRaft-режиме
- **KIP-1089:** Controller auto-scaling и динамическое изменение размера quorum
- **Cordon brokers (KIP-1066):** возможность «оградить» брокер для планового обслуживания без прерывания трафика; отложено на 4.3+ из 4.2

**Observability и метрики:**

- Клиентская телеметрия через ClientTelemetry API (KIP-714, KIP-1217)
- Улучшенная observability для координаторов (group/share)
- Метрики SerDe/Interceptors (KIP-1307)
- Мониторинг idle-потоков: controller (KIP-1190), metadata loader (KIP-1229)

**Безопасность:**

- Ужесточение KafkaPrincipalSerde (KIP-1157 в 4.2)
- Allowlist-based ConnectorClientConfigOverridePolicy (KIP-1188)
- OAuth2/OIDC improvements для SASL OAUTHBEARER

---

## 4. Долгосрочный взгляд: 4.4 → 4.5 (конец 2026 — начало 2027)

### 4.1 Предполагаемые треки (на основе community discussions и паттерна KIP)

> **Дисклеймер:** Следующие направления основаны на community discussions, открытых KIP и паттернах развития. Официальный roadmap на 2027 не опубликован. Спекулятивные прогнозы помечены ⚠️.

**⚡ Tiered Storage 2.0:**
- Автоматическое управление политиками хранения на основе ML-моделей доступа к данным
- Поддержка multi-cloud remote storage (один кластер → S3 + GCS + Azure Blob одновременно)
- Сжатие (compression) для remote-сегментов
- Point-in-time recovery из remote storage без полного рестор

**⚡ Queues 2.0:**
- Гарантированная доставка с key-based ordering в share groups
- Интеграция с внешними системами очередей (JMS-совместимость?)
- Dead-letter queue на уровне share group

**⚡ Streams 2.0:**
- Полный broker-driven rebalancing (вторая фаза KIP-1071)
- State store tiering — вынос state store на S3 для экономии локального хранилища
- Интерактивные запросы (Interactive Queries) поверх tiered state stores
- Встроенная поддержка windowed-агрегаций для потоков с late-arriving events

**⚡ AI/ML Integration:**
- Kafka как data pipeline для AI/ML: нативная интеграция с feature stores (Feast, Tecton)
- In-flight трансформации данных через ML-модели (Kafka Streams + ONNX Runtime)
- Agentic AI patterns — Kafka как event-bus для multi-agent AI систем (Confluent Streaming Agents)

**⚡ Операционное упрощение:**
- Автоматическое обнаружение (auto-discovery) брокеров без ручной конфигурации `bootstrap.servers`
- Dynamic rate limiting без перезагрузки брокера
- Graceful degradation при частичной потере кластера
- Унификация CLI-инструментов (частично начато в KIP-1147)

**⚡ Протокол:**
- HTTP/3 (QUIC) как альтернатива текущему бинарному протоколу поверх TCP
- Возможный gRPC-вариант для клиент-брокерного взаимодействия
- WebSocket-прокси для браузерного доступа к Kafka

---

## 5. Confluent Roadmap: как коммерческий вектор влияет на Apache Kafka

Confluent — компания-основатель и главный контрибьютор Apache Kafka — развивает параллельную коммерческую линейку. Их roadmap влияет на open-source Kafka, поскольку ~80% коммитов в Apache Kafka делают сотрудники Confluent.

### 5.1 Приоритеты Confluent (2026)

Согласно материалам Current 2025 и публичным заявлениям:

**Data Streaming Platform:**
- Унификация потоковой и batch-обработки: Apache Flink стал first-class citizen экосистемы Confluent
- Tableflow — автоматическое преобразование Kafka-топиков в Iceberg/Delta Lake-таблицы (shift-left analytics)

**AI-Native Platform:**
- **Confluent Intelligence** — real-time, context-aware AI поверх Kafka
- **Streaming Agents** — автоматизация бизнес-процессов с AI-агентами, коммуницирующими через Kafka

**Developer Experience:**
- Улучшение Confluent CLI и VS Code extension
- Managed Connectors (200+ коннекторов в Confluent Marketplace)
- Schema Registry 2.0 с расширенной поддержкой Protobuf, Avro и JSON Schema

**Cloud-Native:**
- Serverless Kafka (Confluent Cloud с автоскейлингом)
- Multi-cloud networking (Private Link, VPC Peering)
- Confluent Private Cloud — bridge между on-prem и cloud

### 5.2 WarpStream — Kafka-совместимый транспорт

Confluent приобрела WarpStream в 2024. Это Kafka-совместимый протокол, работающий поверх S3 как storage layer, с агентами вместо брокеров. Развитие WarpStream идёт отдельно от core Kafka, но влияет на:

- Архитектурные идеи separation of compute/storage (аналогично KIP-405)
- Zero-disk Kafka архитектуры (все данные в S3)
- BYOC (Bring Your Own Cloud) модель развёртывания

---

## 6. Как читать roadmap: практические выводы

### 6.1 Для тех, кто уже использует Kafka

**Срочно (ближайшие 6 месяцев):**
- **Мигрируйте на Kafka 4.x**, если ещё на 3.x (3.9.x больше не поддерживается)
- **Планируйте миграцию на KRaft**, если ещё используете ZooKeeper (ZK удалён из кодовой базы в 4.0)
- **Оцените Queues for Kafka** — если у вас есть компонент очереди (RabbitMQ, SQS), возможно, share groups заменят его

**Средняя перспектива (6–12 месяцев):**
- **Оценивайте Tiered Storage** — если затраты на хранение значительны, KIP-405 может сократить их в разы
- **Переходите на новый Consumer Rebalance Protocol** — меньше stop-the-world пауз при ребалансировке

**Долгосрочно (12+ месяцев):**
- **Следите за AI/ML интеграциями** — Kafka станет ключевым компонентом AI data pipelines
- **Готовьтесь к HTTP/3 протоколу** — если ваши клиенты за файрволлами, QUIC может упростить коннективность

### 6.2 Для тех, кто только рассматривает Kafka

- **Kafka 4.2** — идеальная точка входа: KRaft стабилен, ZooKeeper не нужен, Queues for Kafka production-ready
- **Confluent Cloud** — если не хотите управлять кластером, возьмите managed вариант
- **Начинайте с KRaft-режима** — не учите ZooKeeper, он мёртв

---

## 7. Источники

1. **Apache Kafka Improvement Proposals** — [cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)
2. **Release Plan 4.2.0** — [cwiki.apache.org/confluence/display/KAFKA/Release+Plan+4.2.0](https://cwiki.apache.org/confluence/display/KAFKA/Release+Plan+4.2.0)
3. **Kafka Monthly Digest: March 2026** — [Red Hat Developer](https://developers.redhat.com/blog/2026/04/03/kafka-monthly-digest-march-2026)
4. **KIP-932: Queues for Kafka** — [cwiki.apache.org](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)
5. **Queues for Kafka (KIP-932) — Preview Release Notes** — [cwiki.apache.org](https://cwiki.apache.org/confluence/display/KAFKA/Queues+for+Kafka+%28KIP-932%29+-+Preview+Release+Notes)
6. **Let's Take a Look at... KIP-932: Queues for Kafka** — [morling.dev](https://www.morling.dev/blog/kip-932-queues-for-kafka/)
7. **Apache Kafka 4.2.0 Released: Share Groups, Streams & More** — [Confluent Blog](https://www.confluent.io/blog/apache-kafka-4-2-release/)
8. **Apache Kafka 4.0 Release: Default KRaft, Queues, Faster Rebalances** — [Confluent Blog](https://www.confluent.io/blog/latest-apache-kafka-release/)
9. **Apache Kafka Downloads (supported releases)** — [kafka.apache.org](https://kafka.apache.org/community/downloads/)
10. **Software Patterns Lexicon: Upcoming Features and KIPs** — [softwarepatternslexicon.com](https://softwarepatternslexicon.com/kafka/future-trends-and-the-kafka-roadmap/upcoming-features-and-kips-kafka-improvement-proposals/)
11. **Queue Support for Apache Kafka: KIP-932 and KMQ** — [InfoQ](https://www.infoq.com/news/2024/07/apache-kafka-queues/)
12. **Using Queues for Apache Kafka with Strimzi** — [Strimzi Blog](https://strimzi.io/blog/2025/08/20/queues-for-kafka/)

---

*Связанные статьи:*
- [01-roadmap.md](01-roadmap.md) — эта статья
- [02-trends.md](02-trends.md) — Тренды и индустриальная динамика
- [../09-real-world/01-case-studies.md](../09-real-world/01-case-studies.md) — Кейсы компаний, использующих Kafka
- [../02-basics/02-how-it-works.md](../02-basics/02-how-it-works.md) — Как работает Kafka
- [../02-basics/03-core-concepts.md](../02-basics/03-core-concepts.md) — Ключевые концепции
- [../06-performance/03-scalability.md](../06-performance/03-scalability.md) — Масштабирование Kafka
