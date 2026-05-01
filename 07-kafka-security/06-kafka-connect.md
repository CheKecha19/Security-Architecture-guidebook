# Kafka Connect: безопасность интеграций

## Часть 1: Что такое Kafka Connect?

Представь посредника. У тебя есть данные в базе данных, в файлах, в облаке. Ты хочешь, чтобы они попадали в Kafka. Или наоборот — из Kafka в другие системы.

**Kafka Connect** — это фреймворк для интеграции Kafka с внешними системами. **Source Connector** читает из системы и пишет в Kafka. **Sink Connector** читает из Kafka и пишет в систему.

Проблема: Connect — это дверь в Kafka. Если её не защитить — кто угодно может читать и писать данные.

## Зачем защищать Connect?

### Ситуация 1: Source Connector с паролем

JDBC Source Connector читает из PostgreSQL. В конфигурации:

    connection.url=jdbc:postgresql://db:5432/production
    connection.user=admin
    connection.password=supersecret123

Конфигурация хранится в Kafka (топик `connect-configs`). Если Kafka не защищена — любой может прочитать пароль от базы данных.

### Ситуация 2: Sink Connector без аутентификации

FileSink Connector пишет данные в файл. Без авторизации — любой может создать connector, который читает топик `payments` и пишет в `/tmp/payments.txt`. Данные утекли.

### Ситуация 3: Вредоносный Connector

Злоумышленник загружает JAR с вредоносным кодом в Kafka Connect. Connector выполняет произвольный код. Это RCE (Remote Code Execution).

## Часть 2: Аутентификация Connect

### Worker Configuration

    # connect-distributed.properties
    # Аутентификация в Kafka
    security.protocol=SASL_SSL
    sasl.mechanism=SCRAM-SHA-256
    sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="connect-worker" \
        password="worker-password";
    
    # SSL для подключения к Kafka
    ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
    ssl.truststore.password=truststore-password
    
    # Для продюсеров и консьюмеров внутри worker
    producer.security.protocol=SASL_SSL
    producer.sasl.mechanism=SCRAM-SHA-256
    producer.sasl.jaas.config=... // то же самое
    
    consumer.security.protocol=SASL_SSL
    consumer.sasl.mechanism=SCRAM-SHA-256
    consumer.sasl.jaas.config=... // то же самое

### REST API защита

Kafka Connect имеет REST API для управления connectors. По умолчанию — без аутентификации.

    # Защита REST API через basic auth (не production!)
    # Лучше — отключить внешний доступ или использовать reverse proxy с OAuth2
    
    # Ограничение listeners
    listeners=http://localhost:8083
    # Не слушать на всех интерфейсах!

### RBAC для Connect (Confluent)

В Confluent Platform:

    # Права на управление connectors
    ResourceType: ConnectCluster
    Principal: User:connect-admin
    Operation: ALL
    
    # Права на конкретный connector
    ResourceType: Connector
    ResourceName: jdbc-source-payments
    Principal: User:connect-admin
    Operation: ALL

## Часть 3: Authorization для Connectors

### ACL для внутренних топиков

Kafka Connect использует внутренние топики:
- `connect-configs` — конфигурация
- `connect-offsets` — оффсеты
- `connect-status` — статусы

    # Дать права worker на внутренние топики
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:connect-worker \
        --operation All --topic connect-configs \
        --operation All --topic connect-offsets \
        --operation All --topic connect-status

### ACL для source/sink топиков

    # Source connector читает БД и пишет в Kafka
    # Нужны права на запись в топик
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:connect-worker \
        --operation Write --topic source-payments \
        --operation Create --topic source-payments
    
    # Sink connector читает из Kafka и пишет в БД
    # Нужны права на чтение топика
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:connect-worker \
        --operation Read --topic payments \
        --operation Read --topic payments-dlq

### Group ACL для Connect

    # Worker использует consumer group
    kafka-acls.sh --bootstrap-server localhost:9092 \
        --add --allow-principal User:connect-worker \
        --operation Read --group connect-worker-group

## Часть 4: Шифрование конфигурации

### Externalized Secrets (Kafka 2.0+)

Вместо хранения паролей в конфигурации:

    # Использовать FileConfigProvider
    config.providers=file
    config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
    
    # В connector config:
    connection.password=${file:/etc/kafka-connect/secrets/db.password:password}

### Создание secrets файла

    # /etc/kafka-connect/secrets/db.password
    password=supersecret123
    
    # Права доступа
    chmod 600 /etc/kafka-connect/secrets/db.password
    chown kafka:kafka /etc/kafka-connect/secrets/db.password

### HashiCorp Vault Integration

    # Использовать VaultConfigProvider (Confluent)
    config.providers=vault
    config.providers.vault.class=io.confluent.kafka.config.provider.VaultConfigProvider
    
    # В connector config:
    connection.password=${vault:secret/data/db:password}

## Часть 5: Безопасность загрузки connectors

### Валидация JAR

    # Загрузка connector только из доверенных источников
    # Проверка подписи JAR
    # Сканирование на уязвимости
    
    # plugin.path — только доверенные директории
    plugin.path=/usr/share/kafka-connect/plugins
    
    # Права на директорию
    chmod 755 /usr/share/kafka-connect/plugins
    # Только root может писать
    chown root:root /usr/share/kafka-connect/plugins

### Запрет динамической загрузки

    # Если не нужно — отключить REST API полностью
    # Или использовать авторизацию на уровне reverse proxy
    
    # Конfluent: RBAC на управление connectors
    # Open Source: ограничить доступ к REST API через firewall

## Часть 6: Мониторинг Connect

### Что логировать

| Событие | Уровень | Описание |
|---------|---------|----------|
| Создание connector | INFO | Кто создал, какой |
| Изменение конфигурации | WARN | Важно отслеживать |
| Остановка connector | INFO | Когда и кем |
| Ошибки задач | ERROR | Почему упало |
| Перезапуск | INFO | Автоматический |

### Алерты

| Условие | Приоритет |
|---------|-----------|
| Новый connector создан вне окна обслуживания | Critical |
| Изменение конфигурации connector | High |
| Connector task failed > 3 раз | High |
| Необычный источник данных | Warning |

## Вывод

Безопасность Kafka Connect:
1. **Аутентификация** — SCRAM-SHA-256 или mTLS
2. **Authorization** — ACL на внутренние и рабочие топики
3. **Secrets** — внешнее хранилище, не в конфигурации
4. **REST API** — ограничить доступ, использовать reverse proxy
5. **Plugins** — только доверенные JAR, проверка подписи
6. **Мониторинг** — создание, изменение, остановка connectors

Connect — дверь в Kafka. Закрой её замком.

---
_Статья создана на основе анализа материалов по безопасности Kafka_