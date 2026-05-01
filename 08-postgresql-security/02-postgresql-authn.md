# Аутентификация в PostgreSQL

## Часть 1: Зачем нужна аутентификация?

Представь банк. У входа стоит охранник. Каждый клиент показывает паспорт. Охранник проверяет: это действительно тот человек? Без этой проверки — в банк войдёт кто угодно.

**Аутентификация в PostgreSQL** — это «охранник у входа» в базу данных. Каждое подключение проходит проверку: кто это и имеет ли право входа.

## Зачем нужна аутентификация?

### Ситуация 1: Стандартный пароль

После установки PostgreSQL создаёт пользователя `postgres` с паролем по умолчанию (или без пароля). Если не изменить — любой может подключиться.

### Ситуация 2: Сетевой доступ

PostgreSQL по умолчанию слушает на `localhost`. Если администратор изменил на `*` — база доступна из сети. Без аутентификации — катастрофа.

### Ситуация 3: Приложение с паролем в коде

Приложение подключается с паролем `password123`, захардкоженным в коде. Утечка кода = утечка пароля.

## Часть 2: Файл pg_hba.conf

### Что такое HBA

**Host-Based Authentication** — файл, определяющий: кто может подключаться, к какой базе, с какого адреса, каким методом.

    # Формат записи
    # TYPE  DATABASE  USER  ADDRESS  METHOD  [OPTIONS]

### Типы подключений

| Тип | Описание | Безопасность |
|-----|----------|--------------|
| `local` | Unix-сокет | Безопаснее TCP |
| `host` | TCP/IP | Требует SSL |
| `hostssl` | TCP/IP + SSL | Рекомендуется |
| `hostnossl` | TCP/IP без SSL | ❌ Не рекомендуется |

### Методы аутентификации

| Метод | Описание | Безопасность | Когда использовать |
|-------|----------|--------------|-------------------|
| `trust` | Без пароля | ❌ Нет | Только localhost dev |
| `reject` | Запретить | ✅ Да | Блокировка |
| `md5` | MD5-хеш | ⚠️ Устарел | Не использовать |
| `password` | Открытым текстом | ❌ Нет | Никогда |
| `scram-sha-256` | SCRAM | ✅ Высокая | Рекомендуется |
| `ident` | Имя ОС-пользователя | ⚠️ Низкая | Локально |
| `peer` | Имя ОС через Unix-сокет | ✅ Средняя | Локально |
| `ldap` | LDAP/AD | ✅ Высокая | Enterprise |
| `gss` | Kerberos (GSSAPI) | ✅ Высокая | Enterprise |
| `sspi` | Windows SSPI | ✅ Высокая | Windows-домены |
| `radius` | RADIUS | ✅ Средняя | Enterprise |
| `cert` | Клиентский сертификат | ✅ Высокая | mTLS |

### Пример pg_hba.conf

    # Локальные подключения через Unix-сокет
    local   all             postgres                                peer
    local   all             all                                     scram-sha-256
    
    # Удалённые подключения только через SSL + SCRAM
    hostssl all             all             10.0.0.0/24             scram-sha-256
    
    # Репликация только с определённого хоста
    hostssl replication     replicator      10.0.0.5/32             scram-sha-256
    
    # Запретить всё остальное
    host    all             all             0.0.0.0/0               reject
    
    # IPv6
    hostssl all             all             ::1/128                 scram-sha-256

**Важно:** Порядок записей критичен! PostgreSQL применяет первую подходящую запись.

## Часть 3: SCRAM-SHA-256

### Что такое SCRAM

**Salted Challenge Response Authentication Mechanism** — современный метод, введённый в PostgreSQL 10.

**Как работает:**
1. Клиент отправляет имя пользователя
2. Сервер отправляет salt (случайная строка) и nonce
3. Клиент вычисляет proof на основе пароля + salt + nonce
4. Сервер проверяет proof, не получая пароль в открытом виде

### Настройка SCRAM

    # postgresql.conf
    password_encryption = 'scram-sha-256'
    
    # pg_hba.conf
    hostssl all all 10.0.0.0/24 scram-sha-256
    
    # Создать пользователя с SCRAM
    CREATE USER alice WITH PASSWORD 'strong-password';
    -- Пароль автоматически хешируется SCRAM

### Проверка

    -- Какой метод используется для пользователя
    SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'alice';
    -- Результат: SCRAM-SHA-256$4096:...

## Часть 4: SSL/TLS

### Настройка сервера

    # postgresql.conf
    ssl = on
    ssl_cert_file = 'server.crt'
    ssl_key_file = 'server.key'
    ssl_ca_file = 'ca.crt'
    ssl_crl_file = 'crl.pem'
    
    ssl_min_protocol_version = 'TLSv1.2'

### Клиентские настройки

| sslmode | Описание |
|---------|----------|
| disable | Без SSL ❌ |
| allow | SSL, если доступен |
| prefer | Предпочитать SSL |
| require | Требовать SSL |
| verify-ca | Проверить CA |
| verify-full | Проверить CA и hostname ✅ |

    # connection string
    postgresql://user:pass@host/db?sslmode=verify-full&sslrootcert=ca.crt

## Часть 5: LDAP и Active Directory

### Простая LDAP-аутентификация

    # pg_hba.conf
    hostssl all all 10.0.0.0/24 ldap ldapserver=ad.company.com \
        ldapprefix="cn=" ldapsuffix=",dc=company,dc=com"

### LDAPS (LDAP over SSL)

    hostssl all all 10.0.0.0/24 ldap ldapserver=ad.company.com ldaptls=1

### Kerberos (GSSAPI)

    # postgresql.conf
    krb_server_keyfile = 'FILE:/etc/postgresql/krb5.keytab'
    
    # pg_hba.conf
    hostssl all all 0.0.0.0/0 gss include_realm=0

## Часть 6: Лучшие практики

| Практика | Реализация |
|----------|------------|
| scram-sha-256 | Всегда |
| hostssl | Для удалённых |
| Ограничение IP | Не 0.0.0.0/0 |
| Отдельные пользователи | Не postgres для приложений |
| Регулярная ротация паролей | ALTER USER ... PASSWORD |
| Мониторинг failed logins | log_connections, log_disconnections |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| password_encryption = 'scram-sha-256' | ⬜ |
| Нет trust для удалённых | ⬜ |
| hostssl обязателен | ⬜ |
| Слабые пароли изменены | ⬜ |
| Регулярный аудит pg_hba.conf | ⬜ |

## Вывод

Аутентификация в PostgreSQL:
1. **pg_hba.conf** — кто, куда, каким методом
2. **SCRAM-SHA-256** — рекомендуемый метод
3. **SSL/TLS** — шифрование соединений
4. **LDAP/Kerberos** — для enterprise
5. **Практики** — минимум прав, ротация, аудит

Без аутентификации — база данных открыта миру.

---
_Статья создана на основе анализа материалов по PostgreSQL_