# Аутентификация в Kafka

## Часть 1: Зачем нужна аутентификация?

Представь ночной клуб. У входа стоит охранник. Каждый, кто входит, показывает документ. Охранник проверяет: это действительно тот человек, за кого он себя выдаёт? Без этого — в клуб войдёт кто угодно.

**Аутентификация в Kafka** — это именно такой «охранник у входа». Каждый клиент (producer, consumer, admin) должен доказать свою личность, прежде чем подключиться к брокеру.

## Зачем нужна аутентификация?

### Ситуация 1: Поддельный producer

Злоумышленник настраивает producer на адрес брокера и начинает писать мусор в топик `orders`. Без аутентификации — брокер примет любые данные. С аутентификацией — брокер скажет: «Кто ты? Покажи документ».

### Ситуация 2: Компрометация консьюмера

Консьюмер `analytics-app` читает топик `payments`. Если аутентификации нет — любой другой консьюмер может присоединиться и читать те же данные.

### Ситуация 3: Административный доступ

Команда `kafka-topics.sh --delete --topic payments` удаляет все данные. Без аутентификации — любой может это сделать. С аутентификацией — только `User:admin` с правами.

## Часть 2: Методы аутентификации

### PLAINTEXT / SSL (без SASL)

| Метод | Аутентификация | Шифрование | Когда использовать |
|-------|---------------|------------|------------------|
| PLAINTEXT | Нет | Нет | Только localhost dev |
| SSL (без client auth) | Нет | Да | Внутри доверенной сети |
| SSL + client auth (mTLS) | Сертификат | Да | Высокая безопасность |

### SASL (Simple Authentication and Security Layer)

SASL — фреймворк аутентификации. Kafka поддерживает несколько механизмов:

| Механизм | Описание | Безопасность | Когда использовать |
|----------|----------|--------------|-------------------|
| **GSSAPI** | Kerberos | Высокая | Enterprise с Active Directory |
| **PLAIN** | Логин/пароль в открытом виде | ❌ Низкая | Никогда в production |
| **SCRAM-SHA-256** | Challenge-response | ✅ Высокая | Рекомендуется |
| **SCRAM-SHA-512** | Challenge-response | ✅ Высокая | Ещё выше безопасность |
| **OAUTHBEARER** | OAuth 2.0 tokens | ✅ Высокая | Облачные среды |
| **Delegation Token** | Временные токены | ✅ Высокая | Длительные задачи |

## Часть 3: SCRAM-SHA-256 (рекомендуемый)

### Что такое SCRAM?

**SCRAM** (Salted Challenge Response Authentication Mechanism) — современный метод аутентификации.

**Как работает:**
1. Клиент отправляет логин
2. Сервер отправляет salt и nonce (случайные числа)
3. Клиент вычисляет proof на основе пароля, salt, nonce
4. Сервер проверяет proof, не получая пароль в открытом виде

**Преимущества:**
- Пароль никогда не передаётся по сети
- Защита от replay-атак (nonce)
- Salt уникален для каждого пользователя

### Настройка SCRAM в Kafka

    # server.properties
    # Включить SCRAM-SHA-256
    sasl.enabled.mechanisms=SCRAM-SHA-256
    sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
    
    # Listener с SASL_SSL
    listeners=SASL_SSL://:9093
    inter.broker.listener.name=SASL_SSL
    
    # JAAS конфигурация для брокера
    listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=\
        org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="broker" \
        password="broker-password";

### Создание пользователей SCRAM

    # Создать пользователя
    kafka-configs.sh --bootstrap-server localhost:9093 \
        --alter --add-config \
        'SCRAM-SHA-256=[iterations=8192,password=alice-secret]' \
        --entity-type users --entity-name alice
    
    # Проверить
    kafka-configs.sh --bootstrap-server localhost:9093 \
        --describe --entity-type users --entity-name alice
    
    # Список пользователей
    kafka-configs.sh --bootstrap-server localhost:9093 \
        --describe --entity-type users
    
    # Удалить пользователя
    kafka-configs.sh --bootstrap-server localhost:9093 \
        --alter --delete-config 'SCRAM-SHA-256' \
        --entity-type users --entity-name alice

### Клиентская конфигурация

    # producer.properties / consumer.properties
    security.protocol=SASL_SSL
    sasl.mechanism=SCRAM-SHA-256
    sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="alice" \
        password="alice-secret";
    
    # SSL
    ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
    ssl.truststore.password=truststore-password

## Часть 4: Kerberos (GSSAPI)

### Когда использовать

- Enterprise с Active Directory
- Windows/Linux интеграция
- Single Sign-On (SSO)

### Настройка

    # server.properties
    sasl.enabled.mechanisms=GSSAPI
    sasl.mechanism.inter.broker.protocol=GSSAPI
    
    # JAAS конфигурация для Kerberos
    listener.name.sasl_ssl.gssapi.sasl.jaas.config=\
        com.sun.security.auth.module.Krb5LoginModule required \
        useKeyTab=true \
        storeKey=true \
        keyTab="/etc/kafka/keytabs/kafka.keytab" \
        principal="kafka/kafka.example.com@EXAMPLE.COM";

### Клиент

    sasl.mechanism=GSSAPI
    sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
        useKeyTab=true \
        keyTab="/etc/kafka/keytabs/alice.keytab" \
        principal="alice@EXAMPLE.COM";

## Часть 5: OAuth 2.0 (OAUTHBEARER)

### Когда использовать

- Облачные среды (AWS, Azure, GCP)
- Интеграция с облачным IAM
- Delegated authentication

### Настройка (Confluent / custom)

    # server.properties
    sasl.enabled.mechanisms=OAUTHBEARER
    
    # Custom callback handler для валидации токена
    listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class=\
        com.example.kafka.OAuthValidator

### Клиент

    sasl.mechanism=OAUTHBEARER
    sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler
    sasl.oauthbearer.token.endpoint.url=https://auth-server/oauth/token
    sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
        clientId="kafka-client" \
        clientSecret="client-secret";

## Часть 6: mTLS (взаимная аутентификация)

### Когда использовать

- Высокая безопасность
- Микросервисы внутри кластера Kubernetes
- Нет необходимости в username/password

### Настройка сервера

    # server.properties
    listeners=SSL://:9093
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.truststore.password=truststore-password
    ssl.client.auth=required

### Настройка клиента

    security.protocol=SSL
    ssl.keystore.location=/var/private/ssl/kafka.client.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
    ssl.truststore.password=truststore-password

### Идентификация по сертификату

    # ssl.client.auth=required означает:
    # Брокер проверяет сертификат клиента
    # Identity берётся из CN (Common Name) сертификата
    # CN=alice -> Principal = User:alice

## Часть 7: Сравнение методов

| Метод | Аутентификация | Шифрование | Сложность | Масштаб |
|-------|---------------|------------|-----------|---------|
| PLAINTEXT | ❌ Нет | ❌ Нет | Просто | Любой |
| SSL | ❌ Нет | ✅ Да | Средне | Любой |
| SASL/PLAIN | ⚠️ Пароль открытым | По выбору | Просто | Малый |
| SASL/SCRAM | ✅ Challenge-response | По выбору | Средне | Средний |
| SASL/GSSAPI | ✅ Kerberos | По выбору | Сложно | Enterprise |
| SASL/OAUTH | ✅ OAuth 2.0 | По выбору | Сложно | Облако |
| mTLS | ✅ Сертификат | ✅ Да | Средне | Большой |

### Рекомендации по выбору

| Сценарий | Рекомендация |
|----------|-------------|
| Новый проект | SASL_SSL + SCRAM-SHA-256 |
| Enterprise с AD | SASL_SSL + GSSAPI |
| Облако (AWS/Azure) | SASL_SSL + OAUTHBEARER |
| Kubernetes внутри | mTLS |
| Высочайшая безопасность | mTLS + дополнительная SASL |

## Вывод

Аутентификация в Kafka:
1. **SCRAM-SHA-256** — рекомендуемый метод для большинства
2. **Kerberos** — для enterprise с Active Directory
3. **OAuth 2.0** — для облачных сред
4. **mTLS** — для высокой безопасности
5. **Никогда PLAINTEXT или SASL/PLAIN в production**

Без аутентификации — Kafka открыта для всех. Это как ночной клуб без охраны.

---
_Статья создана на основе анализа материалов по безопасности Kafka_