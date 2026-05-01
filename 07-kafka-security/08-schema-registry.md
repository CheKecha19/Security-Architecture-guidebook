# Безопасность Schema Registry

## Часть 1: Что такое Schema Registry?

Представь библиотеку каталогов. В магазине есть все товары, но без каталога — хаос. С каталогом — порядок: каждый товар на своей полке, с описанием.

**Schema Registry** — это «каталог» для Kafka. Хранит версии схем данных (Avro, Protobuf, JSON Schema). Производитель пишет данные по схеме. Потребитель читает, зная структуру.

Без Schema Registry:
- Производитель меняет поле — потребитель падает
- Несовместимые версии — потеря данных
- Нет контроля над форматом

С Schema Registry:
- Версионирование схем
- Совместимость (backward, forward, full)
- Централизованное управление

## Часть 2: Зачем защищать Schema Registry?

### Ситуация 1: Подмена схемы

Злоумышленник регистрирует новую версию схемы с дополнительным полем `malicious_payload`. Потребители принимают данные, обрабатывают поле — выполняется вредоносный код.

### Ситуация 2: Удаление схемы

Злоумышленник удаляет схему `payments-value`. Все приложения, использующие эту схему, падают. Остановка бизнес-процессов.

### Ситуация 3: Несанкционированная регистрация

Разработчик регистрирует схему с PII-данными без маскирования. Данные попадают в Kafka в открытом виде. Compliance-нарушение.

## Часть 3: Аутентификация Schema Registry

### Basic Auth (Confluent)

    # schema-registry.properties
    authentication.method=BASIC
    authentication.realm=SchemaRegistry
    authentication.roles=SchemaRegistryAdmin,SchemaRegistryUser

    # Пользователи
    # password-file должен быть защищён
    schema.registry.basic.auth.user.info=admin:admin-password

### SASL/SSL (Kafka authentication)

    # Аутентификация через Kafka (Confluent)
    kafkastore.security.protocol=SASL_SSL
    kafkastore.sasl.mechanism=SCRAM-SHA-256
    kafkastore.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="schema-registry" password="password";

### mTLS

    # SSL для взаимной аутентификации
    ssl.client.auth=true
    ssl.keystore.location=/var/private/ssl/schema-registry.keystore.jks
    ssl.truststore.location=/var/private/ssl/schema-registry.truststore.jks

## Часть 4: Авторизация

### Subject-level authorization (Confluent)

    # schema-registry.properties
    schema.registry.group.authorizer.class=io.confluent.kafka.schemaregistry.security.authorizer.group.GroupAuthorizer
    
    # ACL для subject
    ResourceType: Subject
    ResourceName: payments-value
    Principal: User:analyst
    Operation: Read
    
    # Администратор может всё
    ResourceType: Subject
    ResourceName: *
    Principal: User:sr-admin
    Operation: All

### Совместимость и контроль

    # Конфигурация совместимости
    # BACKWARD — новая схема читается старым consumer
    # FORWARD — старая схема читается новым consumer
    # FULL — оба направления
    # NONE — нет проверки (опасно!)
    
    PUT /config/payments-value
    {
      "compatibility": "BACKWARD"
    }

### Глобальные настройки

    # Глобальная совместимость
    PUT /config
    {
      "compatibility": "BACKWARD"
    }
    
    # Запретить NONE без одобрения
    # Только администратор может менять на NONE

## Часть 5: Безопасность схем

### Валидация схем

    # Проверка схемы перед регистрацией
    # - Нет вложенных типов без ограничений
    # - Нет полей с default = null для sensitive данных
    # - Все строковые поля имеют maxLength
    
    {
      "type": "record",
      "name": "Payment",
      "fields": [
        {"name": "id", "type": "string", "maxLength": 36},
        {"name": "amount", "type": "long"},
        {"name": "card_mask", "type": "string", "maxLength": 16}
        // ❌ Неправильно: {"name": "card_number", "type": "string"}
        // ✅ Правильно: маска, не полный номер
      ]
    }

### Маскирование в схеме

    # Схема не должна содержать чувствительные данные в открытом виде
    # Использовать:
    # - Хеши вместо ID
    # - Маски вместо полных номеров карт
    # - Токены вместо SSN

### Versioning безопасности

    # При изменении схемы:
    # 1. Code review
    # 2. Проверка на чувствительные поля
    # 3. Проверка совместимости
    # 4. Тестирование на staging
    # 5. Только потом регистрация

## Часть 6: Мониторинг

### Что логировать

| Событие | Уровень | Описание |
|---------|---------|----------|
| Регистрация схемы | INFO | Кто, какую, какую версию |
| Удаление схемы | WARN | Критичная операция |
| Изменение совместимости | WARN | Влияет на все приложения |
| Ошибки валидации | WARN | Некорректная схема |
| Доступ к схеме | DEBUG | Кто запрашивал |

### Алерты

| Условие | Приоритет |
|---------|-----------|
| Удаление subject | Critical |
| Изменение совместимости на NONE | High |
| Регистрация схемы вне окна обслуживания | Warning |
| Ошибки аутентификации | Critical |

## Вывод

Безопасность Schema Registry:
1. **Аутентификация** — Basic, SASL, или mTLS
2. **Authorization** — Subject-level ACL
3. **Совместимость** — не NONE без причины
4. **Валидация** — нет чувствительных данных в схемах
5. **Мониторинг** — регистрация, удаление, изменения
6. **Процесс** — code review перед регистрацией

Schema Registry — врата данных. Защити врата.

---
_Статья создана на основе анализа материалов по безопасности Kafka_