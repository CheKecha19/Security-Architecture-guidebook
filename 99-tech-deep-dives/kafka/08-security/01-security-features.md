---
title: "Встроенные механизмы безопасности Apache Kafka"
track: "08-security"
article: "01"
topic: "kafka"
word-count: 6229
sources: 18
date: 2026-05-13
---

# Встроенные механизмы безопасности Apache Kafka

**TL;DR:** До версии 0.9 Kafka не имела встроенных средств безопасности — единственной защитой была сетевая изоляция. Сейчас платформа предоставляет полноценный стек: TLS-шифрование трафика, 5 механизмов SASL-аутентификации (PLAIN, SCRAM-SHA-256/512, GSSAPI/Kerberos, OAUTHBEARER), модель авторизации на основе ACL с подключаемым Authorizer, делегированные токены и интеграцию с enterprise IdM. Статья детально разбирает каждый механизм с трёх перспектив: инфраструктурный безопасник, DevSecOps-инженер и архитектор ИБ. Время чтения: ~35 минут.

---

## Содержание

- [1. Эволюция безопасности в Kafka](#1-эволюция-безопасности-в-kafka)
- [2. Listener Configuration — фундамент модели безопасности](#2-listener-configuration--фундамент-модели-безопасности)
- [3. Шифрование: TLS/SSL в Kafka](#3-шифрование-tls-в-kafka)
  - [3.1 Механика TLS в Kafka](#31-механика-tls-в-kafka)
  - [3.2 PKI-инфраструктура и управление сертификатами](#32-pki-инфраструктура-и-управление-сертификатами)
  - [3.3 Шифрование на диске (encryption at rest)](#33-шифрование-на-диске-encryption-at-rest)
- [4. Аутентификация: модель SASL](#4-аутентификация-модель-sasl)
  - [4.1 Архитектура JAAS в Kafka](#41-архитектура-jaas-в-kafka)
  - [4.2 SASL/PLAIN — простота с оговорками](#42-saslplain--простота-с-оговорками)
  - [4.3 SASL/SCRAM — безопасная альтернатива паролям](#43-saslscram--безопасная-альтернатива-паролям)
  - [4.4 SASL/GSSAPI (Kerberos) — enterprise-стандарт](#44-saslgssapi-kerberos--enterprise-стандарт)
  - [4.5 SASL/OAUTHBEARER — OAuth 2.0 для микросервисов](#45-sasloauthbearer--oauth-20-для-микросервисов)
  - [4.6 Делегированные токены (Delegation Tokens)](#46-делегированные-токены-delegation-tokens)
  - [4.7 Сравнительная матрица механизмов](#47-сравнительная-матрица-механизмов)
- [5. Авторизация: ACL и модель разрешений](#5-авторизация-acl-и-модель-разрешений)
  - [5.1 Архитектура Authorizer](#51-архитектура-authorizer)
  - [5.2 Структура ACL-правила](#52-структура-acl-правила)
  - [5.3 Типы ресурсов и операций](#53-типы-ресурсов-и-операций)
  - [5.4 Resource Patterns: literal, prefixed, wildcard](#54-resource-patterns-literal-prefixed-wildcard)
  - [5.5 Суперпользователи и поведение по умолчанию](#55-суперпользователи-и-поведение-по-умолчанию)
  - [5.6 Principal Forwarding в KRaft-кластере](#56-principal-forwarding-в-kraft-кластере)
- [6. Безопасность ZooKeeper и KRaft](#6-безопасность-zookeeper-и-kraft)
- [7. Три перспективы безопасности](#7-три-перспективы-безопасности)
  - [7.1 Инфраструктурный безопасник](#71-инфраструктурный-безопасник)
  - [7.2 DevSecOps-инженер](#72-devsecops-инженер)
  - [7.3 Архитектор информационной безопасности](#73-архитектор-информационной-безопасности)
- [8. Итоги](#8-итоги)
- [9. Источники](#9-источники)
- [10. Связанные статьи](#10-связанные-статьи)

---

## 1. Эволюция безопасности в Kafka

**Ключевой момент:** До версии 0.9.0.0 (декабрь 2015) Kafka была полностью открытой системой — любой, кто мог подключиться к порту брокера по TCP, имел полный доступ к данным. Единственным рубежом защиты была сетевая изоляция на уровне firewall.

Версия 0.9 стала переломным моментом. Сообщество добавило четыре ключевых механизма безопасности:

1. **Аутентификация клиентов** через Kerberos или TLS-сертификаты — теперь брокер знает, *кто* делает запрос
2. **Авторизация на основе ACL** — Unix-подобная модель разрешений: кто и к каким данным имеет доступ
3. **Шифрование сетевого трафика** через TLS — сообщения передаются безопасно даже через недоверенные сети
4. **Аутентификация broker↔ZooKeeper** — защита метаданных кластера

**Аналогия:** Представьте закрытый офисный центр. До версии 0.9 это было здание без охраны — любой мог войти, открыть любую дверь и взять документы. С 0.9 появились: турникеты с пропусками (аутентификация), замки на кабинетах (ACL) и зашифрованная внутренняя почта, которую нельзя прочитать, перехватив на этаже (TLS).

В последующих версиях модель расширялась:

| Версия | Добавленный механизм |
|--------|---------------------|
| 0.9.0 | Базовая аутентификация (TLS/Kerberos) + ACL |
| 0.10.2 | SASL/PLAIN и SASL/SCRAM |
| 1.0.0 | Поддержка нескольких listeners с разными протоколами |
| 1.1.0 | Делегированные токены (Delegation Tokens) |
| 2.0.0 | Подключаемые callback handler для SASL, SSL principal mapping |
| 2.1.0 | SASL/OAUTHBEARER (KIP-255) |
| 2.4.0 | KRaft-режим (ранний доступ), StandardAuthorizer для KRaft |
| 3.0.0 | Улучшенная поддержка OAuth 2.0, prefix ACL для ресурсов |
| 3.3.0 | KRaft в production-ready |
| 3.5.0+ | Principal forwarding в KRaft, улучшенное управление SCRAM через метаданные |

Начиная с Kafka 3.5+, рекомендуемая эталонная архитектура безопасности выглядит так:

```
Клиент ──[SASL_SSL]──> Брокер ──[SSL/SASL]──> Другие брокеры
                           │
                           └──[SASL]──> KRaft Controller (или ZooKeeper)
                           
Для KRaft: ACL хранятся в метаданных кластера (__cluster_metadata)
Для ZooKeeper: ACL хранятся в ZK (устаревающий подход)
```

---

## 2. Listener Configuration — фундамент модели безопасности

**Ключевой момент:** Безопасность Kafka начинается с конфигурации listeners. Каждый listener — это отдельная точка входа, которая может использовать свой протокол безопасности. Именно здесь определяется, *как* клиенты подключаются к кластеру.

Kafka поддерживает 4 протокола безопасности:

| Протокол | Аутентификация | Шифрование | Применение |
|----------|---------------|-----------|------------|
| `PLAINTEXT` | Нет | Нет | Только для изолированных dev-сред |
| `SSL` | TLS-сертификаты | TLS | Внутренние сети, где аутентификация через PKI |
| `SASL_PLAINTEXT` | SASL | Нет | Только доверенные сети (не рекомендуется) |
| `SASL_SSL` | SASL + TLS | TLS | **Стандарт для продакшена** |

Конфигурация listeners в `server.properties`:

```properties
# Три раздельных точки входа с разными протоколами
listeners=PLAINTEXT://192.168.1.10:9092,SSL://192.168.1.10:9093,SASL_SSL://192.168.1.10:9094

# Протокол для взаимодействия между брокерами
security.inter.broker.protocol=SSL

# Адреса, которые брокер анонсирует клиентам (если за NAT/балансировщиком)
advertised.listeners=PLAINTEXT://broker.example.com:9092,SSL://broker.example.com:9093,SASL_SSL://broker.example.com:9094
```

**Правило "разных сетей":** В production-среде рекомендуется разделять трафик:
- `SASL_SSL://:9094` — для внешних клиентов (строгая аутентификация + шифрование)
- `SSL://:9093` — для внутренних микросервисов (сертификаты PKI)
- `PLAINTEXT://127.0.0.1:9092` — ТОЛЬКО localhost для отладки (жёстко ограничен биндингом на loopback)

**Аналогия:** Listener — это дверь в здание с разными уровнями контроля. PLAINTEXT — открытая дверь без замка. SSL — дверь с электронным ключом (сертификатом). SASL_SSL — дверь с турникетом (SASL-аутентификация) и бронированным стеклом (TLS-шифрование).

---

## 3. Шифрование: TLS/SSL в Kafka

### 3.1 Механика TLS в Kafka

**Ключевой момент:** Несмотря на то, что технически протокол называется TLS (SSL устарел в 2015), Kafka (как и Java) по историческим причинам использует термин "SSL" в названиях конфигурационных параметров. Фактически используется TLS 1.2/1.3.

Kafka использует TLS на трёх уровнях:
1. **Клиент ↔ брокер** — шифрование данных от продюсеров и консьюмеров
2. **Брокер ↔ брокер** — шифрование inter-broker communication (репликация)
3. **Брокер ↔ ZooKeeper/KRaft Controller** — защита метаданных кластера

Базовая конфигурация TLS для брокера:

```properties
# server.properties

# Включаем SSL listener
listeners=SSL://0.0.0.0:9093

# Пути к keystore и truststore
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=${KAFKA_KEYSTORE_PASSWORD}     # Из переменных окружения/секретов!
ssl.key.password=${KAFKA_KEY_PASSWORD}
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=${KAFKA_TRUSTSTORE_PASSWORD}

# Требуем клиентскую аутентификацию по сертификату
ssl.client.auth=required    # Альтернативы: requested, none

# Минимальная версия протокола (отключаем уязвимые старые версии)
ssl.enabled.protocols=TLSv1.2,TLSv1.3

# Дополнительно: рекомендуемые cipher suites
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384

# Включаем проверку имени хоста в сертификате (защита от MITM)
ssl.endpoint.identification.algorithm=HTTPS
```

Параметр `ssl.endpoint.identification.algorithm=HTTPS` **критически важен**: он заставляет клиента проверять, что hostname в сертификате брокера совпадает с DNS-именем, к которому он подключается. Без этого (значение по умолчанию — пустая строка) Kafka уязвима к MITM-атакам, даже при использовании TLS.

Параметр `ssl.client.auth` определяет, обязан ли клиент предъявить свой сертификат:
- `required` — клиент ОБЯЗАН предъявить сертификат, иначе соединение отклоняется
- `requested` — брокер запрашивает сертификат, но не требует (mTLS опциональный)
- `none` — клиентская аутентификация не требуется (односторонний TLS)

### 3.2 PKI-инфраструктура и управление сертификатами

**Ключевой момент:** TLS-безопасность Kafka держится на трёх компонентах PKI: keystore (закрытый ключ + сертификат узла), truststore (доверенные CA-сертификаты) и процедуре подписи сертификатов через Certificate Authority (CA).

Процесс генерации keystore/truststore для брокера:

```bash
#!/bin/bash
# Генерация сертификатов для Kafka-брокера
PASSWORD=test1234
VALIDITY=365

# 1. Генерируем keystore с закрытым ключом брокера
keytool -keystore kafka.server.keystore.jks \
  -alias localhost -validity $VALIDITY -genkey -keyalg RSA

# 2. Создаём собственный CA (для тестов; в production — использовать корпоративный CA)
openssl req -new -x509 -keyout ca-key -out ca-cert -days $VALIDITY

# 3. Импортируем CA-сертификат в truststore брокера
keytool -keystore kafka.server.truststore.jks \
  -alias CARoot -import -file ca-cert

# 4. Импортируем CA-сертификат в truststore клиента (аналогично)
keytool -keystore kafka.client.truststore.jks \
  -alias CARoot -import -file ca-cert

# 5. Генерируем запрос на подпись сертификата (CSR)
keytool -keystore kafka.server.keystore.jks \
  -alias localhost -certreq -file cert-file

# 6. Подписываем сертификат брокера нашим CA
openssl x509 -req -CA ca-cert -CAkey ca-key \
  -in cert-file -out cert-signed \
  -days $VALIDITY -CAcreateserial

# 7. Импортируем подписанный сертификат обратно в keystore
keytool -keystore kafka.server.keystore.jks \
  -alias CARoot -import -file ca-cert
keytool -keystore kafka.server.keystore.jks \
  -alias localhost -import -file cert-signed
```

**Важное правило:** Common Name (CN) сертификата брокера должен совпадать с FQDN хоста. Если брокер доступен как `kafka-broker-1.example.com`, то CN сертификата должен быть `kafka-broker-1.example.com`. При несовпадении клиент с `ssl.endpoint.identification.algorithm=HTTPS` отвергнет соединение.

**Аналогия:** Keystore — это паспорт пользователя (удостоверяет личность). Truststore — список доверенных миграционных служб (CA), чьи паспорта считаются подлинными. Когда брокер предъявляет паспорт клиенту, клиент проверяет: выдан ли паспорт доверенной службой (есть ли CA в truststore) и совпадает ли имя в паспорте с тем, кем человек представился (CN check).

### 3.3 Шифрование на диске (encryption at rest)

**Ключевой момент:** Apache Kafka **не предоставляет встроенного шифрования данных на диске**. Это один из наиболее часто задаваемых вопросов: "зашифрованы ли данные в логах Kafka?". Ответ — нет, не по умолчанию.

Шифрование данных на диске в Kafka реализуется на уровне, лежащем ниже:

1. **Шифрование файловой системы/диска:**
   - LUKS (Linux Unified Key Setup) — прозрачное шифрование на уровне блочного устройства
   - BitLocker (Windows)
   - dm-crypt

2. **Шифрование на уровне облачного провайдера:**
   - AWS EBS encryption (для MSK)
   - GCP Persistent Disk encryption
   - Azure Managed Disk encryption

3. **Confluent Cloud:** предоставляет шифрование данных на диске по умолчанию, с опцией BYOK (Bring Your Own Key).

Существует предложение [KIP-317](https://cwiki.apache.org/confluence/display/KAFKA/KIP-317%3A+Add+end-to-end+data+encryption+functionality+to+Apache+Kafka) о добавлении end-to-end шифрования непосредственно в Kafka, но на 2026 год оно остаётся в статусе предложения.

---

## 4. Аутентификация: модель SASL

### 4.1 Архитектура JAAS в Kafka

**Ключевой момент:** Kafka использует Java Authentication and Authorization Service (JAAS) как основу для конфигурации SASL. JAAS-конфигурация может быть задана двумя способами: через статический файл или через параметр `sasl.jaas.config` непосредственно в конфигурации брокера/клиента.

Структура JAAS-конфигурации:

```
Секция {
    LoginModule required|requisite|sufficient|optional
    параметр1=значение1
    параметр2=значение2
    ...;
};
```

Для брокеров используется секция `KafkaServer`, для клиентов — `KafkaClient`. Если в брокере настроено несколько listener'ов с SASL, можно использовать префиксные секции, например `sasl_ssl.KafkaServer`.

Современный подход (Kafka 2.0+) — использовать `sasl.jaas.config` вместо статического файла:

```properties
# Для брокера: в server.properties
listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="${SCRAM_ADMIN_PASSWORD}";

# Для клиента: в client.properties
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="${CLIENT_USER}" \
  password="${CLIENT_PASSWORD}";
```

Приоритет JAAS-конфигурации (от высшего к низшему):
1. `listener.name.{listenerName}.{saslMechanism}.sasl.jaas.config` — самый специфичный
2. `{listenerName}.KafkaServer` секция в статическом файле
3. `KafkaServer` секция в статическом файле — самый общий

**Аналогия:** JAAS — это бланк анкеты, который заполняет каждый посетитель при входе в здание. Разные LoginModule'ы — это разные способы проверки личности: одни принимают паспорт (Kerberos), другие — водительские права (SCRAM), третьи — электронный пропуск (OAUTHBEARER).

### 4.2 SASL/PLAIN — простота с оговорками

**Ключевой момент:** SASL/PLAIN — самый простой механизм: логин и пароль передаются открытым текстом. **Категорически нельзя использовать без TLS** (т.е. только как `SASL_SSL`, никогда `SASL_PLAINTEXT`).

Конфигурация брокера для SASL/PLAIN:

```properties
# server.properties
listeners=SASL_SSL://0.0.0.0:9094
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

# Определяем пользователей — имена и пароли в JAAS
listener.name.sasl_ssl.plain.sasl.jaas.config=\
  org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret" \
  user_admin="admin-secret" \
  user_alice="alice-secret" \
  user_producer_app="producer-app-secret";
```

Формат: `user_{имя_пользователя}="пароль"`. Параметры `username` и `password` задают учётные данные, которые брокер использует для inter-broker communication.

Конфигурация клиента:

```properties
# client.properties
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="alice" \
  password="alice-secret";
```

**Проблемы SASL/PLAIN:**
- Пароли хранятся в конфигурационных файлах на диске (даже если это зашифрованный `server.properties`)
- Пароли передаются по сети (хотя и внутри TLS-туннеля)
- Нет защиты от replay-атак на уровне протокола
- Для каждого пользователя нужно перезагружать конфигурацию брокера

**Решение для продакшена (Kafka 2.0+):** Подключаемые callback-обработчики, которые получают пароли из внешнего источника (LDAP, HashiCorp Vault, БД):

```properties
# Замена хранения паролей в JAAS на callback handler
sasl.server.callback.handler.class=com.example.CustomPlainServerCallbackHandler
sasl.client.callback.handler.class=com.example.CustomPlainClientCallbackHandler
```

### 4.3 SASL/SCRAM — безопасная альтернатива паролям

**Ключевой момент:** SCRAM (Salted Challenge Response Authentication Mechanism, RFC 5802) решает главную проблему PLAIN: пароль никогда не передаётся по сети. Вместо этого используется challenge-response протокол с солёным хешированием. Kafka поддерживает SCRAM-SHA-256 и SCRAM-SHA-512.

**Как работает SCRAM:**
1. Клиент отправляет имя пользователя и client nonce
2. Сервер отвечает: соль (salt), количество итераций (iteration count), server nonce и комбинированный nonce
3. Клиент вычисляет proof, используя пароль, соль и итерации — доказывает, что знает пароль
4. Сервер проверяет proof, вычисляет server signature и отправляет клиенту
5. Клиент проверяет server signature — подтверждает, что сервер тоже знает пароль (взаимная аутентификация!)

**Аналогия:** Обычная аутентификация (PLAIN) — как показать охраннику паспорт. SCRAM — как игра "камень-ножницы-бумага" с математическим доказательством: вы доказываете, что знаете секретную комбинацию, не показывая её.

Создание SCRAM-учётных данных:

```bash
# При инициализации кластера (до запуска брокеров):
# Создаём initial credentials для inter-broker communication
kafka-storage.sh format \
  -t $(kafka-storage.sh random-uuid) \
  -c config/server.properties \
  --add-scram 'SCRAM-SHA-256=[name="admin",password="admin-secret"]' \
  --add-scram 'SCRAM-SHA-512=[name="admin",password="admin-secret"]'

# После запуска — динамическое управление через kafka-configs.sh:
# Создать пользователя
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=alice-secret]' \
  --entity-type users --entity-name alice \
  --command-config client.properties

# Просмотреть учётные данные пользователя (пароль НЕ показывается)
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --entity-type users --entity-name alice \
  --command-config client.properties

# Удалить учётные данные
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --delete-config 'SCRAM-SHA-256' \
  --entity-type users --entity-name alice \
  --command-config client.properties
```

Что хранится в метаданных кластера (не на диске!):
- **Соль (salt)** — случайная строка
- **Количество итераций (iterations)** — по умолчанию 4096 (рекомендуется 8192+)
- **StoredKey** = H(H(password, salt, iterations)) — хеш от хеша
- **ServerKey** = HMAC(H(password), "Server Key") — для подписи сервера

Сам пароль в метаданных НЕ хранится. Даже имея доступ к метаданным кластера, злоумышленник не сможет восстановить исходный пароль (в отличие от PLAIN, где пароли лежат в `server.properties`).

Конфигурация брокера:

```properties
# server.properties
listeners=SASL_SSL://0.0.0.0:9094
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512

listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-secret";
```

Конфигурация клиента:

```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="alice" \
  password="alice-secret";
```

**SCRAM — рекомендуемый механизм по умолчанию для продакшена**, когда нет корпоративного Kerberos. Он обеспечивает безопасную аутентификацию без внешней инфраструктуры, а учётные данные надёжно хранятся в метаданных кластера.

### 4.4 SASL/GSSAPI (Kerberos) — enterprise-стандарт

**Ключевой момент:** GSSAPI — реализация Kerberos-аутентификации в Kafka. Это стандарт де-факто для крупных организаций, уже использующих Active Directory или MIT Kerberos. Kerberos обеспечивает единый вход (SSO) и централизованное управление учётными записями.

Архитектура Kerberos включает три стороны:
- **KDC (Key Distribution Center)** — центр выдачи билетов (Kerberos-сервер)
- **Service principal** — учётная запись сервиса Kafka (kafka/hostname@REALM)
- **User principal** — учётная запись пользователя/клиента

Принцип работы:
1. Клиент получает TGT (Ticket Granting Ticket) от KDC
2. Клиент запрашивает service ticket для Kafka у KDC, предъявляя TGT
3. Клиент предъявляет service ticket брокеру Kafka
4. Брокер проверяет ticket (расшифровывает своим keytab)
5. Обе стороны аутентифицированы без передачи пароля

**Аналогия:** Kerberos — это система "единого билета" в парке аттракционов. Вы один раз покупаете браслет (TGT) на входе. Для каждого аттракциона (сервиса) кассир выдаёт вам билетик (service ticket) по предъявлению браслета. Аттракцион проверяет билетик и пропускает вас. Браслет и билетики имеют ограниченный срок действия.

Конфигурация брокера:

```properties
# server.properties
listeners=SASL_PLAINTEXT://0.0.0.0:9094
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanisms=GSSAPI

# Имя сервиса — берётся из первой части principal (kafka в kafka/hostname@REALM)
sasl.kerberos.service.name=kafka

# Правила маппинга Kerberos principal → короткое имя (для ACL)
sasl.kerberos.principal.to.local.rules=RULE:[1:$1@$0](.*@EXAMPLE.COM)s/@.*//,DEFAULT
```

JAAS-конфигурация брокера:

```
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka_server.keytab"
    principal="kafka/kafka1.hostname.com@EXAMPLE.COM";
};

// Аутентификация брокера в ZooKeeper
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka_server.keytab"
    principal="kafka/kafka1.hostname.com@EXAMPLE.COM";
};
```

Конфигурация клиента:

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=GSSAPI
sasl.kerberos.service.name=kafka

# Для долгоживущих процессов — keytab
sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
  useKeyTab=true \
  storeKey=true \
  keyTab="/etc/security/keytabs/kafka_client.keytab" \
  principal="kafka-client-1@EXAMPLE.COM";

# Для CLI-утилит — ticket cache (после kinit)
sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
  useTicketCache=true;
```

Создание Kerberos principal'ов:

```bash
# Для каждого брокера
sudo kadmin.local -q 'addprinc -randkey kafka/{hostname}@{REALM}'
sudo kadmin.local -q "ktadd -k /etc/security/keytabs/{keytabname}.keytab kafka/{hostname}@{REALM}"

# Для клиентов
sudo kadmin.local -q 'addprinc kafka-client-1@{REALM}'
```

**Особенности и подводные камни Kerberos:**
- Все хосты должны разрешаться по FQDN — иначе аутентификация не работает
- Keytab-файлы должны быть доступны на чтение пользователю, от которого запускается Kafka
- Требуется синхронизация времени между всеми узлами (NTP) — Kerberos-билеты имеют временные метки
- Каждый брокер должен иметь свой уникальный keytab (свой principal)

Когда Kerberos оправдан:
- Организация уже использует Active Directory / MIT Kerberos
- Нужен единый вход (SSO) для всех сервисов
- Требуется строгий контроль жизненного цикла учётных записей
- Соответствие требованиям регуляторов (PCI DSS, 152-ФЗ)

### 4.5 SASL/OAUTHBEARER — OAuth 2.0 для микросервисов

**Ключевой момент:** OAUTHBEARER (KIP-255) интегрирует Kafka с экосистемой OAuth 2.0 / OpenID Connect. Клиент получает JWT-токен от OAuth-сервера (Keycloak, Okta, Azure AD) и предъявляет его брокеру. Это идеальный выбор для микросервисной архитектуры и cloud-native окружения.

Принцип работы:
1. Клиент получает JWT access token от Authorization Server (OAuth 2.0 flow)
2. Клиент подключается к брокеру по SASL/OAUTHBEARER и передаёт токен
3. Брокер валидирует токен: проверяет подпись, срок действия, issuer, audience
4. Брокер извлекает principal из claim'ов токена (обычно `sub`)
5. Principal используется для ACL-проверок

**Аналогия:** OAUTHBEARER — это электронный пропуск с QR-кодом. Охранник (брокер) сканирует QR-код, проверяет подлинность пропуска у центрального сервера и срок его действия, извлекает из него имя сотрудника и уровень доступа. Сам пропуск можно за несколько секунд отозвать централизованно.

Базовая JAAS-конфигурация (не-production/unsecured mode для тестов):

```
KafkaClient {
    org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required
    unsecuredLoginStringClaim_sub="alice";
};
```

Production-конфигурация с внешним OAuth-сервером (через Strimzi Kafka OAuth или Confluent):

```properties
# Брокер (server.properties)
listeners=SASL_SSL://0.0.0.0:9094
sasl.enabled.mechanisms=OAUTHBEARER
sasl.mechanism.inter.broker.protocol=OAUTHBEARER

# OAuth-валидация на стороне брокера
listener.name.sasl_ssl.oauthbearer.sasl.jaas.config=\
  org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  unsecuredLoginStringClaim_sub="admin";
listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class=\
  io.strimzi.kafka.oauth.server.JaasServerOauthValidatorCallbackHandler

# Клиент (client.properties)
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.token.endpoint.uri="https://auth.example.com/oauth2/token" \
  oauth.client.id="kafka-client" \
  oauth.client.secret="${CLIENT_SECRET}";
sasl.login.callback.handler.class=\
  io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
```

**Преимущества OAUTHBEARER:**
- Централизованное управление токенами (отзыв, expiry)
- Интеграция с корпоративными IdM (Keycloak, Okta, Azure AD)
- Стандартный для микросервисов подход
- Токены могут содержать дополнительную информацию (claims: roles, группы)
- Короткоживущие токены + refresh token ротация

**Strimzi Kafka OAuth** ([strimzi-kafka-oauth](https://github.com/strimzi/strimzi-kafka-oauth)) — наиболее зрелая open-source реализация OAuth 2.0 для Kafka, поддерживает:
- OAuth 2.0 over PLAIN (для обратной совместимости)
- JWT-валидация без обращения к OAuth-серверу (introspection или локальная проверка JWKS)
- Keycloak, Hydra, Azure AD и другие провайдеры

### 4.6 Делегированные токены (Delegation Tokens)

**Ключевой момент:** Механизм делегированных токенов (KIP-48) решает проблему распространения учётных данных Kerberos на множество клиентов. Вместо того чтобы каждый клиент имел свой Kerberos principal, администратор создаёт делегированный токен, который клиенты используют как временный "пропуск".

Сценарий: В кластере используется Kerberos-аутентификация. Вам нужно запустить 100 Spark executor'ов, которые будут читать из Kafka. Вместо 100 Kerberos-ключей вы создаёте один delegation token.

```bash
# Создание делегированного токена
kafka-delegation-tokens.sh --bootstrap-server localhost:9092 \
  --create --max-life-time-period -1 \
  --command-config client.properties \
  --renewer-principal User:admin

# Вывод:
# TokenID: ABCDEF...  HMAC: abcdef123456...
# Этот HMAC передаётся клиентам как пароль
```

Клиент использует токен через SASL/SCRAM:

```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="${TOKEN_ID}" \
  password="${TOKEN_HMAC}";
```

Токен можно продлевать (`--renew`) и отзывать (`--remove`). Для делегированных токенов существуют отдельные ACL-операции (`CreateTokens`, `DescribeTokens`).

### 4.7 Сравнительная матрица механизмов

| Характеристика | PLAIN | SCRAM | GSSAPI | OAUTHBEARER | Delegation Token |
|---------------|-------|-------|--------|-------------|-----------------|
| **Уровень безопасности** | Низкий | Высокий | Высокий | Высокий | Средний-высокий |
| **Передача пароля** | Открытый текст (внутри TLS) | Никогда (challenge-response) | Никогда (ticket-based) | Токен (внутри TLS) | Токен (внутри TLS) |
| **Внешняя инфраструктура** | Нет | Нет | KDC (Kerberos/AD) | OAuth/OIDC Server | Kerberos + SCRAM |
| **Динамическое управление** | ❌ Требуется перезагрузка | ✅ Через kafka-configs | ❌ Через KDC admin | ✅ Через IdM | ✅ Через CLI API |
| **Хранение секретов** | JAAS-файл на диске | Метаданные кластера (StoredKey) | Keytab-файл | IdM-сервер | Метаданные кластера |
| **Replay-защита** | ❌ | ✅ (nonce) | ✅ (timestamp + lifetime) | ✅ (exp + jti) | ✅ (expiry) |
| **Сложность настройки** | ★☆☆☆☆ | ★★☆☆☆ | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| **Типовой сценарий** | Dev / изолированные среды | Production без Kerberos | Enterprise с AD | Микросервисы / Cloud | Spark/Flink на Kerberos |
| **Рекомендация** | ⚠️ Только dev | ✅ Основной для production | ✅ Enterprise-стандарт | ✅ Cloud-native | Специфический |

---

## 5. Авторизация: ACL и модель разрешений

### 5.1 Архитектура Authorizer

**Ключевой момент:** Kafka использует подключаемую (pluggable) модель авторизации. Реализация Authorizer указывается через `authorizer.class.name` в `server.properties` и должна расширять `org.apache.kafka.server.authorizer.Authorizer`.

Два варианта Authorizer:

1. **Для KRaft-кластеров** (современный подход, рекомендуется):
   ```properties
   authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
   ```
   ACL хранятся непосредственно в KRaft metadata log (`@metadata` топик). Не требуется ZooKeeper.

2. **Для ZooKeeper-кластеров** (устаревающий):
   ```properties
   authorizer.class.name=kafka.security.authorizer.AclAuthorizer
   ```
   ACL хранятся в ZooKeeper.

В этой статье мы фокусируемся на KRaft-подходе, так как Kafka 4.0+ полностью отказывается от ZooKeeper.

### 5.2 Структура ACL-правила

**Ключевой момент:** ACL-правило в Kafka имеет формат:

> **Principal {P} [Allowed|Denied] Operation {O} From Host {H} on Resource {R} matching ResourcePattern {RP}**

Разбор компонентов:

| Компонент | Возможные значения | Пример |
|-----------|-------------------|--------|
| **Principal P** | `User:имя` (по умолчанию) | `User:Bob`, `User:alice` |
| **Разрешение** | `Allow` или `Deny` | `Allow` |
| **Операция O** | Read, Write, Create, Delete, Alter, Describe, ClusterAction, DescribeConfigs, AlterConfigs, IdempotentWrite, CreateTokens, DescribeTokens, All | `Read`, `Write` |
| **Хост H** | IP-адрес или `*` (все хосты). ТОЛЬКО IP, hostnames не поддерживаются | `198.51.100.0` |
| **Ресурс R** | Topic, Group, Cluster, TransactionalId, DelegationToken, User | `Topic:orders` |
| **ResourcePattern RP** | literal, prefixed, wildcard | `literal` |

**Аналогия:** ACL в Kafka — это табличка на двери кабинета: "Сотрудник Алиса (Principal) из отдела продаж (Host=IP) ИМЕЕТ ПРАВО (Allow) ЧИТАТЬ (Read) ДОКУМЕНТЫ (Topic) с грифом 'финансы' (Resource)".

**Правило Deny имеет приоритет над Allow.** Если для Principal определён и Allow, и Deny на одну операцию — Deny побеждает.

### 5.3 Типы ресурсов и операций

**Ресурсы Kafka, для которых можно задавать ACL:**

| Ресурс | Описание | Код ошибки при отказе |
|--------|----------|----------------------|
| **Topic** | Топик. Запись/чтение/управление топиками | `TOPIC_AUTHORIZATION_FAILED (29)` |
| **Group** | Consumer group. Join, sync, heartbeat | `GROUP_AUTHORIZATION_FAILED (30)` |
| **Cluster** | Кластер в целом. Controlled shutdown, cluster actions | `CLUSTER_AUTHORIZATION_FAILED (31)` |
| **TransactionalId** | Транзакции. Commit, abort | `TRANSACTIONAL_ID_AUTHORIZATION_FAILED (53)` |
| **DelegationToken** | Делегированные токены. Создание, описание | Ошибка авторизации |
| **User** | Операции CreateTokens и DescribeTokens для других пользователей | Ошибка авторизации |

**Операции (Operations):**

| Операция | Применяется к | Что разрешает |
|----------|--------------|---------------|
| `Read` | Topic, Group | Чтение сообщений, чтение consumer group |
| `Write` | Topic | Запись сообщений в топик |
| `Create` | Topic, Cluster | Создание топиков |
| `Delete` | Topic | Удаление топиков |
| `Alter` | Topic | Изменение конфигурации топика, количества партиций |
| `Describe` | Topic, Group, Cluster, TransactionalId | Получение метаданных |
| `ClusterAction` | Cluster | Управление кластером (leader election, reassignment) |
| `DescribeConfigs` | Topic, Broker | Чтение конфигурации |
| `AlterConfigs` | Topic, Broker | Изменение конфигурации |
| `IdempotentWrite` | Cluster | Идемпотентная запись (для exactly-once продюсеров) |
| `CreateTokens` | DelegationToken, User | Создание делегированных токенов |
| `DescribeTokens` | DelegationToken, User | Просмотр информации о токенах |
| `All` | Любой | Все операции (суперпользователь) |

**Матрица "операция × ресурс" — минимально необходимые права:**

| Роль | Topic | Group | TransactionalId | Cluster |
|------|-------|-------|-----------------|---------|
| **Продюсер** | Write + Describe + Create | — | Write + Describe (если транзакции) | IdempotentWrite (если идемпотентный) |
| **Консьюмер** | Read + Describe | Read | — | — |
| **Администратор** | All | All | All | ClusterAction + DescribeConfigs + AlterConfigs |

### 5.4 Resource Patterns: literal, prefixed, wildcard

**Ключевой момент:** Kafka поддерживает три типа шаблонов для имён ресурсов, что позволяет создавать гибкие политики без перечисления каждого топика.

| Pattern Type | Поведение | Пример имени | Синтаксис CLI |
|-------------|-----------|-------------|--------------|
| **literal** (по умолчанию) | Точное совпадение имени | `Test-topic` | `--topic Test-topic` |
| **prefixed** | Совпадение по префиксу | `Test-` (все топики, начинающиеся с Test-) | `--topic Test- --resource-pattern-type prefixed` |
| **wildcard** | Любое имя (`*`) | `*` (все топики) | `--topic '*'` |

CLI-команды для управления ACL:

```bash
# ДОБАВЛЕНИЕ ACL

# Разрешить Bob'у читать и писать в Test-topic с двух IP
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Bob \
  --allow-host 198.51.100.0 --allow-host 198.51.100.1 \
  --operation Read --operation Write \
  --topic Test-topic

# Разрешить Peter'у продюсировать в ЛЮБОЙ топик (wildcard)
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Peter \
  --allow-host 198.51.200.1 \
  --producer --topic '*'

# Разрешить Jane продюсировать во все топики с префиксом "Test-"
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Jane \
  --producer --topic Test- \
  --resource-pattern-type prefixed

# Удобные ролевые флаги (генерируют набор ACL автоматически)
# --producer даёт: Write, Describe, Create на топик
# --consumer даёт: Read, Describe на топик + Read на consumer group
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Bob \
  --producer --topic Test-topic

kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:Alice \
  --consumer --topic Test-topic --group Group-1

# Deny-правила (имеют приоритет над Allow)
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal 'User:*' --allow-host '*' \
  --deny-principal User:BadBob --deny-host 198.51.100.3 \
  --operation Read --topic Test-topic

# УДАЛЕНИЕ ACL
kafka-acls.sh --bootstrap-server localhost:9092 --remove \
  --allow-principal User:Bob \
  --operation Read --operation Write \
  --topic Test-topic

# ПРОСМОТР ACL
# Все ACL для конкретного ресурса
kafka-acls.sh --bootstrap-server localhost:9092 --list --topic Test-topic

# Все ACL, влияющие на Test-topic (включая wildcard и prefixed)
kafka-acls.sh --bootstrap-server localhost:9092 --list \
  --topic Test-topic --resource-pattern-type match
```

### 5.5 Суперпользователи и поведение по умолчанию

**Ключевой момент:** По умолчанию, если для ресурса не определено ни одного ACL, доступ к нему ЗАПРЕЩЁН для всех, кроме суперпользователей. Это поведение "deny-by-default" — важный принцип нулевого доверия.

Суперпользователи задаются в `server.properties`:

```properties
# Список суперпользователей через точку с запятой (не запятую!)
super.users=User:admin;User:operator
```

Суперпользователям разрешены ВСЕ операции на ВСЕХ ресурсах, независимо от ACL.

**Изменение поведения по умолчанию:**

Если вам нужно, чтобы ресурсы без ACL были доступны всем (менее безопасно, но удобно для быстрого старта):

```properties
allow.everyone.if.no.acl.found=true
```

При этой настройке: если у ресурса нет ACL — доступ открыт всем. Если есть хотя бы один ACL — применяются только ACL-правила (открытый доступ отключается).

**Аналогия:** Поведение по умолчанию — это офис, где все двери закрыты, кроме обозначенных табличками. `allow.everyone.if.no.acl.found=true` — офис, где двери без табличек открыты, но как только повесили табличку — дверь работает по правилам на ней.

### 5.6 Principal Forwarding в KRaft-кластере

**Ключевой момент:** В KRaft-кластере административные запросы (CreateTopics, DeleteTopics) клиент отправляет на брокер, а брокер перенаправляет их на активный контроллер. При этом важно, чтобы авторизация выполнялась от имени *исходного клиента*, а не брокера-посредника. Для этого используется механизм **Principal Forwarding**.

Процесс:
1. Клиент отправляет `CreateTopics` на брокер
2. Брокер упаковывает запрос в `Envelope`-запрос, добавляя client principal
3. Envelope-запрос отправляется на контроллер
4. Контроллер авторизует:
   - Envelope-запрос — от имени брокера (проверяет, что брокер имеет право пересылать)
   - Вложенный запрос — от имени КЛИЕНТА (проверяет, что клиент имеет право CreateTopics)

Для этого Kafka должна уметь сериализовать/десериализовать principal. Если используется кастомный `principal.builder.class`, он должен реализовывать интерфейс `org.apache.kafka.common.security.auth.KafkaPrincipalSerde`.

Встроенный `DefaultKafkaPrincipalBuilder` использует формат `DefaultPrincipalData.json` из исходного кода Kafka и работает без дополнительной настройки.

---

## 6. Безопасность ZooKeeper и KRaft

**Ключевой момент:** Метаданные кластера (список топиков, партиций, брокеров, конфигурация, ACL) критически важны. Компрометация хранилища метаданных = компрометация всего кластера.

### ZooKeeper (устаревающий подход)

В ZooKeeper-based кластерах аутентификация broker↔ZK настраивается через SASL/Kerberos:

```properties
# zookeeper.properties
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
```

JAAS-конфигурация для ZooKeeper:

```
Server {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/path/to/server/keytab"
    storeKey=true
    useTicketCache=false
    principal="zookeeper/yourzkhostname";
};
```

При `zookeeper.set.acl=true` в брокере метаданные в ZK защищаются: только брокеры могут их модифицировать, но znodes остаются world-readable. Логика: данные в ZK не чувствительные, но некорректная модификация может нарушить работу кластера.

### KRaft (современный подход)

В KRaft-кластере все метаданные, включая ACL, хранятся в реплицированном логе метаданных (`@metadata` partition). Безопасность обеспечивается теми же механизмами, что и для обычных топиков:

- **Inter-broker/controller TLS** для шифрования трафика между контроллерами и брокерами
- **SASL-аутентификация** для доступа к контроллеру
- **ACL на кластерном уровне** для административных операций

В KRaft-кластере ACL управляются полностью через брокер, без отдельного ZooKeeper-соединения. Для kafka-acls.sh используется `--bootstrap-server` (или `--bootstrap-controller` для прямого доступа к контроллеру):

```bash
# Через брокер
kafka-acls.sh --bootstrap-server localhost:9092 --list

# Напрямую через контроллер (если нужно)
kafka-acls.sh --bootstrap-controller localhost:9093 --list
```

KRaft устраняет целый класс проблем безопасности, связанных с ZooKeeper: не нужно защищать отдельный сервис, не нужно управлять отдельной PKI-инфраструктурой для ZK, нет риска несанкционированного доступа через ZK-порт.

---

## 7. Три перспективы безопасности

### 7.1 Инфраструктурный безопасник

**Что я должен знать о сетевых портах и протоколах Kafka?**

Kafka — это TCP-сервис, работающий на прикладном уровне (Layer 7 OSI). Типовые порты:

| Порт | Назначение | Протокол | Интерфейс |
|------|-----------|----------|-----------|
| 9092 | PLAINTEXT listener | TCP | Внутренний / loopback |
| 9093 | SSL listener | TCP (TLS 1.2+) | Внутренняя сеть |
| 9094 | SASL_SSL listener | TCP (TLS 1.2+) | Внешний / DMZ |
| 9095 | KRaft controller | TCP (TLS) | Только контроллеры + брокеры |
| 2181 | ZooKeeper (устаревший) | TCP | Только брокеры |

**Сетевая сегментация:**

```
[Internet / External]
        │
   [DMZ Network]
        │
   [Load Balancer / Reverse Proxy]
        │
   [Kafka SASL_SSL :9094]  ← внешний доступ только через SASL_SSL
        │
   [Internal Network]
        │
   [Kafka SSL :9093]  ← внутренние микросервисы
   [Kafka PLAINTEXT :9092 | bind 127.0.0.1]  ← только localhost
```

**Правила файрвола (минимально необходимое):**

```bash
# IPTABLES — разрешаем ТОЛЬКО то, что нужно

# Продюсеры/консьюмеры (внешние) → SASL_SSL порт
iptables -A INPUT -p tcp --dport 9094 -s 10.0.0.0/8 -j ACCEPT

# Внутренние микросервисы → SSL порт
iptables -A INPUT -p tcp --dport 9093 -s 192.168.0.0/16 -j ACCEPT

# Inter-broker communication (TLS)
iptables -A INPUT -p tcp --dport 9093 -s 192.168.0.0/16 -j ACCEPT

# KRaft контроллер — ТОЛЬКО с IP брокеров и контроллеров
iptables -A INPUT -p tcp --dport 9095 -s 192.168.10.0/24 -j ACCEPT

# ВСЁ ОСТАЛЬНОЕ — DROP
iptables -P INPUT DROP
```

**PKI-инфраструктура (мое хозяйство):**

Как инфраструктурный безопасник, я отвечаю за:
- Управление корневым и промежуточными CA
- Процедуру выпуска и отзыва сертификатов (CRL/OCSP)
- Ротацию сертификатов до истечения срока
- Защиту приватных ключей (HSM или защищённое хранилище)
- Контроль доступа к keystore/truststore файлам (chmod 600, владелец: kafka)

**⚠️ Критически важно:** Проверить, что `ssl.endpoint.identification.algorithm=HTTPS` установлен на всех брокерах и клиентах. Это предотвращает MITM-атаки, когда злоумышленник подменяет брокер своим сертификатом.

### 7.2 DevSecOps-инженер

**Что я должен встроить в CI/CD пайплайн?**

**1. SAST-сканирование конфигураций Kafka в репозитории:**

```yaml
# .github/workflows/kafka-security-scan.yml
name: Kafka Security Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Проверка server.properties на небезопасные настройки
      - name: Check Kafka Config Security
        run: |
          # Запрещаем PLAINTEXT для продакшена (кроме 127.0.0.1)
          if grep -r "PLAINTEXT" kafka-configs/production/; then
            echo "❌ PLAINTEXT listener в production-конфигурации!"
            exit 1
          fi
          
          # Проверяем наличие allow.everyone.if.no.acl.found=true
          if grep -r "allow.everyone.if.no.acl.found=true" kafka-configs/; then
            echo "⚠️ Найден allow.everyone.if.no.acl.found=true — пересмотрите политику"
          fi
          
          # Проверяем endpoint identification
          if ! grep -r "ssl.endpoint.identification.algorithm=HTTPS" kafka-configs/; then
            echo "❌ Отсутствует ssl.endpoint.identification.algorithm=HTTPS"
            exit 1
          fi
          
          # Запрещаем хардкод паролей
          if grep -rP "(password|secret)\s*=\s*(?!\$\{)" kafka-configs/; then
            echo "❌ Найдены хардкод-пароли в конфигурации!"
            exit 1
          fi
```

**2. SCA (Software Composition Analysis) — проверка версии Kafka на CVE:**

```bash
# Trivy scan образов Kafka
trivy image confluentinc/cp-kafka:7.7.0

# OWASP Dependency-Check для Java-клиентов
mvn dependency-check:check
```

**3. Сканирование секретов (детект токенов, паролей в коммитах):**

```yaml
- name: Secret Detection
  uses: gitleaks/gitleaks-action@v2
  with:
    config-path: .gitleaks.toml
```

**4. Валидация Terraform/Helm для Kafka (IaC scanning):**

```bash
# Проверка Terraform-конфигурации Amazon MSK
terraform validate
checkov -d terraform/kafka/

# Проверка Helm-чарта Strimzi
helm lint kafka-helm/
kubeconform -summary kafka-manifests/
```

**5. Инжектирование секретов через HashiCorp Vault при деплое:**

```yaml
# Фрагмент деплой-пайплайна
- name: Inject Kafka Secrets from Vault
  run: |
    export KAFKA_KEYSTORE_PASSWORD=$(vault read -field=keystore_password secret/kafka/production)
    export KAFKA_TRUSTSTORE_PASSWORD=$(vault read -field=truststore_password secret/kafka/production)
    export SCRAM_ADMIN_PASSWORD=$(vault read -field=admin_password secret/kafka/scram)
    envsubst < kafka-configs/server.properties.template > kafka-configs/server.properties
```

**6. Policy as Code — OPA-политики для Kafka в Kubernetes:**

```rego
# policy.kafka.rego — запрещаем создание топиков без ACL в неймспейсе production
package kafka.security

deny[msg] {
    input.kind == "KafkaTopic"
    input.metadata.namespace == "production"
    not input.spec.acls
    msg := sprintf("Топик %v в production должен иметь ACL", [input.metadata.name])
}
```

### 7.3 Архитектор информационной безопасности

**STRIDE-модель угроз для кластера Kafka:**

| Категория | Угроза | Механизм защиты | Где реализовано |
|-----------|--------|----------------|-----------------|
| **S**poofing (подмена) | Фейковый брокер в кластере | mTLS (ssl.client.auth=required), проверка сертификата с CA | TLS + Principal проверка |
| **S**poofing (подмена) | Подмена клиента | SASL-аутентификация + ACL-проверка Principal | SASL + Authorizer |
| **T**ampering (модификация) | Изменение сообщений в транзите | TLS-шифрование + message integrity (HMAC in TLS) | TLS 1.2+ |
| **T**ampering (модификация) | Модификация логов на диске | Шифрование диска (LUKS), контроль доступа к файлам | OS-level |
| **R**epudiation (отказ) | Клиент отрицает отправку сообщения | Audit logging (включить authorizer logging), transactional ID | Kafka Authorizer logs |
| **I**nformation Disclosure (утечка) | Перехват трафика (sniffing) | TLS-шифрование всех каналов | TLS на всех listeners |
| **I**nformation Disclosure (утечка) | Чтение данных из логов брокера | Шифрование диска + ACL на данные + ограничение доступа к файлам | OS + Kafka ACL |
| **D**enial of Service | SYN flood, connection exhaustion | Rate limiting на network level, connection throttling | FW / reverse proxy |
| **D**enial of Service | Неограниченная запись в топик | Quota management (`quota.producer.byte-rate`), resource limits | Kafka Quotas |
| **E**levation of Privilege | Эскалация до super.user | Контроль доступа к server.properties, audit super.user действий | OS permissions + audit |

**Trust Boundaries (границы доверия):**

```
┌─────────────────────────────────────────────────────┐
│                UNTRUSTED ZONE                        │
│  [External Clients]  [Internet]  [3rd-party APIs]    │
└──────────────────────┬──────────────────────────────┘
                       │ TRUST BOUNDARY 1: SASL_SSL auth + TLS
┌──────────────────────┴──────────────────────────────┐
│                DMZ / EDGE ZONE                       │
│  [Kafka SASL_SSL :9094]                             │
└──────────────────────┬──────────────────────────────┘
                       │ TRUST BOUNDARY 2: Internal mTLS
┌──────────────────────┴──────────────────────────────┐
│              INTERNAL TRUSTED ZONE                   │
│  [Internal Microservices]  [Kafka SSL :9093]        │
│  [Kafka Brokers]  [KRaft Controllers]               │
└──────────────────────┬──────────────────────────────┘
                       │ TRUST BOUNDARY 3: OS-level hardening
┌──────────────────────┴──────────────────────────────┐
│              HOST / DATA LAYER                       │
│  [Disk Encryption]  [Log Files]  [Config Files]     │
└─────────────────────────────────────────────────────┘
```

**Классификация данных, проходящих через Kafka:**

| Категория данных | Требования | Механизм в Kafka |
|-----------------|-----------|-----------------|
| **Публичные (Public)** | Базовые | PLAINTEXT или SSL |
| **Внутренние (Internal)** | Шифрование в транзите | SSL/SASL_SSL |
| **Конфиденциальные (Confidential)** | Шифрование + аутентификация | SASL_SSL + ACL |
| **ПДн / Restricted** | Полное соответствие 152-ФЗ | SASL_SSL + ACL + disk encryption + audit logging |

**Compliance Mapping:**

| Регулятор / Стандарт | Требования к Kafka | Как обеспечить |
|---------------------|-------------------|----------------|
| **152-ФЗ (ПДн)** | Защита персональных данных, ограничение доступа, аудит | SASL_SSL + ACL + шифрование диска + audit logging + контроль доступа |
| **187-ФЗ (КИИ)** | Категорирование объектов, защита от компьютерных атак | STRIDE threat model + пентестинг + мониторинг безопасности + incident response |
| **GDPR** | Data subject rights, right to erasure, data retention | Retention policies в Kafka + процедура удаления данных по запросу (compaction + delete records) |
| **PCI DSS** | Защита данных карт (Req. 3, 4), контроль доступа (Req. 7), аудит (Req. 10) | TLS everywhere, strong SASL auth, ACL, audit logging, шифрование диска |
| **SOC 2** | Security, Availability, Confidentiality | Полный цикл: hardening, мониторинг, backup/DR, инцидент-менеджмент |
| **ISO 27001 (A.10 — A.18)** | Криптография, физическая безопасность, операции, коммуникации, контроль доступа, suppliers | Комбинация TLS + SASL + ACL + OS hardening + процедуры |

**Рекомендация архитектора:** Минимальный приемлемый уровень безопасности для продакшена — `SASL_SSL` с SCRAM-SHA-512 на ВСЕХ listener'ах, включая inter-broker communication. Это покрывает ~80% требований compliance "из коробки" (шифрование + аутентификация + авторизация). Остальное (disk encryption, audit trail) добавляется на уровне инфраструктуры.

---

## 8. Итоги

**Ключевые выводы:**

1. **Kafka 0.9 стала водоразделом** — до неё безопасности не было как класса, после — полноценный стек: TLS + SASL + ACL.

2. **Listener Configuration — первый рубеж.** Разделяйте трафик по протоколам: внешний `SASL_SSL`, внутренний `SSL`, отладка через `PLAINTEXT` только на loopback.

3. **TLS — обязателен.** Всегда используйте `ssl.endpoint.identification.algorithm=HTTPS`. Включайте `ssl.client.auth=required` для mTLS между брокерами. Минимальная версия протокола — TLS 1.2.

4. **Аутентификация — выбирайте механизм под контекст:**
   - **SCRAM-SHA-512** — универсальный выбор для продакшена без внешней инфраструктуры
   - **Kerberos/GSSAPI** — если организация уже использует Active Directory
   - **OAUTHBEARER** — для cloud-native микросервисов с OAuth 2.0 / OpenID Connect
   - **PLAIN** — только для разработки, всегда поверх TLS

5. **ACL работает по принципу deny-by-default** — без явных ACL доступ запрещён всем, кроме суперпользователей. Используйте prefixed patterns для масштабирования политик, не создавайте ACL на каждый топик вручную.

6. **Encryption at rest — не встроено в Kafka.** Обеспечивается на уровне ОС (LUKS, BitLocker) или облачного провайдера.

7. **KRaft упрощает безопасность** — устраняет отдельный ZooKeeper, ACL хранятся в метаданных кластера, Principal Forwarding обеспечивает корректную авторизацию делегированных запросов.

8. **Три перспективы — три зоны ответственности:**
   - Инфраструктурный безопасник: порты, сеть, PKI, сертификаты, firewall
   - DevSecOps: CI/CD-сканирование, секреты, IaC-валидация, policy enforcement
   - Архитектор ИБ: threat model, trust boundaries, compliance, классификация данных

**Что дальше:** В следующей статье трека — [02-hardening.md](./02-hardening.md) — мы разберём пошаговое hardening-руководство: от secure defaults до Docker/Kubernetes-окружения и CIS benchmarks.

---

## 9. Источники

### Официальная документация

1. [Apache Kafka Security Overview (3.5)](https://kafka.apache.org/35/security/security-overview/) — официальный обзор архитектуры безопасности Kafka
2. [Apache Kafka — Authorization and ACLs (4.1)](https://kafka.apache.org/41/security/authorization-and-acls/) — полное описание ACL-модели, CLI-команд, типов ресурсов и операций
3. [Apache Kafka — Authentication using SASL (4.1)](https://kafka.apache.org/41/security/authentication-using-sasl/) — детальная документация JAAS, GSSAPI, PLAIN, SCRAM, OAUTHBEARER
4. [Apache Kafka — Listener Configuration](https://kafka.apache.org/33/security/listener-configuration/) — протоколы безопасности, настройка listeners и inter-broker communication
5. [Apache Kafka KIP-11 — ACL Structure](https://cwiki.apache.org/confluence/x/XIUWAw) — KIP, определивший структуру ACL в Kafka
6. [Apache Kafka KIP-255 — OAuth Authentication via SASL/OAUTHBEARER](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=75968876) — KIP, добавивший поддержку OAuth 2.0 в Kafka
7. [Apache Kafka KIP-290 — Resource Patterns](https://cwiki.apache.org/confluence/x/QpvLB) — KIP, добавивший prefixed и wildcard resource patterns
8. [Apache Kafka KIP-317 — End-to-End Data Encryption](https://cwiki.apache.org/confluence/display/KAFKA/KIP-317%3A+Add+end-to-end+data+encryption+functionality+to+Apache+Kafka) — текущий статус предложения end-to-end шифрования
9. [Apache Kafka KIP-48 — Delegation Tokens](https://cwiki.apache.org/confluence/x/tfmnAw) — механизм делегированных токенов
10. [Apache Kafka KIP-373 — User-scoped Token Operations](https://cwiki.apache.org/confluence/x/cwOQBQ) — CreateTokens/DescribeTokens для пользовательских ресурсов

### Корпоративные блоги и технические материалы

11. [Confluent Blog — Apache Kafka Security 101: TLS, Kerberos, SASL, and Authorizer in Apache Kafka 0.9](https://www.confluent.io/blog/apache-kafka-security-authorization-authentication-encryption/) — основополагающая статья о безопасности Kafka 0.9 от Confluent (Morgan Stanley, Ismael Juma и др.) — генерация TLS-сертификатов, конфигурация JAAS, настройка Kerberos
12. [Confluent Developer — Kafka Security FAQs](https://developer.confluent.io/faq/apache-kafka/security/) — FAQ по end-to-end encryption, encryption at rest, SASL-конфигурации, ACL-настройке
13. [Confluent Developer — Kafka Authorization (Course)](https://developer.confluent.io/courses/security/authorization) — образовательный курс по авторизации и ACL

### Внешние технические ресурсы

14. [Confluent Documentation — Configuring SCRAM](https://docs.confluent.io/platform/7.3/kafka/authentication_sasl/authentication_sasl_scram.html) — документация по SCRAM SHA-256/512, credential management
15. [Strimzi Kafka OAuth](https://github.com/strimzi/strimzi-kafka-oauth) — open-source реализация OAuth 2.0 для Kafka, документация по OAUTHBEARER

### Дополнительные источники (просмотрены, контент дополняет основные)

16. [Meshiq — Essential Kafka Security Best Practices for 2024](https://www.meshiq.com/blog/essential-kafka-security-best-practices-for-2024/) — best practices, обзор механизмов безопасности
17. [Conduktor — Kafka ACLs and Authorization Patterns](https://www.conduktor.io/glossary/kafka-acls-and-authorization-patterns) — практические паттерны ACL-конфигурации
18. [Stack Overflow — Configure ACLs with SASL_SSL OAUTHBEARER in KRaft mode](https://stackoverflow.com/questions/79767876/how-to-configure-acls-with-sasl-ssl-oauthbearer-in-apache-kafka-kraft-mode-mul) — практический пример настройки OAUTHBEARER + ACL в KRaft

### Общепризнанные знания (Verified Knowledge)

- **Kafka: The Definitive Guide** (O'Reilly, 2nd edition) — глава 8: Security — архитектура безопасности, SASL-механизмы, ACL, шифрование
- **Designing Event-Driven Systems** (O'Reilly) — паттерны безопасности в event-driven архитектурах на базе Kafka
- **Kafka Summit talks:** "Securing Kafka at Scale" (2021-2024) — production experience крупных компаний

---

## 10. Связанные статьи

- [Базовая архитектура Kafka](../02-basics/02-how-it-works.md) — компоненты кластера, которые мы защищаем
- [Ключевые концепты Kafka](../02-basics/03-core-concepts.md) — продюсеры, консьюмеры, consumer groups, топики
- [Деплой и конфигурация Kafka](../04-software/02-implementation.md) — пошаговая установка, server.properties
- [Мониторинг Kafka](../07-operations/01-monitoring.md) — мониторинг событий безопасности, JMX-метрики
- [Troubleshooting Kafka](../07-operations/04-troubleshooting.md) — диагностика проблем аутентификации и авторизации
- [Hardening Kafka](./02-hardening.md) — следующая статья трека: пошаговое hardening-руководство
- [Attack Surface Kafka](./03-attack-surface.md) — анализ поверхности атаки, CVE, векторы атак
- [Admin Security Practices](./04-admin-security-practices.md) — безопасное администрирование, incident response
