# Миграция на Apache Kafka: стратегии, инструменты и подводные камни

> **Предыдущая статья:** [Реальные кейсы](01-case-studies.md) · **Следующая статья:** [Анти-паттерны](03-anti-patterns.md)

**Главный вывод:** Миграция на Kafka (или с Kafka) — это не DNS-переключение и не одноразовое мероприятие. Это контролируемое перекрытие двух систем, при котором старый кластер продолжает обслуживать production, пока новый доказывает свою состоятельность. Ключевой принцип: мигрируете не кластер — мигрируете рабочие нагрузки по одной.

---

## 1. Классификация миграционных сценариев

Прежде чем обсуждать стратегии — определим, о какой именно миграции речь. В экосистеме Kafka различают четыре принципиально разных сценария:

| Сценарий | Суть | Сложность | Типичные причины |
|----------|------|-----------|------------------|
| **С традиционной MQ на Kafka** | Замена RabbitMQ/ActiveMQ/IBM MQ на Kafka | Средняя | Рост объёмов, потребность в replay, аналитика |
| **Между Kafka-кластерами** | Переезд на новые брокеры (облако/DC/железо) | Высокая | Смена провайдера, апгрейд инфраструктуры |
| **ZooKeeper → KRaft** | Архитектурная миграция контроллера метаданных | Высокая | Подготовка к Kafka 4.0+, снятие лимита 200K партиций |
| **Версионный апгрейд в рамках Kafka** | 3.x → 4.x с KRaft | Низкая–средняя | Новые фичи, прекращение поддержки ZK |

В этой статье мы последовательно разберём все четыре сценария с практическими примерами.

---

## 2. Миграция с традиционных Message Queue на Kafka

Это наиболее частый сценарий: компания выросла, RabbitMQ упирается в throughput, и нужна горизонтально масштабируемая событийная шина.

### 2.1 Ключевые различия, которые надо осознать до миграции

**Парадигма:** MQ — это message-centric (брокер отвечает за доставку конкретному потребителю). Kafka — log-centric (потребители сами читают лог в своём темпе). Это фундаментальный сдвиг: RabbitMQ удаляет сообщение после подтверждения (ack), Kafka хранит всё с настраиваемым retention-периодом.

**Гарантии порядка:** В RabbitMQ порядок сообщений гарантируется только в рамках одной очереди с одним потребителем. В Kafka порядок гарантирован в пределах партиции — мощнейшее преимущество при миграции event-sourcing систем.

**Паттерны маршрутизации:** RabbitMQ exchange + binding → Kafka topic + partition key. Routing key становится ключом партиции, очереди становятся consumer groups.

### 2.2 Стратегия «Dual Write + Dual Read» (рекомендуемая)

Это наиболее безопасный путь — обе системы работают параллельно, постепенно переключая трафик.

**Фаза 1 — Подготовка (неделя до старта):**
- Инвентаризация всех exchanges, queues, bindings, consumer groups
- Документирование форматов сообщений и гарантий доставки
- Проектирование партиционирования (routing key → partition key)
- Развёртывание целевого Kafka-кластера с production-конфигурацией

**Фаза 2 — Dual Write (пишем в обе системы):**

```python
# Адаптер Dual-Write для постепенной миграции с RabbitMQ на Kafka
from kafka import KafkaProducer
import pika
import json

class DualWriteProducer:
    """Пишет сообщения одновременно в RabbitMQ и Kafka"""
    def __init__(self, rabbitmq_host, kafka_bootstrap):
        self.rabbit_conn = pika.BlockingConnection(
            pika.ConnectionParameters(host=rabbitmq_host))
        self.rabbit_channel = self.rabbit_conn.channel()
        self.kafka_producer = KafkaProducer(
            bootstrap_servers=kafka_bootstrap,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',               # Максимальная надёжность
            enable_idempotence=True   # Избегаем дубликатов при ретраях
        )
        self.kafka_enabled = False    # Флажок для постепенного включения

    def publish(self, exchange, routing_key, message, kafka_topic):
        # RabbitMQ (всегда работает)
        self.rabbit_channel.basic_publish(
            exchange=exchange, routing_key=routing_key,
            body=json.dumps(message))

        # Kafka (включается постепенно через feature flag)
        if self.kafka_enabled:
            self.kafka_producer.send(
                kafka_topic, key=routing_key, value=message)
```

**Фаза 3 — Backfill исторических данных:**

Для топиков, где нужна история (аналитика, переобучение моделей), единоразово переливаем накопленные данные:

```bash
# Инструмент: Kafka Connect RabbitMQ Source Connector
# Настройка коннектора для стриминга RabbitMQ → Kafka
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rabbitmq-source-orders",
    "config": {
      "connector.class": "io.confluent.connect.rabbitmq.RabbitMQSourceConnector",
      "kafka.topic": "orders_migrated",
      "rabbitmq.queue": "order_queue",
      "rabbitmq.host": "rabbitmq-old.internal",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "tasks.max": "4"
    }
  }'
```

**Фаза 4 — Постепенное переключение потребителей:**

Потребители переключаются группами, начиная с наименее критичных:
1. Аналитика и batch-потребители (могут пережить задержку)
2. Сервисы с высокой толерантностью к дубликатам (идемпотентные обработчики)
3. Критичные сервисы (платежи, заказы) — с пристальным мониторингом
4. Финальное отключение RabbitMQ

**Фаза 5 — Валидация и откат:**

```bash
# Сверка количества сообщений между системами
# RabbitMQ (до отключения)
rabbitmqctl list_queues name messages

# Kafka
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list kafka-new:9092 --topic orders_migrated
```

### 2.3 Типичные ошибки при миграции с MQ

1. **Игнорирование разницы в семантике подтверждений.** RabbitMQ auto-ack удаляет сообщение немедленно. В Kafka offset committ — явная операция. Если потребитель падает до commit — сообщение будет обработано повторно. Решение: делать обработку идемпотентной.

2. **Неправильный выбор ключа партиции.** Routing key «order.created» — плохой ключ партиции (100% сообщений в одну партицию). Нужен ключ с высоким кардиналитетом: `order_id`, `user_id`.

3. **Забывают про Schema Registry.** RabbitMQ не принуждает к схеме данных (любой байтовый blob). Kafka требует схемы для совместимости продюсеров и потребителей. Добавьте Schema Registry с самого начала.

---

## 3. Миграция между Kafka-кластерами (Zero-Downtime)

Самый сложный и ответственный сценарий: переезд production-нагрузок между разными физическими кластерами — смена облачного провайдера, переезд из on-prem в облако, замена железа.

### 3.1 Инвентаризация — начало начал

**Первый шаг — не MirrorMaker 2. Первый шаг — инвентаризация.** Кластер Kafka — это не монолит, а коллекция рабочих нагрузок с разными риск-профилями. Лог-топик и платёжный топик не должны делить один план миграции.

**Что инвентаризировать:**

```
Topic: payment.events
├── Владелец: команда Payments
├── Партиции: 32
├── Retention: 7 дней
├── Cleanup policy: delete
├── Продюсеры: payment-api (3 инстанса), fraud-detector (2)
├── Consumer groups: billing-service (lag tolerance ≤ 30s),
│   analytics-pipeline (может пережить до 2 часов задержки)
├── Schema Registry: payments-value (Avro, BACKWARD совместимость)
├── ACL: Read — billing, analytics | Write — payment-api, fraud-detector
├── Критичность: HIGH
└── Окно отката: ≤ 5 минут
```

**Группировка в волны миграции:** низкорисковые нагрузки (логи, метрики, аналитика) → среднерисковые → критические (платежи, заказы).

### 3.2 MirrorMaker 2 как основной инструмент

**MirrorMaker 2 (MM2)** — стандартный инструмент для непрерывной репликации между кластерами. Он построен на фреймворке Kafka Connect и умеет не только копировать данные, но и синхронизировать consumer offsets — критически важно для миграции.

**Ключевое:** настройте `offset-syncs.topic.location = target` — тогда топики синхронизации смещений создаются на целевом кластере, и пользователю MM2 не нужны права записи в исходный кластер.

```yaml
# Конфигурация MM2 для миграции (Strimzi custom resource)
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: migration-mm2
spec:
  version: 4.0.0
  replicas: 3
  connectCluster: target-cluster
  mirrors:
    - sourceCluster: source-cluster
      targetCluster: target-cluster
      sourceConnector:
        tasksMax: 32              # Параллелизм для скорости репликации
        config:
          offset-syncs.topic.location: target
          replication.factor: 3   # RF для internal-топиков MM2
          refresh.topics.interval.seconds: 30
          sync.topic.acls.enabled: true
      checkpointConnector:
        tasksMax: 8               # Синхронизация consumer offsets
        config:
          checkpoints.topic.replication.factor: 3
          sync.group.offsets.enabled: true
          emit.checkpoints.enabled: true
      topicsPattern: "payment\\..*|orders\\..*|inventory\\..*"
```

### 3.3 Таблица валидации перед переключением

Прежде чем переключать потребителей, проверьте следующие пункты:

| Проверка | Что доказывает | Сигнал опасности |
|----------|---------------|------------------|
| Lag репликации (< 100ms) | Целевой кластер почти синхронизирован | Lag растёт в пиковые часы |
| Синхронизация consumer offsets | Потребители могут безопасно продолжить чтение | Группа стартует слишком рано/поздно |
| Паритет конфигурации топиков | Поведение идентично исходному | Retention или compaction отличаются |
| Паритет ACL | Клиенты подключатся после переключения | Ошибки авторизации в dry run |
| Совместимость схем | Продюсеры и потребители согласуются | Deserialization errors |
| Тест отката | Путь назад работает | Исходный кластер не принимает клиентов |

### 3.4 Процедура переключения (cutover)

Переключение НЕ делается всем кластером сразу. Для каждой волны миграции:

1. **Зафиксировать конфигурацию** — запретить risky-изменения в исходном кластере на время переключения
2. **Подтвердить lag в пределах порога** — MirrorMaker 2 почти догнал исходный кластер
3. **Оповестить владельцев приложений** — дать окно в 30 минут
4. **Переключить потребителей ИЛИ продюсеров** — порядок зависит от топика:
   - Сначала потребителей, затем продюсеров — если опаснее потеря данных
   - Сначала продюсеров, затем потребителей — если опаснее дублирование
   - Для большинства случаев: потребители → продюсеры
5. **Приложение меняет bootstrap servers** — вот здесь AutoMQ/Kafka-совместимость экономит время: достаточно обновить строку подключения и security-конфигурацию
6. **Мониторинг на уровне приложения** — не только broker-метрики, но и бизнес-метрики

```bash
# Проверка lag репликации перед cutover
kafka-consumer-groups --bootstrap-server target-cluster:9092 \
  --group billing-service --describe

# Проверка, что все consumer offsets синхронизированы через MM2
kafka-consumer-groups --bootstrap-server source-cluster:9092 \
  --group billing-service --describe
kafka-consumer-groups --bootstrap-server target-cluster:9092 \
  --group billing-service --describe
# → OFFSET должны совпадать на обоих кластерах
```

### 3.5 План отката

**Откат — это не теоретическая опция, это часть плана.** Если что-то пошло не так:

1. Вернуть приложения на исходный `bootstrap_servers` (rollback deployment)
2. Убедиться, что MirrorMaker 2 продолжает репликацию в обратную сторону
3. Провести root cause analysis, не удаляя целевой кластер
4. Повторить переключение после исправления

---

## 4. Миграция ZooKeeper → KRaft

С выходом Apache Kafka 4.0 (февраль 2026) поддержка ZooKeeper полностью прекращена. Если вы всё ещё на ZK — миграция обязательна.

### 4.1 Почему это нужно сделать

| Характеристика | ZooKeeper | KRaft |
|---------------|-----------|-------|
| Лимит партиций | ~200 000 | Потенциально миллионы |
| Время восстановления контроллера | Часы (на 2M партиций) | Секунды |
| Администрирование | Kafka + ZK = 2 системы | Единая система |
| Время остановки брокера | Минуты | Секунды |
| Скорость reassignment | Медленная (sequential ZK writes) | Быстрая (Raft log) |

### 4.2 Четыре фазы миграции (KIP-866)

Миграция спроектирована так, что откат возможен на каждом этапе до финального:

**Фаза 1 — Подготовка:**
Включаем KRaft-миграцию на каждом брокере, не меняя роль. Кластер продолжает использовать ZooKeeper, но начинает дублировать метаданные в KRaft-лог.

```properties
# server.properties — добавляем на ВСЕХ брокерах
process.roles=broker                        # Пока только брокер
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
zookeeper.metadata.migration.enable=true    # Ключевой параметр KIP-866
```

**Фаза 2 — Развёртывание KRaft-контроллеров:**
Запускаем выделенные контроллеры (3 или 5 узлов, обязательно нечётное число). Они формируют Raft-кворум и принимают дублируемую метадату.

```properties
# controller.properties — для каждого контроллера
process.roles=controller
node.id=1                       # Уникальный ID
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
zookeeper.metadata.migration.enable=true
```

**Фаза 3 — Миграция метаданных:**
После стабилизации KRaft-кворума выполняем команду миграции:

```bash
# Переключение активного контроллера на KRaft
kafka-features.sh --bootstrap-server broker:9092 \
  upgrade --feature metadata.version=4.0

# Мониторинг статуса миграции
kafka-metadata-quorum.sh --bootstrap-server controller1:9093 \
  describe --status
# → Должен показать: "MigrationInProgress" → "DualWrite" → "KraftOnly"
```

**Фаза 4 — Отключение ZooKeeper:**
Только когда ВСЕ брокеры перешли в KRaft-режим и метаданные верифицированы:

```bash
# Отключение ZooKeeper на каждом брокере
# Изменяем server.properties:
#   zookeeper.connect= (пусто)
#   zookeeper.metadata.migration.enable=false

# Перезапускаем брокеры по одному
# После перезапуска всех — останавливаем ZooKeeper ensemble
```

### 4.3 Чек-лист перед миграцией ZK→KRaft

- [ ] Все брокеры обновлены до версии ≥ 3.5 (поддержка KIP-866)
- [ ] KRaft-контроллеры развёрнуты на выделенных узлах (не совмещены с брокерами)
- [ ] Кворум из 3 контроллеров для production (5 — для очень крупных кластеров)
- [ ] Отказоустойчивость проверена: остановка одного контроллера не ломает кластер
- [ ] Метрики KRaft добавлены в мониторинг (ActiveControllerCount, MetadataLag)
- [ ] Протестирован откат (остановка KRaft-контроллеров → возврат к ZK-контроллеру)
- [ ] План коммуникации с командами-потребителями готов

---

## 5. Версионный апгрейд Kafka (3.x → 4.x)

Это наиболее простой сценарий, но с критической оговоркой: если вы мигрируете с 3.x на 4.x, миграция ZK→KRaft должна быть выполнена ДО или одновременно.

### 5.1 Стандартная процедура rolling upgrade

```bash
# 1. Обновляем ПО на всех брокерах (без перезапуска)
# 2. Поочерёдный перезапуск брокеров с новой версией

# Перед обновлением — зафиксировать версию протокола
kafka-features.sh --bootstrap-server broker1:9092 \
  describe
# → Проверить: metadata.version совместим со старой версией

# После обновления всех брокеров — повысить версию протокола
kafka-features.sh --bootstrap-server broker1:9092 \
  upgrade --feature metadata.version=4.0

# Проверить результат
kafka-broker-api-versions.sh --bootstrap-server broker1:9092
```

### 5.2 Ключевые изменения в 4.0 (влияющие на миграцию)

1. **Полное удаление ZooKeeper-зависимости.** Если не мигрировали на KRaft — 4.0 не запустится.
2. **Новый протокол Consumer Rebalance (KIP-848).** Consumer groups с новым протоколом требуют версии клиента ≥ 3.9.
3. **Улучшенная поддержка Tiered Storage (KIP-405).** Требует обновления конфигурации брокеров для remote storage.
4. **Docker-образы без ZK.** Официальные образы 4.0 содержат только KRaft-режим.

---

## 6. Общий алгоритм принятия решений

```
Мигрируете на Kafka или меняете кластер?
│
├── Традиционная MQ → Kafka
│   └── Dual Write + Dual Read
│       → Начать с наименее критичных consumer groups
│       → Schema Registry с самой первой партии топиков
│
├── Kafka-кластер А → Kafka-кластер Б
│   ├── Инвентаризация → Группировка нагрузок по волнам
│   ├── MirrorMaker 2 с offset-syncs.topic.location=target
│   ├── Валидация: lag, offsets, ACL, схемы, rollback test
│   └── Cutover по одной волне, мониторинг на уровне приложения
│
├── ZK → KRaft (обязательно перед 4.0)
│   └── KIP-866: 4 фазы, откат возможен до последней
│
└── 3.x → 4.x
    └── Rolling upgrade + KRaft обязателен
```

---

## 7. Практические рекомендации (выжимка)

1. **Никогда не мигрируйте «большим взрывом» (Big Bang).** Даже для небольших систем фазовый подход безопаснее. Единственное исключение: кластер без production-нагрузок.

2. **Инвентаризация ценнее инструментов.** Не начинайте с MirrorMaker 2. Начните со списка топиков, их владельцев, потребителей и допустимого окна простоя. Кластер Kafka — это коллекция разных рабочих нагрузок, а не монолит.

3. **Тест отката — обязателен.** Если вы не протестировали возврат на исходную систему, у вас нет плана миграции — у вас есть надежда. Проведите минимум один dry run отката до начала production-миграции.

4. **Schema Registry — с первого дня.** Миграция без схем гарантирует проблемы с десериализацией при переключении потребителей. Avro с BACKWARD-совместимостью — золотой стандарт.

5. **Мониторинг на уровне приложения, не только брокеров.** Lag репликации может быть нулевым, а данные — некорректными. Сравнивайте бизнес-метрики до и после переключения.

6. **Не экономьте на контроллерах при ZK→KRaft.** 3 контроллера для production — минимум. Совмещение контроллера и брокера на одном узле допустимо только для dev-среды.

---

## Источники

1. [Red Hat Developer — Mastering Kafka migration with MirrorMaker 2](https://developers.redhat.com/articles/2024/01/04/mastering-kafka-migration-mirrormaker-2) — практическое руководство по MM2-миграции, 2024
2. [AutoMQ Blog — Zero-Downtime Kafka Migration](https://www.automq.com/blog/zero-downtime-kafka-migration-a-step-by-step-guide-for-production-clusters) — пошаговый план миграции production-кластеров, май 2026
3. [OSO Blog — Guide to ZooKeeper to KRaft migration](https://oso.sh/blog/guide-to-zookeeper-to-kraft-migration/) — уроки реальных продакшен-миграций ZK→KRaft, 2026
4. [Apache Kafka KIP-866](https://cwiki.apache.org/confluence/display/KAFKA/KIP-866+ZooKeeper+to+KRaft+Migration) — официальный дизайн миграции ZK→KRaft с возможностью отката
5. [Confluent Docs — Migrate from ZooKeeper to KRaft](https://docs.confluent.io/platform/current/installation/migrate-zk-kraft.html) — официальная документация Confluent, чек-листы
6. [Software Patterns Lexicon — Successful Migration Strategies](https://softwarepatternslexicon.com/kafka/case-studies-and-real-world-applications/migrating-legacy-systems-to-kafka/strategies-for-successful-migration/) — phased vs big bang, mapping legacy → Kafka
7. [CodeStudy — MQ to Kafka Migration Guide](https://www.codestudy.net/blog/mq-to-kafka-migration/) — практические примеры кода, от RabbitMQ к Kafka
8. [OneUptime — How to Migrate from RabbitMQ to Kafka](https://oneuptime.com/blog/post/2026-01-21-rabbitmq-to-kafka-migration/view) — migration checklist, dual-write bridge, январь 2026
9. [Redpanda — Kafka Migration Best Practices](https://www.openlogic.com/blog/kafka-migration-best-practices) — best practices и частые ошибки при миграции версий Kafka
10. [Confluent Developer — Dual-Write Problem](https://www.confluent.io/blog/dual-write-problem/) — Transactional Outbox pattern для решения проблемы dual-write
