# Шифрование в PostgreSQL

## Часть 1: Зачем нужно шифрование?

Представь письмо. Обычный конверт — любой может вскрыть. Сейф — вскрыть сложно, содержимое защищено.

**Шифрование в PostgreSQL** — «сейф» для данных:
- **In-transit** — при передаче между клиентом и сервером
- **At-rest** — при хранении на диске
- **In-database** — отдельные чувствительные колонки

## Зачем нужно шифрование?

### Ситуация 1: Перехват трафика

Приложение и база в разных дата-центрах. Трафик через публичную сеть. Без шифрования — данные видны при перехвате.

### Ситуация 2: Кража диска

Сервер украден из дата-центра. Диски содержат базу. Без at-rest encryption — данные читаемы.

### Ситуация 3: Утечка чувствительных данных

База содержит номера карт. Даже при получении доступа к базе — без шифрования колонки данные видны.

## Часть 2: SSL/TLS для соединений

### Как работает SSL

1. Клиент подключается
2. Сервер предъявляет сертификат
3. Клиент проверяет сертификат (CA, hostname)
4. Устанавливается зашифрованный канал
5. Все данные шифруются

### Настройка сервера

    # postgresql.conf
    ssl = on
    ssl_cert_file = '/etc/ssl/certs/server.crt'
    ssl_key_file = '/etc/ssl/private/server.key'
    ssl_ca_file = '/etc/ssl/certs/ca.crt'
    ssl_crl_file = '/etc/ssl/crl/crl.pem'
    
    ssl_min_protocol_version = 'TLSv1.2'

### Проверка

    -- Проверить, что SSL работает
    SHOW ssl;
    
    -- Текущие SSL-подключения
    SELECT * FROM pg_stat_ssl;
    
    -- Детали соединения
    SELECT ssl, version, cipher, bits
    FROM pg_stat_ssl
    WHERE pid = pg_backend_pid();

### Клиентские настройки

| sslmode | Описание | Безопасность |
|---------|----------|--------------|
| disable | Без SSL | ❌ Нет |
| allow | SSL, если доступен | ⚠️ |
| prefer | Предпочитать SSL | ⚠️ |
| require | Требовать SSL | ✅ |
| verify-ca | Проверить CA | ✅ |
| verify-full | Проверить CA и hostname | ✅✅ |

## Часть 3: Transparent Data Encryption (TDE)

### Что такое TDE

Шифрование всего диска с данными. PostgreSQL «не знает» о шифровании — это делает ОС.

### LUKS (Linux)

    # Создать зашифрованный раздел
    cryptsetup luksFormat /dev/sdb1
    cryptsetup luksOpen /dev/sdb1 pgdata
    mkfs.ext4 /dev/mapper/pgdata
    mount /dev/mapper/pgdata /var/lib/postgresql

### Облачное шифрование

| Провайдер | Решение | Ключи |
|-----------|---------|-------|
| AWS | EBS encrypted volumes | AWS KMS |
| Azure | Storage Service Encryption | Azure Key Vault |
| GCP | Customer-managed encryption keys | Cloud KMS |

### Преимущества и недостатки

| | TDE |
|---|-----|
| ✅ Прозрачно для приложения | ❌ Не защищает от SQL-инъекций |
| ✅ Защита от кражи диска | ❌ Нет защиты на уровне колонок |
| ✅ Автоматическое управление (в облаке) | ❌ Ключи в памяти |

## Часть 4: Шифрование колонок (pgcrypto)

### Расширение pgcrypto

    CREATE EXTENSION IF NOT EXISTS pgcrypto;

### Симметричное шифрование

    -- Шифрование
    SELECT pgp_sym_encrypt('secret', 'my-password');
    
    -- Дешифрование
    SELECT pgp_sym_decrypt(encrypted_column, 'my-password');
    
    -- С настройками
    SELECT pgp_sym_encrypt('secret', 'password', 
        'cipher-algo=aes256, compress-algo=2');

### Практический пример

    CREATE TABLE customers (
        id serial PRIMARY KEY,
        name text NOT NULL,
        email text NOT NULL,
        ssn bytea,  -- Зашифрованный
        credit_card bytea  -- Зашифрованный
    );
    
    -- Вставка
    INSERT INTO customers (name, email, ssn)
    VALUES (
        'John Doe',
        'john@example.com',
        pgp_sym_encrypt('123-45-6789', 'app-secret')
    );
    
    -- Чтение
    SELECT id, name, email,
        pgp_sym_decrypt(ssn, 'app-secret') as ssn
    FROM customers;

### Хеширование (не шифрование)

    -- Пароли — хешировать, не шифровать
    SELECT crypt('user_password', gen_salt('bf', 10));
    
    -- Проверка
    SELECT crypt('input', stored_hash) = stored_hash;

## Часть 5: Управление ключами

### Хранение ключей

| Подход | Безопасность | Описание |
|--------|------------|----------|
| В базе | ❌ Низкая | Хранить в отдельной таблице |
| В приложении | ⚠️ Средняя | Передавать через параметры |
| В файле | ⚠️ Средня | Файл с правами 600 |
| KMS | ✅ Высокая | AWS KMS, Azure Key Vault |
| HashiCorp Vault | ✅ Высокая | Специализированное хранилище |
| HSM | ✅ Максимальная | Аппаратный модуль |

### Пример с Vault

    -- Приложение получает ключ из Vault при старте
    -- Ключ никогда не хранится в коде или базе
    -- vault kv get secret/postgres/encryption-key

## Часть 6: Лучшие практики

| Практика | Реализация |
|----------|------------|
| TLS 1.2+ | Минимум для соединений |
| TDE | Шифрование диска всегда |
| pgcrypto | Для чувствительных колонок |
| KMS/Vault | Для управления ключами |
| Не хранить ключи | В коде или базе |
| Ротация ключей | Регулярно |
| Аудит | Кто шифровал/дешифровал |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| SSL включён | ⬜ |
| TDE настроено | ⬜ |
| Чувствительные колонки зашифрованы | ⬜ |
| Ключи в KMS/Vault | ⬜ |
| Регулярная ротация | ⬜ |

## Вывод

Шифрование в PostgreSQL — многослойное:
1. **SSL/TLS** — данные в пути
2. **TDE** — данные на диске
3. **pgcrypto** — данные в колонках
4. **KMS/Vault** — управление ключами

Правило: **шифровать всегда, управлять ключами централизованно.**

---
_Статья создана на основе анализа материалов по PostgreSQL_