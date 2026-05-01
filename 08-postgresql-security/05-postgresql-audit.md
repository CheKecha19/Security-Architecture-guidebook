# Аудит в PostgreSQL

## Часть 1: Зачем нужен аудит?

Представь магазин без камер. Кража — и непонятно, кто, когда, как. С камерами — всё видно.

**Аудит в PostgreSQL** — «камеры наблюдения» для базы данных. Записывает: кто подключался, что читал, что изменил, когда.

## Зачем нужен аудит?

### Ситуация 1: Расследование инцидента

Клиенты жалуются: «Мои данные изменились!» Без аудита — непонятно кто виноват. С аудитом — видно: `User:app` выполнил `UPDATE` в 14:35.

### Ситуация 2: Compliance (SOX)

Аудиторы требуют: покажите, кто имел доступ к финансовым данным. Аудит-логи — доказательство.

### Ситуация 3: Подозрительная активность

В 3 часа ночи — необычно много запросов от `User:app`. Аудит показывает: кто и откуда подключался.

## Часть 2: Встроенное логирование

### log_statement

    # postgresql.conf
    log_statement = 'all'     -- Всё
    log_statement = 'ddl'     -- Только DDL (CREATE, ALTER, DROP)
    log_statement = 'mod'     -- DML (INSERT, UPDATE, DELETE)

**Проблема:** логирует весь SQL, включая чувствительные данные.

### log_connections / log_disconnections

    # postgresql.conf
    log_connections = on
    log_disconnections = on
    
    # Результат:
    LOG:  connection authorized: user=alice database=mydb
    LOG:  disconnection: session time: 0:05:12 user=alice

### log_line_prefix

    # Формат строки лога
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
    
    # 2024-01-15 10:30:00 [12345]: [1-1] user=alice,db=mydb,app=psql,client=10.0.0.5

## Часть 3: Расширение pgaudit

### Установка

    CREATE EXTENSION IF NOT EXISTS pgaudit;
    
    # postgresql.conf
    shared_preload_libraries = 'pgaudit'
    pgaudit.log = 'write, ddl'
    pgaudit.log_parameter = on
    pgaudit.log_rows = on

### Уровни логирования

| Параметр | Что логируется |
|----------|----------------|
| none | Ничего |
| read | SELECT, COPY FROM |
| write | INSERT, UPDATE, DELETE |
| ddl | CREATE, ALTER, DROP |
| role | GRANT, REVOKE |
| function | EXECUTE |
| all | Всё |

### Примеры логов

    -- DDL
    AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.users,
    CREATE TABLE users (id serial PRIMARY KEY, name text)
    
    -- DML
    AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,
    INSERT INTO users (name) VALUES ('alice')
    
    -- С параметрами
    AUDIT: ... INSERT INTO users (name) VALUES ($1)
    PARAMETERS: [alice]

## Часть 4: Trigger-based аудит

### Триггер для аудита изменений

    -- Таблица истории
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

    -- Кто изменил запись
    SELECT * FROM audit_log WHERE table_name = 'users';
    
    -- Что изменилось
    SELECT changed_at, changed_by,
        old_data->>'name as old_name,
        new_data->>'name as new_name
    FROM audit_log WHERE operation = 'UPDATE';

## Часть 5: Централизация в SIEM

### Syslog

    # postgresql.conf
    log_destination = 'syslog'
    syslog_facility = 'LOCAL0'
    syslog_ident = 'postgres'

### Filebeat + ELK

    filebeat.inputs:
      - type: log
        paths:
          - /var/log/postgresql/*.log
        fields:
          service: postgresql
    
    output.elasticsearch:
      hosts: ["http://localhost:9200"]

### Мониторинг

    -- Запросы из необычных мест
    SELECT client_addr, usename, COUNT(*) 
    FROM pg_stat_activity 
    GROUP BY client_addr, usename;

## Лучшие практики

| Практика | Описание |
|----------|----------|
| pgaudit | Обязательно для production |
| Логи в SIEM | Централизация |
| Ротация | Не переполнить диск |
| Защита логов | Права 600 |
| Триггеры | Для критичных таблиц |
| Мониторинг | Алерты на аномалии |

## Вывод

Аудит в PostgreSQL:
1. **pgaudit** — структурированные логи
2. **Триггеры** — история изменений
3. **SIEM** — централизация
4. **Мониторинг** — аномалии

Без аудита — нет visibility. Без visibility — нет безопасности.

---
_Статья создана на основе анализа материалов по PostgreSQL_