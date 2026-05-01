# Hardening PostgreSQL

## Часть 1: Что такое харденинг?

Представь, что ты строишь сейф. Не просто железную коробку, а многослойную защиту: толстые стенки, сложный замок, датчики движения, охрана.

**Харденинг PostgreSQL** — это процесс усиления защиты базы данных: отключение ненужного, включение необходимого, настройка контроля доступа.

## Зачем нужен харденинг?

### Ситуация 1: Стандартная установка

PostgreSQL после установки слушает на всех интерфейсах (`listen_addresses = '*'`). Любой может попытаться подключиться. Это удобно для разработки, опасно для production.

### Ситуация 2: Лишние расширения

В базе установлены расширения, которые не используются: `adminpack`, `file_fdw`, `dblink`. Это дополнительная поверхность атаки.

### Ситуация 3: Слабые пароли

Пользователь `postgres` имеет пароль `postgres`. Это известный дефолт, который проверяют первым делом.

## Часть 2: CIS Benchmarks

### Что такое CIS?

**Center for Internet Security** (CIS) — организация, которая публикует стандарты безопасности. **CIS PostgreSQL Benchmark** — чек-лист из ~100 пунктов для защиты PostgreSQL.

### Ключевые пункты

| Категория | Пункт | Описание |
|-----------|-------|----------|
| Установка | 1.1 | Установка из доверенных репозиториев |
| Установка | 1.2 | Проверка контрольных сумм |
| Сеть | 2.1 | Ограничение listen_addresses |
| Сеть | 2.2 | SSL обязателен |
| Сеть | 2.3 | Не использовать trust |
| Аутентификация | 3.1 | Superuser только для администрирования |
| Аутентификация | 3.2 | scram-sha-256 |
| Аутентификация | 3.3 | Ротация паролей |
| Аудит | 4.1 | pgaudit включён |
| Аудит | 4.2 | log_connections |
| Аудит | 4.3 | log_disconnections |
| Конфигурация | 5.1 | Отключить лишние функции |
| Конфигурация | 5.2 | Ограничить доступ к файлам |
| Обновление | 6.1 | Последняя стабильная версия |
| Backup | 7.1 | Регулярные бэкапы |
| Backup | 7.2 | Тестирование восстановления |

## Часть 3: Сетевые настройки

### listen_addresses

    # Неправильно — слушать все интерфейсы
    listen_addresses = '*'
    
    # Правильно — только нужные
    listen_addresses = 'localhost,10.0.0.5'
    
    # Для standalone
    listen_addresses = '127.0.0.1'

### Порт

    # Стандартный порт — известен всем
    port = 5432
    
    # Изменить на нестандартный (security through obscurity)
    port = 15432
    
    # Но настоящая безопасность — firewall

### Firewall

    # Linux (iptables)
    iptables -A INPUT -p tcp --dport 5432 -s 10.0.0.0/24 -j ACCEPT
    iptables -A INPUT -p tcp --dport 5432 -j DROP
    
    # Или через pg_hba.conf
    hostssl all all 10.0.0.0/24 scram-sha-256
    host    all all 0.0.0.0/0  reject

## Часть 4: Аутентификация и авторизация

### Пароли

    # Проверить метод шифрования
    SHOW password_encryption;
    
    # Должно быть: scram-sha-256
    
    # Изменить пароль superuser
    ALTER USER postgres WITH PASSWORD 'new-strong-password';
    
    # Создать отдельного администратора
    CREATE ROLE admin WITH LOGIN PASSWORD 'strong-password' SUPERUSER;
    ALTER USER postgres WITH NOLOGIN;  -- Отключить стандартный логин

### Роли

    -- Удалить ненужные роли
    DROP ROLE IF EXISTS test_user;
    DROP ROLE IF EXISTS temp_user;
    
    -- Отозвать права у PUBLIC
    REVOKE ALL ON SCHEMA public FROM PUBLIC;
    
    -- Создать роли для приложений
    CREATE ROLE app_read WITH NOLOGIN;
    CREATE ROLE app_write WITH NOLOGIN;
    
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
    GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO app_write;

## Часть 5: Конфигурация безопасности

### postgresql.conf

    # === Сеть ===
    listen_addresses = 'localhost'
    port = 5432
    ssl = on
    ssl_min_protocol_version = 'TLSv1.2'
    ssl_prefer_server_ciphers = on
    
    # === Аутентификация ===
    password_encryption = 'scram-sha-256'
    
    # === Аудит ===
    log_connections = on
    log_disconnections = on
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
    log_statement = 'ddl'
    
    # === Лимиты ===
    max_connections = 100  # Не больше нужного
    
    # === Расширения ===
    shared_preload_libraries = 'pgaudit'
    
    # === Файлы ===
    log_file_mode = 0600
    
    # === Отключить опасные функции ===
    allow_system_table_mods = off
    ignore_system_indexes = off
    
    # === Таймауты ===
    statement_timeout = '30s'
    lock_timeout = '10s'
    idle_in_transaction_session_timeout = '60s'

### pg_hba.conf

    # Локальные подключения
    local   all             postgres                                peer
    local   all             all                                     scram-sha-256
    
    # Удалённые только через SSL
    hostssl all             all             10.0.0.0/24             scram-sha-256
    
    # Запретить всё остальное
    host    all             all             0.0.0.0/0               reject
    
    # IPv6
    hostssl all             all             ::1/128                 scram-sha-256

### Отключение опасных функций

    -- Проверить установленные расширения
    SELECT * FROM pg_extension;
    
    -- Удалить опасные
    DROP EXTENSION IF EXISTS adminpack;
    DROP EXTENSION IF EXISTS file_fdw;
    
    -- Ограничить pg_read_file
    -- Только суперпользователи
    
    -- Отключить LOAD для непривилегированных
    ALTER SYSTEM SET local_preload_libraries = '';

## Часть 6: Обновление и патчи

### Версии PostgreSQL

| Версия | Статус | Рекомендация |
|--------|--------|--------------|
| 9.x | EOL | Обновить срочно |
| 10 | EOL | Обновить |
| 11 | EOL | Обновить |
| 12 | Поддержка | Планировать обновление |
| 13 | Поддержка | OK |
| 14 | Поддержка | OK |
| 15 | Поддержка | OK |
| 16 | Актуальная | Рекомендуется |

### Процесс обновления

    1. Проверить release notes
    2. Сделать бэкап
    3. Протестировать на staging
    4. Обновить: pg_upgrade или логическая репликация
    5. Проверить работоспособность
    6. Мониторить

### Безопасность патчей

    # Проверить текущую версию
    SELECT version();
    
    # Подписаться на рассылки безопасности
    # postgresql.org/list/pgsql-announce/
    # postgresql.org/support/security/

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Минимум привилегий | Только необходимые права |
| SSL | Всегда |
| Скрем | Только scram-sha-256 |
| Логи | Всё включить |
| Обновление | Регулярно |
| Бэкап | Регулярно + тестирование |
| Firewall | Ограничить доступ |
| CIS | Следовать бенчмаркам |
| Отключить лишнее | Расширения, функции |
| Мониторинг | Подозрительная активность |

### Чек-лист харденинга

| Проверка | Статус |
|----------|--------|
| listen_addresses ограничены | ⬜ |
| SSL включён и настроен | ⬜ |
| scram-sha-256 | ⬜ |
| Superuser защищён | ⬜ |
| PUBLIC права отозваны | ⬜ |
| pgaudit включён | ⬜ |
| Логи в SIEM | ⬜ |
| Расширения минимизированы | ⬜ |
| Последняя версия | ⬜ |
| Бэкап настроен | ⬜ |
| Firewall настроен | ⬜ |
| CIS Benchmark пройден | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| CIS | Следовать бенчмаркам | Все пункты |
| PCI DSS | Харденинг БД | SSL, аудит, права |
| SOX | Контроль изменений | Аудит DDL |
| HIPAA | Защита PHI | Харденинг + шифрование |
| NIST | Управление конфигурацией | Стандартизированная настройка |

## Вывод

Харденинг PostgreSQL — не разовая задача, а процесс:
1. **Базовая конфигурация** — listen, SSL, auth
2. **Роли и права** — минимум привилегий
3. **Аудит** — всё логировать
4. **Обновление** — регулярные патчи
5. **Мониторинг** — подозрительное отслеживать

Ключевой принцип: **отключить всё ненужное, усилить всё нужное.**

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_