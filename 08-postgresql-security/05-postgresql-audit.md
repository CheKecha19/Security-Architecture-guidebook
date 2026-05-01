# Аудит в PostgreSQL

## Часть 1: Зачем нужен аудит?

Представь, что в магазине произошла кража. Нет камер — нет доказательств. Есть камеры — можно понять: кто, когда, что взял.

**Аудит в PostgreSQL** — это «камеры наблюдения» для базы данных. Записывает: кто и когда обращался к данным, что изменил, что удалил.

## Зачем нужен аудит?

### Ситуация 1: Расследование инцидента

Клиенты жалуются: «Мои данные изменились!» Без аудита — невозможно понять: кто и когда вносил правки.

### Ситуация 2: Compliance (SOX)

Аудиторы требуют: покажите, кто имел доступ к финансовым данным и что изменял. Без аудита — нарушение.

### Ситуация 3: Анализ угроз

Подозрительная активность: необычно много запросов в 3 часа ночи. Аудит показывает: кто и откуда подключался.

## Часть 2: Встроенные механизмы аудита

### log_statement

    # postgresql.conf
    log_statement = 'all'  -- Все команды
    log_statement = 'ddl'  -- Только DDL
    log_statement = 'mod'  -- DML (INSERT, UPDATE, DELETE)
    log_statement = 'none' -- Ничего

**Проблема:** логирует весь SQL, включая чувствительные данные (пароли, карты).

### log_connections и log_disconnections

    # postgresql.conf
    log_connections = on
    log_disconnections = on
    
    # Лог
    LOG:  connection authorized: user=alice database=mydb application=psql
    LOG:  disconnection: session time: 0:05:12 user=alice database=mydb

### log_line_prefix

    # Формат строки лога
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
    
    # Результат
    2024-01-15 10:30:00 [12345]: [1-1] user=alice,db=mydb,app=psql,client=10.0.0.5 LOG:  statement: SELECT * FROM users;

## Часть 3: Расширение pgaudit

### Установка

    # Установить расширение
    CREATE EXTENSION IF NOT EXISTS pgaudit;
    
    # Настройка
    # postgresql.conf
    shared_preload_libraries = 'pgaudit'
    pgaudit.log = 'write, ddl'
    pgaudit.log_catalog = off
    pgaudit.log_parameter = on
    pgaudit.log_relation = on
    pgaudit.log_rows = on
    pgaudit.log_statement = on
    pgaudit.log_client = off
    pgaudit.log_level = log

### Уровни логирования

| Параметр | Что логируется |
|----------|----------------|
| none | Ничего |
| read | SELECT, COPY FROM |
| write | INSERT, UPDATE, DELETE, COPY TO |
| ddl | CREATE, ALTER, DROP |
| role | GRANT, REVOKE |
| function | EXECUTE |
| misc | SET, SHOW, DISCARD, NOTIFY |
| misc_set | SET (для безопасности) |
| all | Всё |

### Примеры логов

    -- DDL
    AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.users,
    CREATE TABLE users (id serial PRIMARY KEY, name text)
    
    -- DML
    AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,
    INSERT INTO users (name) VALUES ('alice')
    
    -- SELECT
    AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.users,
    SELECT * FROM users WHERE id = 1
    
    -- GRANT
    AUDIT: SESSION,1,1,ROLE,GRANT,TABLE,public.users,
    GRANT SELECT ON users TO analyst

### Детальное логирование

    -- С параметрами
    pgaudit.log_parameter = on
    
    -- Результат
    AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,
    INSERT INTO users (name, email) VALUES ($1, $2)
    PARAMETERS: [alice, alice@example.com]
    
    -- С количеством строк
    pgaudit.log_rows = on
    
    -- Результат
    AUDIT: SESSION,1,1,WRITE,UPDATE,TABLE,public.users,,
    UPDATE users SET status = 'active'
    ROWS: 150

## Часть 4: Аудит на уровне таблиц (trigger-based)

### Audit trigger

    -- Таблица для истории
    CREATE TABLE audit_log (
        id bigserial PRIMARY KEY,
        table_name text NOT NULL,
        operation text NOT NULL,
        old_data jsonb,
        new_data jsonb,
        changed_at timestamp DEFAULT now(),
        changed_by text DEFAULT current_user
    );
    
    -- Функция аудита
    CREATE OR REPLACE FUNCTION audit_trigger()
    RETURNS TRIGGER AS $$
    BEGIN
        IF (TG_OP = 'DELETE') THEN
            INSERT INTO audit_log (table_name, operation, old_data)
            VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD));
            RETURN OLD;
        ELSIF (TG_OP = 'UPDATE') THEN
            INSERT INTO audit_log (table_name, operation, old_data, new_data)
            VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
            RETURN NEW;
        ELSIF (TG_OP = 'INSERT') THEN
            INSERT INTO audit_log (table_name, operation, new_data)
            VALUES (TG_TABLE_NAME, TG_OP, row_to_json(NEW));
            RETURN NEW;
        END IF;
        RETURN NULL;
    END;
    $$ LANGUAGE plpgsql SECURITY DEFINER;
    
    -- Применить к таблице
    CREATE TRIGGER users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();

### Просмотр истории

    -- Кто и когда изменил запись
    SELECT * FROM audit_log 
    WHERE table_name = 'users' 
    ORDER BY changed_at DESC;
    
    -- Что изменилось
    SELECT 
        changed_at,
        changed_by,
        old_data->>'name as old_name,
        new_data->>'name as new_name
    FROM audit_log
    WHERE table_name = 'users' AND operation = 'UPDATE';

## Часть 5: Централизация и SIEM

### Логи в syslog

    # postgresql.conf
    logging_collector = on
    log_destination = 'syslog'
    syslog_facility = 'LOCAL0'
    syslog_ident = 'postgres'

### Интеграция с SIEM

| SIEM | Способ интеграции |
|------|-------------------|
| Splunk | Forwarder читает логи |
| ELK (Elastic) | Filebeat → Logstash → Elasticsearch |
| QRadar | Syslog collector |
| ArcSight | SmartConnector |
| Datadog | Agent собирает логи |

### Пример: Filebeat для PostgreSQL

    # filebeat.yml
    filebeat.inputs:
      - type: log
        enabled: true
        paths:
          - /var/log/postgresql/*.log
        fields:
          service: postgresql
        fields_under_root: true
    
    output.elasticsearch:
      hosts: ["http://localhost:9200"]

### Мониторинг подозрительной активности

    -- Запросы из необычных мест
    SELECT client_addr, usename, COUNT(*) 
    FROM pg_stat_activity 
    GROUP BY client_addr, usename;
    
    -- Пользователи без активности > 90 дней
    SELECT usename, max(backend_start) as last_activity
    FROM pg_stat_activity
    GROUP BY usename;

## Лучшие практики

| Практика | Описание |
|----------|----------|
| pgaudit | Обязательно для production |
| Логи в SIEM | Централизация и анализ |
| Ротация логов | Не переполнить диск |
| Защита логов | Права 600, отдельный диск |
| Аудит DDL | Всегда логировать |
| Аудит прав | GRANT, REVOKE |
| Триггеры | Для критичных таблиц |
| Мониторинг | Алерты на подозрительное |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| pgaudit установлен и настроен | ⬜ |
| Логируются DDL и DML | ⬜ |
| Логи централизованы (SIEM) | ⬜ |
| Ротация настроена | ⬜ |
| Триггеры на критичных таблицах | ⬜ |
| Мониторинг подозрительного | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Аудит доступа к данным карт | pgaudit + SIEM |
| SOX | Аудит изменений финансовых данных | Триггеры + pgaudit |
| HIPAA | Аудит доступа к PHI | pgaudit + мониторинг |
| GDPR | Аудит обработки персональных данных | Полный аудит |
| ISO 27001 | Логирование событий безопасности | SIEM + ротация |

## Вывод

Аудит в PostgreSQL — обязательный элемент безопасности:
1. **pgaudit** — структурированные логи
2. **Триггеры** — история изменений
3. **SIEM** — централизация и анализ

Без аудита — нет visibility. Без visibility — нет безопасности.

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_