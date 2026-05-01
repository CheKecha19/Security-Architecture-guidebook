# Hardening PostgreSQL

## Часть 1: Что такое харденинг?

Представь замок. Не просто дверь с засовом, а многослойная защита: ров, подъёмный мост, внешняя стена, башни, внутренний двор, гарнизон.

**Харденинг PostgreSQL** — усиление защиты базы: отключение лишнего, включение нужного, минимизация поверхности атаки.

## Зачем нужен харденинг?

### Ситуация 1: Стандартная установка

После `initdb` PostgreSQL слушает на `localhost`, но `listen_addresses` легко изменить на `*`. Без харденинга — опасно.

### Ситуация 2: Лишние расширения

Установлены `adminpack`, `file_fdw`, `dblink` — но не используются. Это дополнительная поверхность атаки.

### Ситуация 3: Слабые пароли

Пользователь `postgres` с паролем `postgres`. Первое, что проверяет злоумышленник.

## Часть 2: CIS Benchmarks

### Что такое CIS

**Center for Internet Security** — организация, публикующая стандарты безопасности. **CIS PostgreSQL Benchmark** — ~100 пунктов для защиты.

### Ключевые пункты

| Категория | Пункты |
|-----------|--------|
| Установка | Доверенные репозитории, проверка контрольных сумм |
| Сеть | Ограничение listen_addresses, SSL |
| Аутентификация | scram-sha-256, запрет trust |
| Авторизация | Минимум привилегий, аудит ролей |
| Аудит | pgaudit, логи |
| Конфигурация | Отключение опасных функций |
| Обновление | Последняя стабильная версия |
| Бэкап | Регулярно + тестирование |

## Часть 3: Сетевая безопасность

### listen_addresses

    # Неправильно:
    listen_addresses = '*'
    
    # Правильно:
    listen_addresses = 'localhost,10.0.0.5'

### Порт и firewall

    # Стандартный порт — известен
    port = 5432
    
    # Firewall: только нужные IP
    iptables -A INPUT -p tcp --dport 5432 -s 10.0.0.0/24 -j ACCEPT
    iptables -A INPUT -p tcp --dport 5432 -j DROP

## Часть 4: Аутентификация и авторизация

### Пароли

    -- Метод шифрования
    SHOW password_encryption;  -- Должно быть: scram-sha-256
    
    -- Сменить пароль superuser
    ALTER USER postgres WITH PASSWORD 'new-strong-password';
    
    -- Отключить стандартный логин
    ALTER USER postgres WITH NOLOGIN;
    CREATE ROLE admin WITH LOGIN PASSWORD 'strong' SUPERUSER;

### Роли и права

    -- Отозвать PUBLIC
    REVOKE ALL ON SCHEMA public FROM PUBLIC;
    
    -- Создать роли для приложений
    CREATE ROLE app_read NOLOGIN;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;

## Часть 5: Конфигурация

### postgresql.conf — безопасный шаблон

    # Сеть
    listen_addresses = 'localhost'
    ssl = on
    ssl_min_protocol_version = 'TLSv1.2'
    
    # Аутентификация
    password_encryption = 'scram-sha-256'
    
    # Аудит
    log_connections = on
    log_disconnections = on
    log_line_prefix = '%t [%p]: user=%u,db=%d,client=%h '
    
    # Лимиты
    max_connections = 100
    
    # Расширения
    shared_preload_libraries = 'pgaudit'
    
    # Таймауты
    statement_timeout = '30s'
    idle_in_transaction_session_timeout = '60s'

### pg_hba.conf

    local   all             postgres                                peer
    local   all             all                                     scram-sha-256
    hostssl all             all             10.0.0.0/24             scram-sha-256
    host    all             all             0.0.0.0/0               reject

### Отключение опасного

    -- Удалить неиспользуемые расширения
    DROP EXTENSION IF EXISTS adminpack;
    DROP EXTENSION IF EXISTS file_fdw;
    
    -- Отключить загрузку произвольных библиотек
    ALTER SYSTEM SET local_preload_libraries = '';

## Часть 6: Обновление

### Версии PostgreSQL

| Версия | Статус |
|--------|--------|
| 9.x-11 | EOL, обновить срочно |
| 12-14 | Поддержка |
| 15-16 | Рекомендуется |

### Процесс

    1. Прочитать release notes
    2. Сделать бэкап
    3. Протестировать на staging
    4. Обновить (pg_upgrade или логическая репликация)
    5. Проверить

## Лучшие практики

| Практика | Реализация |
|----------|------------|
| Минимум привилегий | Только необходимое |
| SSL | Всегда |
| Скрем | Только scram-sha-256 |
| Логи | Всё включить |
| Обновление | Регулярно |
| Бэкап | Регулярно + тест |
| CIS | Следовать бенчмаркам |
| Мониторинг | Подозрительное |

## Вывод

Харденинг PostgreSQL:
1. **Сеть** — ограничить доступ
2. **Аутентификация** — scram-sha-256
3. **Авторизация** — минимум прав
4. **Аудит** — всё логировать
5. **Обновление** — регулярные патчи
6. **CIS** — следовать бенчмаркам

Принцип: **отключить всё ненужное, усилить всё нужное.**

---
_Статья создана на основе анализа материалов по PostgreSQL_