# Шифрование в PostgreSQL: TLS, TDE и pgcrypto

## Что такое шифрование в PostgreSQL?

Представь письмо. Обычный конверт — любой может вскрыть и прочитать. Сейф — вскрыть сложно, содержимое защищено. Получатель имеет ключ — открывает, читает. Никто по пути не видел, что внутри.

**Шифрование в PostgreSQL** — это «сейф» для данных. Защищает три состояния:
- **In-transit** — при передаче между клиентом и сервером
- **At-rest** — при хранении на диске сервера
- **In-database** — отдельные чувствительные колонки

Без шифрования данные в PostgreSQL — открытый текст. Любой, кто имеет доступ к сети или диску, может прочитать всё: пароли, карты, персональные данные.

## Эволюция и мотивация

Ранние версии PostgreSQL (до 8.0) не имели встроенного шифрования. Все данные передавались открытым текстом. Даже пароли (MD5-хеши) передавались в открытом виде при аутентификации.

Версия 8.0 (2005) ввела поддержку **SSL** для шифрования соединений. Версия 9.1 добавила **pgcrypto** — расширение для шифрования колонок. Версия 10 улучшила SSL, добавив поддержку TLS 1.2. Версия 13 добавила **SCRAM-SHA-256**, убирающую передачу паролей.

Но PostgreSQL до сих пор не имеет встроенного **TDE** (Transparent Data Encryption) — шифрования всего диска на уровне СУБД. Для TDE используется OS-level шифрование (LUKS) или облачное шифрование (EBS). Это архитектурное решение: PostgreSQL фокусируется на функциональности, шифрование хранения — ответственность инфраструктуры.

## Зачем это нужно?

### Сценарий 1: Перехват трафика

Приложение и база в разных дата-центрах. Трафик через публичную сеть. Без шифрования — злоумышленник перехватывает пакеты и читает все SQL-запросы и результаты: «SELECT * FROM users WHERE ssn = '123-45-6789'». Видит и запрос, и ответ.

С TLS — перехваченные данные — случайные байты. Невозможно прочитать без приватного ключа сервера.

### Сценарий 2: Кража диска

Сервер украден из дата-центра. Жёсткий диск содержит файлы PostgreSQL: `/var/lib/postgresql/base/13456/1234`. Без at-rest encryption — данные читаемы любым hex-редактором. Файлы PostgreSQL — структурированные, легко распознать строки.

С LUKS: диск зашифрован. Без ключа — бесполезен. Даже forensic recovery не поможет.

### Сценарий 3: Утечка чувствительных данных

База содержит номера карт в таблице `payments`. Столбец `card_number` — plaintext. Даже при получении доступа к базе (через SQL Injection или скомпрометированное приложение) — данные видны.

С pgcrypto: столбец `card_number` зашифрован. Приложение расшифровывает при чтении. Злоумышленник видит только зашифрованные данные.

## Основные концепции

### SSL/TLS для соединений

**TLS** (Transport Layer Security) криптографический протокол для защиты соединений.

**Как работает:**
1. **Handshake** — клиент и сервер договариваются о версии TLS, cipher suite, обмениваются сертификатами
2. **Key exchange** — генерация сеансового симметричного ключа через асимметричную криптографию (RSA, ECDHE)
3. **Encrypted transmission** — все данные шифруются симметричным ключом (AES)
4. **Integrity check** — MAC (Message Authentication Code) гарантирует неизменность

**Настройка сервера:**
```
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_crl_file = '/etc/ssl/crl/crl.pem'
ssl_min_protocol_version = 'TLSv1.2'
```

**Проверка:**
```sql
-- Проверить, что SSL работает
SHOW ssl;

-- Текущие SSL-подключения
SELECT * FROM pg_stat_ssl;

-- Детали соединения
SELECT ssl, version, cipher, bits FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

**Клиентские настройки (sslmode):**

| Режим | Описание | Безопасность |
|-------|----------|--------------|
| `disable` | Без SSL | ❌ Нет |
| `allow` | SSL, если доступен | ⚠️ |
| `prefer` | Предпочитать SSL | ⚠️ |
| `require` | Требовать SSL | ✅ |
| `verify-ca` | Проверить CA | ✅ |
| `verify-full` | Проверить CA и hostname | ✅✅ |

**Рекомендация:** `sslmode=verify-full` для production.

### Transparent Data Encryption (TDE)

PostgreSQL не имеет встроенного TDE. Альтернативы:

| Уровень | Решение | Когда использовать |
|---------|---------|-------------------|
| **OS-level** | LUKS, dm-crypt, BitLocker | Bare metal, on-premise |
| **Cloud** | AWS EBS encryption, Azure Disk Encryption | Облако |
| **Filesystem** | eCryptfs | Специфические случаи |

**LUKS для PostgreSQL:**
```bash
# Создать зашифрованный раздел
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 pgdata
mkfs.ext4 /dev/mapper/pgdata
mount /dev/mapper/pgdata /var/lib/postgresql
```

**AWS EBS encryption:**
```
# При создании инстанса
EBS volumes:
  - Encrypted: Yes
  - KMS key: aws/ebs или customer-managed key
```

### Шифрование колонок (pgcrypto)

**pgcrypto** — расширение для криптографических операций.

**Симметричное шифрование:**
```sql
-- Установить расширение
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Шифрование
SELECT pgp_sym_encrypt('secret data', 'my-password');

-- Дешифрование
SELECT pgp_sym_decrypt(encrypted_column, 'my-password');

-- С настройками
SELECT pgp_sym_encrypt('secret', 'password', 'cipher-algo=aes256');
```

**Практический пример:**
```sql
CREATE TABLE customers (
  id serial PRIMARY KEY,
  name text NOT NULL,
  email text NOT NULL,
  ssn bytea,  -- Зашифрованный
  credit_card bytea  -- Зашифрованный
);

-- Вставка
INSERT INTO customers (name, email, ssn)
VALUES ('John Doe', 'john@example.com', pgp_sym_encrypt('123-45-6789', 'app-secret'));

-- Чтение
SELECT id, name, email,
  pgp_sym_decrypt(ssn, 'app-secret') as ssn
FROM customers;
```

**Хеширование (не шифрование):**
```sql
-- Пароли — хешировать, не шифровать
SELECT crypt('user_password', gen_salt('bf', 10));

-- Проверка
SELECT crypt('input', stored_hash) = stored_hash;
```

## Сравнение подходов

| Подход | In-transit | At-rest | In-database | Сложность | Производительность |
|--------|-----------|---------|-------------|-----------|-------------------|
| **No encryption** | ❌ | ❌ | ❌ | Низкая | Максимальная |
| **SSL only** | ✅ | ❌ | ❌ | Низкая | Снижение ~5-15% |
| **SSL + LUKS** | ✅ | ✅ | ❌ | Средняя | Снижение ~10-20% |
| **SSL + pgcrypto** | ✅ | ❌ | ✅ | Средняя | Снижение ~15-25% |
| **SSL + LUKS + pgcrypto** | ✅ | ✅ | ✅ | Высокая | Снижение ~20-30% |

## Уроки из реальных инцидентов

### Инцидент 1: Утечка через незашифрованный backup (2019)

Компания делала backup PostgreSQL через `pg_dump`. Backup хранился на NFS без шифрования. Злоумышленник получил доступ к NFS (через уязвимость в NFS server) и скачал backup.

**Хронология:** Backup содержал все данные, включая SSN, карты, пароли. Открытым текстом. Никакого шифрования.

**Ущерб:** Утечка данных 1 миллиона клиентов. Регуляторный штраф $2,000,000.

**Как защита изменила бы исход:** Шифрование backup: `pg_dump | gpg --encrypt`. Хранение backup в encrypted S3 bucket. Ограничение доступа к NFS.

### Инцидент 2: Heartbleed и PostgreSQL (2014)

Heartbleed (CVE-2014-0160) — уязвимость в OpenSSL. PostgreSQL использовала OpenSSL для TLS.

**Хронология:** Злоумышленник отправлял malformed heartbeat request. Сервер отвечал 64KB памяти, включая приватные ключи, пароли, сообщения.

**Ущерб:** Компрометация приватных ключей. Возможность расшифровки прошлых и будущих соединений.

**Как защита изменила бы исход:** Регулярное обновление OpenSSL. Perfect Forward Secrecy (ECDHE) — даже при компрометации ключа, прошлые сессии защищены. Мониторинг security advisories.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **OpenSSL** | TLS implementation | Runtime | NIST SC-13 |
| **pgcrypto** | Шифрование колонок | Разработка | NIST SC-28 |
| **LUKS / dm-crypt** | OS-level disk encryption | Настройка | NIST SC-28 |
| **Cloud KMS** | Managed encryption | Облако | SOC 2 |
| **Vault** | Управление секретами | CI/CD | NIST SC-28 |
| **SSL Labs** | Проверка TLS конфигурации | Регулярно | CIS |

## Архитектурные решения

### Defense-in-Depth для шифрования

| Слой | Защита | Ключевой вопрос |
|------|--------|----------------|
| Network | Firewall | Кто имеет сетевой доступ? |
| Transport | TLS 1.3 | Перехваченный трафик читаем? |
| Storage | LUKS / Cloud KMS | Украденный диск читаем? |
| Application | pgcrypto | При компрометации БД — данные защищены? |
| Backup | Шифрование backup | Backup в безопасном месте? |

### Метрики шифрования

| Метрика | Целевое значение | Проверка |
|---------|-----------------|----------|
| SSL coverage | 100% | `pg_stat_ssl` |
| TLS version | TLS 1.2+ | SSL Labs scan |
| Weak cipher usage | 0 | testssl.sh |
| Disk encryption | 100% | OS-level check |
| Backup encryption | 100% | Backup verification |

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «SSL достаточно для безопасности PostgreSQL».

SSL защищает трафик. Но не защищает данные на диске, не защищает от SQL Injection, не защищает от привилегированного доступа. Нужна комбинация: SSL + at-rest + pgcrypto + ACL.

**Ошибка 2:** «Шифрование сильно снижает производительность».

Modern CPUs имеют AES-NI (hardware acceleration). Overhead TLS с AES-GCM — 5-15%. pgcrypto — 15-25%. Для большинства систем приемлемо. «Не шифровать из-за производительности» — false economy.

**Ошибка 3:** «TDE в PostgreSQL работает из коробки».

PostgreSQL не имеет встроенного TDE. Нужно использовать OS-level (LUKS) или cloud-level (EBS encryption). Это не функция PostgreSQL, это функция инфраструктуры.

### Три ситуационных вопроса

**Вопрос 1:** «Как защитить данные карт в PostgreSQL по PCI DSS?»

Ответ: **TLS 1.2+** для всех соединений. **SCRAM-SHA-256** для аутентификации. **pgcrypto** для шифрования PAN (Primary Account Number). **RLS** — только процессинговый сервис видит полные номера. **LUKS** или **EBS encryption** для at-rest. **pgaudit** для аудита. **Удаление** после обработки.

**Вопрос 2:** «Как ротировать ключи шифрования без downtime?»

Ответ: **pgcrypto** — нет встроенной ротации. Нужно: расшифровать старым ключом, зашифровать новым. Для LUKS — `cryptsetup reencrypt`. Для cloud KMS — автоматическая ротация (AWS KMS). **Application-level** — хранить key ID с данными, поддерживать несколько ключей.

**Вопрос 3:** «В чём разница между hashing и encryption?»

Ответ: **Hashing** — односторонняя функция. Из данных получается фиксированная строка. Нельзя восстановить данные. Используется для паролей. **Encryption** — двусторонняя. Можно зашифровать и расшифровать. Используется для данных, которые нужно читать.

## Чек-лист понимания

- [ ] Что защищает **SSL**? Что не защищает?
- [ ] Какие **версии TLS** безопасны?
- [ ] Что такое **pgcrypto**?
- [ ] Как зашифровать **колонку**?
- [ ] Какие варианты **at-rest шифрования**?
- [ ] Чем **hashing** отличается от **encryption**?
- [ ] Как проверить **SSL-соединение**?
- [ ] Что такое **TDE** и есть ли в PostgreSQL?
- [ ] Как **ротировать ключи**?
- [ ] Как защитить **backup**?

### Ответы на чек-лист

1. **SSL** защищает: конфиденциальность (шифрование), целостность (MAC), аутентификацию сервера. **Не защищает:** данные на диске, SQL Injection, привилегированный доступ.

2. **Безопасны:** TLS 1.2 и 1.3. **Не безопасны:** SSL 2.0, 3.0, TLS 1.0, 1.1.

3. **pgcrypto** — расширение PostgreSQL для криптографии. Шифрование, хеширование, генерация случайных чисел.

4. **Колонка:** `ALTER TABLE table ADD COLUMN encrypted bytea; INSERT ... pgp_sym_encrypt(data, key); SELECT ... pgp_sym_decrypt(encrypted, key);`

5. **At-rest:** LUKS (Linux), BitLocker (Windows), EBS encryption (AWS), Azure Disk Encryption.

6. **Hashing** — односторонний, нельзя восстановить. **Encryption** — двусторонний, можно расшифровать.

7. **Проверка:** `SHOW ssl;` `SELECT * FROM pg_stat_ssl;` SSL Labs scan.

8. **TDE** (Transparent Data Encryption) — шифрование диска. **Нет** в PostgreSQL из коробки. Использовать OS-level или cloud-level.

9. **Ротация:** pgcrypto — нет встроенной, ручная. LUKS — `cryptsetup reencrypt`. Cloud KMS — автоматическая.

10. **Backup:** Шифрование при создании (`pg_dump | gpg`). Хранение в encrypted S3. Ограничение доступа.

## Ключевые выводы для собеседования

- **SSL — минимум** — но не единственная защита
- **pgcrypto — для sensitive колонок** — пароли, карты, SSN
- **At-rest — обязательно** — LUKS или облачное шифрование
- **Backup — шифровать** — иначе бэкап = утечка
- **PostgreSQL нет TDE** — используйте инфраструктуру

---

_Статья создана на основе анализа материалов по PostgreSQL_
