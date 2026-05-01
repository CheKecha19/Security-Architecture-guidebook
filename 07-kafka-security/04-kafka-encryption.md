# Шифрование в Kafka: TLS и шифрование at-rest

## Часть 1: Зачем нужно шифрование?

Представь, что ты отправляешь важное письмо. Обычный конверт — почтальон может прочитать. Сейф — никто не прочитает по пути. Получатель имеет ключ — открывает.

**Шифрование в Kafka** защищает данные:
- **In-transit** — при передаче между producer → broker → consumer
- **At-rest** — при хранении на диске брокеров
- **In-use** — при обработке (менее применимо к Kafka, но важно для консьюмеров)

## Зачем нужно шифрование?

### Ситуация 1: Перехват трафика

Злоумышленник имеет доступ к сети дата-центра. Без шифрования — он видит все сообщения: номера карт, транзакции, пароли. С TLS — видит только зашифрованный поток байтов.

### Ситуация 2: Кража диска

Жёсткий диск с данными Kafka украден из дата-центра. Без at-rest encryption — данные читаемы. С encryption — бесполезны без ключа.

### Ситуация 3: Compliance

PCI DSS требует: данные карт должны быть зашифрованы при передаче и хранении. TLS + at-rest encryption закрывает это требование.

## Часть 2: TLS для Kafka

### Что защищает TLS?

| Аспект | Защита | Как |
|--------|--------|-----|
| Конфиденциальность | Нельзя прочитать | Шифрование симметричным ключом |
| Целостность | Нельзя изменить | MAC (Message Authentication Code) |
| Аутентификация сервера | Это настоящий брокер | Сертификат сервера |
| Аутентификация клиента (опционально) | Это настоящий клиент | Сертификат клиента (mTLS) |

### Версии TLS

| Версия | Статус | Рекомендация |
|--------|--------|--------------|
| SSL 2.0, 3.0 | ❌ Устарел | Запретить |
| TLS 1.0, 1.1 | ❌ Устарел | Запретить |
| TLS 1.2 | ✅ Приемлем | Минимум |
| TLS 1.3 | ✅ Рекомендуется | Использовать |

### Настройка TLS на брокере

    # server.properties
    # Включить SSL listener
    listeners=SSL://:9093
    
    # Keystore (сертификат брокера)
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.key.password=key-password
    
    # Truststore (доверенные CA)
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.truststore.password=truststore-password
    
    # Протоколы и шифры
    ssl.protocol=TLSv1.3
    ssl.enabled.protocols=TLSv1.2,TLSv1.3
    ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256
    
    # mTLS (взаимная аутентификация)
    ssl.client.auth=required  # Или requested для опциональной

### Создание keystore и truststore

    # 1. Создать keystore для брокера
    keytool -keystore kafka.server.keystore.jks \
        -alias localhost -validity 365 -genkey -keyalg RSA
    
    # 2. Создать CA (Certificate Authority)
    openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
    
    # 3. Подписать сертификат брокера
    keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file
    openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365
    keytool -keystore kafka.server.keystore.jks -alias localhost -import -file cert-signed
    
    # 4. Создать truststore с CA
    keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca-cert
    
    # 5. Экспортировать сертификат для клиентов
    keytool -keystore kafka.server.keystore.jks -alias localhost -export -file server-cert

### Настройка клиента

    # producer.properties / consumer.properties
    security.protocol=SSL
    ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
    ssl.truststore.password=truststore-password
    
    # Для mTLS:
    ssl.keystore.location=/var/private/ssl/kafka.client.keystore.jks
    ssl.keystore.password=keystore-password

### Проверка TLS

    # Проверить подключение
    openssl s_client -connect localhost:9093 -tls1_3
    
    # Проверить сертификат
    keytool -list -v -keystore kafka.server.keystore.jks

## Часть 3: SASL_SSL (аутентификация + шифрование)

Комбинация SASL для аутентификации и SSL для шифрования:

    # server.properties
    listeners=SASL_SSL://:9093
    security.inter.broker.protocol=SASL_SSL
    
    # SASL
    sasl.enabled.mechanisms=SCRAM-SHA-256
    sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
    
    # SSL (тот же keystore/truststore)
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.truststore.password=truststore-password

### Клиентская конфигурация SASL_SSL

    security.protocol=SASL_SSL
    sasl.mechanism=SCRAM-SHA-256
    sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="alice" password="alice-secret";
    
    ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
    ssl.truststore.password=truststore-password

## Часть 4: Шифрование at-rest

### Шифрование диска (OS-level)

| Метод | Описание | Когда использовать |
|-------|----------|-------------------|
| LUKS | Linux Unified Key Setup | Linux bare metal |
| dm-crypt | Device mapper encryption | Linux |
| BitLocker | Windows | Windows servers |
| FileVault | macOS | Mac (редко для серверов) |

    # LUKS для диска с данными Kafka
    cryptsetup luksFormat /dev/sdb1
    cryptsetup luksOpen /dev/sdb1 kafka-data
    mkfs.ext4 /dev/mapper/kafka-data
    mount /dev/mapper/kafka-data /var/lib/kafka

### Облачное шифрование

| Провайдер | Решение | Управление ключами |
|-----------|---------|-------------------|
| AWS | EBS encrypted volumes | AWS KMS |
| Azure | Storage Service Encryption | Azure Key Vault |
| GCP | Customer-managed encryption keys | Cloud KMS |

    # AWS EBS encrypted
    # При создании инстанса:
    # EBS volumes: Encrypted = Yes
    # KMS key: aws/ebs или customer-managed

### Шифрование логов Kafka

    # log.dirs указывает на зашифрованный раздел
    log.dirs=/var/lib/kafka
    
    # Убедиться, что раздел смонтирован и зашифрован
    mount | grep kafka
    # /dev/mapper/kafka-data on /var/lib/kafka type ext4 (rw)

## Часть 5: Шифрование backups

### Резервное копирование с шифрованием

    # Kafka backup через kafka-backup tool или MirrorMaker
    # Данные на приёмнике тоже должны быть зашифрованы
    
    # Или: tar + GPG
    tar czf - /var/lib/kafka | gpg --encrypt --recipient backup@example.com > kafka-backup.tar.gz.gpg
    
    # Расшифровка
    gpg --decrypt kafka-backup.tar.gz.gpg | tar xzf -

## Часть 6: Управление ключами

### Жизненный цикл ключей

| Этап | Действие | Период |
|------|----------|--------|
| Создание | Генерация ключа | При инициализации |
| Распространение | Доставка на брокеры | При добавлении брокера |
| Использование | Шифрование/дешифрование | Постоянно |
| Ротация | Смена ключа | 1-2 года |
| Отзыв | Аннулирование | При компрометации |
| Уничтожение | Физическое удаление | По политике |

### KMS (Key Management Service)

    # AWS KMS для управления ключами
    # Kafka использует ключ KMS для шифрования данных
    # Автоматическая ротация ключей
    
    # Azure Key Vault
    # GCP Cloud KMS

## Вывод

Шифрование в Kafka:
1. **TLS 1.3** — для передачи данных
2. **mTLS** — для взаимной аутентификации
3. **At-rest** — LUKS, облачное шифрование
4. **Backups** — шифрование при архивировании
5. **Управление ключами** — KMS, ротация

Правило: **шифруй всё: в пути и в покое.**

---
_Статья создана на основе анализа материалов по безопасности Kafka_