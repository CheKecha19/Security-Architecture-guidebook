# Шифрование в PostgreSQL

## Часть 1: Зачем нужно шифрование?

Представь, что ты отправляешь важное письмо. Обычный конверт — любой может вскрыть. Теперь представь сейф: взломать сложно, содержимое защищено.

**Шифрование в PostgreSQL** — это «сейф» для данных. Защищает:
- Данные в пути (по сети)
- Данные на диске
- Отдельные чувствительные колонки

## Зачем нужно шифрование?

### Ситуация 1: Перехват трафика

Приложение и база данных в разных дата-центрах. Трафик проходит через общую сеть. Без шифрования злоумышленник может перехватить запросы и увидеть данные.

### Ситуация 2: Кража диска

Физический сервер украли из дата-центра. Без шифрования диска данные доступны любому, кто подключит диск.

### Ситуация 3: Утечка персональных данных

База содержит номера социального страхования (SSN). Даже если злоумышленник получит доступ к базе, без шифрования колонки он увидит SSN.

## Часть 2: Шифрование соединений (SSL/TLS)

### Как работает SSL в PostgreSQL?

1. Клиент подключается к серверу
2. Сервер предъявляет сертификат
3. Клиент проверяет сертификат (CA, hostname)
4. Устанавливается зашифрованный канал
5. Все данные передаются через TLS

### Настройка сервера

    # postgresql.conf
    ssl = on
    ssl_cert_file = '/etc/ssl/certs/server.crt'
    ssl_key_file = '/etc/ssl/private/server.key'
    ssl_ca_file = '/etc/ssl/certs/ca.crt'
    
    # Протоколы
    ssl_min_protocol_version = 'TLSv1.2'
    ssl_max_protocol_version = 'TLSv1.3'
    
    # Шифры (только надёжные)
    ssl_ciphers = 'HIGH:!aNULL:!MD5'

### Проверка SSL

    -- Проверить, что SSL работает
    SHOW ssl;
    
    -- Посмотреть текущие SSL-подключения
    SELECT * FROM pg_stat_ssl;
    
    -- Информация о сертификате
    SELECT * FROM pg_stat_ssl WHERE pid = pg_backend_pid();

### Клиентские настройки

| sslmode | Описание | Безопасность |
|---------|----------|--------------|
| disable | Без SSL | ❌ Нет |
| allow | SSL, если доступен | ⚠️ Низкая |
| prefer | Предпочитать SSL | ⚠️ Средняя |
| require | Требовать SSL | ⚠️ Средняя (без проверки сертификата) |
| verify-ca | Проверить CA | ✅ Высокая |
| verify-full | Проверить CA и hostname | ✅ Максимальная |

**Рекомендация для production:** `sslmode=verify-full`

    # Пример connection string
    postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=ca.crt

## Часть 3: Прозрачное шифрование данных (TDE)

### Что такое TDE?

**Transparent Data Encryption** — шифрование всего диска с данными. PostgreSQL не знает о шифровании: это делает операционная система.

### Варианты TDE

| Способ | Уровень | Описание |
|--------|---------|----------|
| LUKS/dm-crypt | Диск | Шифрование всего раздела |
| EBS encryption | Облако | AWS EBS encrypted volumes |
| Azure Disk Encryption | Облако | Шифрование дисков Azure |
| GCP CMEK | Облако | Customer-managed encryption keys |

### LUKS/dm-crypt (Linux)

    # Создать зашифрованный раздел
    cryptsetup luksFormat /dev/sdb1
    
    # Открыть
    cryptsetup luksOpen /dev/sdb1 pgdata
    
    # Создать файловую систему
    mkfs.ext4 /dev/mapper/pgdata
    
    # Монтировать
    mount /dev/mapper/pgdata /var/lib/postgresql

### Облачное шифрование

| Провайдер | Решение | Ключи |
|-----------|---------|-------|
| AWS | EBS encrypted + KMS | AWS KMS |
| Azure | Disk Encryption + Key Vault | Azure Key Vault |
| GCP | CMEK | Cloud KMS |

**Преимущества TDE:**
- Прозрачно для приложения
- Защита от кражи диска
- Автоматическое управление ключами (в облаке)

**Недостатки:**
- Не защищает от атак через SQL
- Нет защиты на уровне колонок
- Ключи хранятся в памяти (если получить доступ к серверу — данные доступны)

## Часть 4: Шифрование на уровне колонок (pgcrypto)

### Расширение pgcrypto

    -- Установить расширение
    CREATE EXTENSION IF NOT EXISTS pgcrypto;
    
    -- Проверить версию
    SELECT * FROM pg_available_extensions WHERE name = 'pgcrypto';

### Симметричное шифрование

    -- Шифрование
    SELECT pgp_sym_encrypt('secret data', 'my-password');
    
    -- Дешифрование
    SELECT pgp_sym_decrypt(
        pgp_sym_encrypt('secret data', 'my-password'),
        'my-password'
    );
    
    -- Шифрование с солью (рекомендуется)
    SELECT pgp_sym_encrypt(
        'secret data',
        'my-password',
        'cipher-algo=aes256, compress-algo=2'
    );

### Асимметричное шифрование (PGP)

    -- Генерация ключей (в приложении или через GnuPG)
    -- Шифрование публичным ключом
    SELECT pgp_pub_encrypt('secret', dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----...'));
    
    -- Дешифрование приватным ключом
    SELECT pgp_pub_decrypt(encrypted_data, dearmor('-----BEGIN PGP PRIVATE KEY BLOCK-----...'));

### Практический пример

    -- Таблица с шифрованными колонками
    CREATE TABLE customers (
        id serial PRIMARY KEY,
        name text NOT NULL,
        email text NOT NULL,
        ssn bytea, -- Зашифрованный SSN
        credit_card bytea -- Зашифрованная карта
    );
    
    -- Вставка (шифрование)
    INSERT INTO customers (name, email, ssn, credit_card)
    VALUES (
        'John Doe',
        'john@example.com',
        pgp_sym_encrypt('123-45-6789', 'app-secret-key'),
        pgp_sym_encrypt('4111-1111-1111-1111', 'app-secret-key')
    );
    
    -- Чтение (дешифрование)
    SELECT 
        id,
        name,
        email,
        pgp_sym_decrypt(ssn, 'app-secret-key') as ssn,
        pgp_sym_decrypt(credit_card, 'app-secret-key') as credit_card
    FROM customers
    WHERE id = 1;

### Хеширование

Для паролей и данных, которые не нужно расшифровывать:

    -- Хеширование пароля
    SELECT crypt('user_password', gen_salt('bf', 10));
    
    -- Проверка пароля
    SELECT crypt('user_password', stored_hash) = stored_hash;
    
    -- MD5 (устаревший)
    SELECT md5('data');
    
    -- SHA-256
    SELECT digest('data', 'sha256');

## Часть 5: Управление ключами

### Хранение ключей

| Подход | Описание | Безопасность |
|--------|----------|--------------|
| В базе | Хранить ключи в отдельной таблице | ❌ Низкая |
| В приложении | Передавать ключи через параметры | ⚠️ Средняя |
| В файле | Файл с правами 600 | ⚠️ Средняя |
| KMS (AWS/Azure/GCP) | Облачный Key Management Service | ✅ Высокая |
| HashiCorp Vault | Специализированное хранилище | ✅ Высокая |
| HSM | Hardware Security Module | ✅ Максимальная |

### HashiCorp Vault

    # Получить ключ из Vault
    vault kv get secret/postgres/encryption-key
    
    # Интеграция с приложением
    # Приложение запрашивает ключ при старте
    # Ключ никогда не хранится в коде

### AWS KMS

    -- Использование KMS через обёртку
    -- Приложение вызывает KMS API для дешифрования ключа
    -- Использует расшифрованный ключ для pgcrypto

## Лучшие практики

| Практика | Описание |
|----------|----------|
| TLS 1.2+ | Минимальная версия для соединений |
| TDE | Шифрование диска всегда |
| pgcrypto | Для чувствительных колонок |
| KMS/Vault | Для управления ключами |
| Не хранить ключи | В коде или в базе |
| Ротация ключей | Регулярно |
| Аудит | Кто и когда шифровал/дешифровал |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| SSL включён для всех подключений | ⬜ |
| TDE настроено (LUKS/EBS/...) | ⬜ |
| Чувствительные колонки зашифрованы | ⬜ |
| Ключи хранятся в KMS/Vault | ⬜ |
| Регулярная ротация ключей | ⬜ |
| Нет хардкода ключей | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Шифрование данных карт | pgcrypto + KMS |
| GDPR | Защита персональных данных | TDE + колоночное шифрование |
| HIPAA | Шифрование PHI | TDE + аудит |
| SOX | Контроль доступа к данным | Шифрование + аудит |

## Вывод

Шифрование в PostgreSQL многослойное:
1. **SSL/TLS** — данные в пути
2. **TDE** — данные на диске
3. **pgcrypto** — данные в колонках
4. **KMS/Vault** — управление ключами

Ключевой принцип: **шифровать всегда, управлять ключами централизованно.**

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_