# Архитектура Kafka и безопасность распределённого журнала

## Часть 1: Что такое Kafka?

Представь центральную диспетчерскую аэропорта. Каждый рейс приземляется — диспетчер делает запись в журнал: рейс SU-1402, полоса 2, 14:35. Служба багажа читает этот журнал и отправляет тележки. Служба питания читает тот же журнал — готовит еду. Пограничный контроль сверяется — выставляет наряды. Все читают одну и ту же запись, каждый делает свою работу.

**Apache Kafka** — это именно такой «диспетчерский журнал» для данных. Распределённая система, которая принимает потоки событий, сохраняет их, и позволяет множеству приложений читать эти события независимо друг от друга.

Ключевое отличие Kafka от обычной очереди сообщений: Kafka **сохраняет данные на диске**, даже после прочтения. Записи хранятся заданное время (retention), и любой потребитель может «перемотать» назад и прочитать сначала.

## Зачем нужна безопасность Kafka?

### Ситуация 1: Финансовые транзакции

Банк использует Kafka для потока транзакций. Если злоумышленник получит доступ к топику `transactions` — он увидит все переводы, балансы, счета клиентов.

### Ситуация 2: Конкурентная разведка

E-commerce компания публикует заказы в Kafka. Конкурент подключился к топику — видит цены, объёмы, тренды. Это коммерческая тайна.

### Ситуация 3: Внутренняя угроза

Сотрудник имеет доступ к топику с зарплатами (`salaries`). Он читает данные коллег, сохраняет, уходит в конкурентную компанию с этими данными.

## Часть 2: Основные компоненты

### Брокер (Broker)

**Брокер** — это сервер Kafka. Хранит данные, обрабатывает запросы продюсеров и консьюмеров. В production — кластер из нескольких брокеров (обычно 3-9).

| Параметр | Описание | Безопасность |
|----------|----------|--------------|
| `broker.id` | Уникальный ID брокера | — |
| `listeners` | Слушаемые адреса | Только внутренние сети |
| `log.dirs` | Директория с данными | Шифрование диска |
| `offsets.topic.replication.factor` | Репликация offset-топика | Минимум 3 |

### Топик (Topic)

**Топик** — это категория сообщений. Например, `payments`, `orders`, `user-events`.

| Параметр | Описание | Безопасность |
|----------|----------|--------------|
| `partitions` | Количество партиций | Влияет на параллелизм |
| `replication.factor` | Репликация | Минимум 3 для надёжности |
| `retention.ms` | Время хранения | Короче для sensitive данных |
| `cleanup.policy` | Удаление или компактирование | `delete` для sensitive |

### Партиция (Partition)

Топик разбивается на **партиции** — подмножества сообщений. Каждая партиция — это упорядоченный лог.

    [Partition 0]  [Partition 1]  [Partition 2]
    offset 0        offset 0        offset 0
    offset 1        offset 1        offset 1
    ...             ...             ...

**Безопасность:** Сообщения внутри партиции упорядочены. Но порядок между партициями не гарантирован. Для sensitive данных — может потребоваться один partition (потеря параллелизма, но строгий порядок).

### Producer (Продюсер)

Приложение, которое пишет в Kafka.

    Producer
       |
       v
    [Kafka Broker]
       |
       +---> Partition 0
       +---> Partition 1
       +---> Partition 2

**Безопасность:** Producer должен быть аутентифицирован. Только авторизованные продюсеры могут писать в конкретный топик.

### Consumer (Консьюмер)

Приложение, которое читает из Kafka.

    [Kafka Broker]
       |
       +---> Consumer Group A (3 инстанса)
       +---> Consumer Group B (1 инстанс)

**Безопасность:** Консьюмеры объединяются в **Consumer Group**. Каждый partition читается одним консьюмером в группе. Нужно контролировать: кто может присоединяться к группе.

### ZooKeeper / KRaft

**ZooKeeper** (устаревает) или **KRaft** (новое) — управляет метаданными кластера: кто брокер-лидер, какие топики существуют.

| | ZooKeeper | KRaft |
|---|-----------|-------|
| Роль | Внешняя зависимость | Встроено в Kafka |
| Безопасность | Отдельная аутентификация | Единая с Kafka |
| Сложность | Выше (3 системы) | Ниже (только Kafka) |
| Рекомендация | Мигрировать на KRaft | Использовать для новых |

## Часть 3: Модели безопасности Kafka

### Модель 1: PLAINTEXT (небезопасная)

    [Producer] ----> [Broker] ----> [Consumer]
        |                               |
        +-- Нет шифрования              +-- Нет аутентификации
        +-- Нет аутентификации          +-- Любой может читать

**Использование:** Только для разработки и тестов. В production — запрещена.

### Модель 2: SSL/TLS (шифрование)

    [Producer] --SSL/TLS--> [Broker] --SSL/TLS--> [Consumer]
        |                               |
        +-- Сертификат клиента            +-- Сертификат сервера
        +-- Сертификат сервера            +-- Проверка CA

**Что защищает:**
- Перехват трафика (man-in-the-middle)
- Чтение данных в пути

**Что НЕ защищает:**
- Кто подключается (нужна аутентификация)
- Что можно делать (нужна авторизация)

### Модель 3: SASL_SSL (аутентификация + шифрование)

    [Producer] --SASL_SSL--> [Broker] --SASL_SSL--> [Consumer]
        |                               |
        +-- SCRAM-SHA-256               +-- SCRAM-SHA-256
        +-- или GSSAPI (Kerberos)      +-- или GSSAPI
        +-- или OAuth Bearer            +-- или OAuth Bearer

**Что защищает:**
- Кто подключается (аутентификация)
- Перехват (шифрование)

**Что НЕ защищает:**
- Что можно делать (нужна авторизация)

### Модель 4: mTLS (взаимная аутентификация)

    [Producer] --mTLS--> [Broker] --mTLS--> [Consumer]
        |                               |
        +-- Сертификат клиента          +-- Сертификат клиента
        +-- Проверка CN                 +-- Проверка CN

**Что защищает:**
- Кто подключается (сертификат)
- Перехват (шифрование)
- Проверка идентичности (CN в сертификате)

### Модель 5: Полная защита (SASL_SSL + ACL + Audit)

    [Producer] --SASL_SSL--> [Broker] --SASL_SSL--> [Consumer]
        |              |                   |              |
        |              v                   |              v
        |    [Authentication]              |    [Authentication]
        |    SCRAM-SHA-256                 |    SCRAM-SHA-256
        |                                  |
        v                                  v
    [Authorization]                  [Authorization]
    ACL: Write to topic              ACL: Read from topic
        "orders"                         "orders"
        |
        v
    [Audit]
    Log: User:app1 wrote to orders at 14:35

## Часть 4: Настройка безопасности

### Шаг 1: Отключить PLAINTEXT

    # server.properties
    # Неправильно:
    listeners=PLAINTEXT://:9092
    
    # Правильно:
    listeners=SASL_SSL://:9093
    inter.broker.listener.name=SASL_SSL

### Шаг 2: Настроить SSL

    # server.properties
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.key.password=key-password
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.truststore.password=truststore-password
    
    ssl.client.auth=required  # Для mTLS
    ssl.enabled.protocols=TLSv1.2,TLSv1.3

### Шаг 3: Настроить SASL

    # server.properties
    sasl.enabled.mechanisms=SCRAM-SHA-256
    sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
    
    # JAAS конфигурация для брокера
    listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=\
        org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="broker" \
        password="broker-password";

### Шаг 4: Настроить ACL

    # Включить авторизацию
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    allow.everyone.if.no.acl.found=false  # Запретить по умолчанию

### Шаг 5: Настроить аудит

    # Логирование
    log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender
    log4j.additivity.kafka.authorizer.logger=false

## Часть 5: Топология безопасного кластера

### Однодатацентровая топология

    [Producer]     [Producer]
       |              |
       v              v
    [Load Balancer]
       |
       v
    +-----------------+
    |  Kafka Cluster  |
    |  +-----------+  |
    |  | Broker 1  |  |
    |  | (Leader)  |  |
    |  +-----------+  |
    |  +-----------+  |
    |  | Broker 2  |  |
    |  | (Follower)|  |
    |  +-----------+  |
    |  +-----------+  |
    |  | Broker 3  |  |
    |  | (Follower)|  |
    |  +-----------+  |
    +-----------------+
       |
       v
    [Consumer]     [Consumer]

### Мультидатацентровая топология (MirrorMaker)

    [DC1: Kafka Cluster]  <---MirrorMaker--->  [DC2: Kafka Cluster]
         |                                          |
         v                                          v
    [Producers/Consumers]                   [Producers/Consumers]
    
    # MirrorMaker — репликация между DC
    # Каждый DC — независимый кластер
    # Данные реплицируются для DR

## Вывод

Архитектура Kafka и безопасность:
1. **Компоненты:** Брокер, топик, партиция, producer, consumer
2. **Модели безопасности:** От PLAINTEXT до SASL_SSL + ACL + Audit
3. **Настройка:** SSL → SASL → ACL → Audit
4. **Топология:** Одна или несколько зон, MirrorMaker для DR

Kafka — центральная нервная система данных. Защити её как банк.

---
_Статья создана на основе анализа материалов по безопасности Kafka_