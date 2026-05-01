# Безопасность Kafka Streams и ksqlDB

## Часть 1: Что такое Kafka Streams?

Представь конвейер. На входе — сырые данные. На выходе — обработанные. Между ними — станции: фильтрация, агрегация, обогащение.

**Kafka Streams** — это библиотека для обработки потоков данных в Kafka. Приложение читает из топиков, обрабатывает, пишет в другие топики.

**ksqlDB** — SQL-слоёй над Streams. Позволяет писать запросы к потокам данных, как к таблицам.

Проблема: Streams-приложение имеет доступ к данным. Если его скомпрометировать — данные утечут.

## Часть 2: Аутентификация Streams-приложения

### Consumer и Producer в одном

Streams-приложение одновременно и читает (consumer), и пишет (producer).

    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "payment-processor");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
    
    // Аутентификация SASL/SSL
    props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, "SASL_SSL");
    props.put(SaslConfigs.SASL_MECHANISM, "SCRAM-SHA-256");
    props.put(SaslConfigs.SASL_JAAS_CONFIG,
        "org.apache.kafka.common.security.scram.ScramLoginModule required " +
        "username=\"streams-app\" password=\"app-password\";");
    
    // SSL
    props.put(SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG, "/var/private/ssl/kafka.client.truststore.jks");
    props.put(SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG, "password");
    
    KafkaStreams streams = new KafkaStreams(builder.build(), props);

### Application ID как группа

`application.id` используется как:
- Имя consumer group
- Префикс для внутренних топиков (changelog, repartition)
- Идентификатор в Kafka Coordinator

**Важно:** Уникальный application.id для каждого приложения. Иначе — конфликт групп.

## Часть 3: Авторизация для Streams

### Необходимые права

Streams-приложение использует внутренние топики:

| Топик | Назначение | Права |
|-------|-----------|-------|
| `input-topic` | Входные данные | Read |
| `output-topic` | Результат | Write, Create |
| `KSTREAM-AGGREGATE-STATE-STORE-changelog` | Состояние (changelog) | All (auto-created) |
| `KTABLE-join-repartition` | Repartition топик | All (auto-created) |

### ACL для Streams

    # Права на входной топик
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:streams-app \
        --operation Read --topic input-payments
    
    # Права на выходной топик
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:streams-app \
        --operation Write --topic processed-payments \
        --operation Create --topic processed-payments
    
    # Права на внутренние топики (префикс application.id)
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:streams-app \
        --operation All --topic payment-processor-
        --resource-pattern-type Prefixed
    
    # Права на группу
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:streams-app \
        --operation Read --group payment-processor

### State Store безопасность

Streams хранит состояние локально (RocksDB) и в changelog топиках.

    // Локальное состояние — шифрование диска
    // Changelog топик — шифрование в Kafka (топик-level encryption)
    
    // Настройка retention для changelog
    props.put(StreamsConfig.WINDOW_STORE_CHANGE_LOG_ADDITIONAL_RETENTION_MS_CONFIG, 86400000);

## Часть 4: Безопасность ksqlDB

### Аутентификация ksqlDB Server

    # ksql-server.properties
    security.protocol=SASL_SSL
    sasl.mechanism=SCRAM-SHA-256
    sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="ksql-server" password="server-password";
    
    # SSL
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks

### Authorization ksqlDB

ksqlDB выполняет DDL (CREATE STREAM, CREATE TABLE) — это создание топиков.

    # Права на DDL
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:ksql-server \
        --operation Create --topic '*'
    
    # Но лучше — ограничить префикс
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:ksql-server \
        --operation All --topic ksql- --resource-pattern-type Prefixed

### RBAC в Confluent ksqlDB

    # Создание роли
    confluent iam rbac role create KsqlUser
    
    # Назначение роли
    confluent iam rbac role-binding create \
        --principal User:analyst \
        --role KsqlUser \
        --ksql-cluster-id ksql-cluster

### Query Authorization

    -- Только чтение из определённых streams
    CREATE STREAM user_payments AS SELECT * FROM payments WHERE user_id = '123';
    
    -- Для этого нужны права на чтение payments и запись в user_payments

## Часть 5: Шифрование состояния

### Шифрование RocksDB (at rest)

    // Настройка через конфигурацию
    props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/lib/kafka-streams");
    
    // Шифрование диска на уровне OS (LUKS)
    // Или через Confluent: encrypted state stores

### Шифрование changelog топиков

    // Changelog топики содержат состояние
    // Настройка через topic-level encryption или disk encryption
    
    // Cleanup policy для changelog
    props.put(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_COMPACT);

## Часть 6: Мониторинг Streams

### Что мониторить

| Метрика | Значение |
|---------|----------|
| committed-offset | Обработка идёт |
| lag | Отставание от входного топика |
| process-latency | Задержка обработки |
| dropped-records | Потерянные записи |
| failed-tasks | Упавшие задачи |

### Безопасность-специфичные алерты

| Условие | Приоритет |
|---------|-----------|
| Streams app с новым application.id | Warning |
| Чтение из чувствительного топика | Info |
| Запись в неожиданный топик | High |
| Ошибки аутентификации | Critical |

## Вывод

Безопасность Kafka Streams:
1. **Аутентификация** — SASL/SSL для приложения
2. **ACL** — права на входные, выходные и внутренние топики
3. **Application ID** — уникальный, с префиксом для внутренних топиков
4. **State Store** — шифрование локального и changelog состояния
5. **ksqlDB** — RBAC, ограничение DDL
6. **Мониторинг** — lag, latency, failed tasks

Streams обрабатывает данные. Защити обработку.

---
_Статья создана на основе анализа материалов по безопасности Kafka_