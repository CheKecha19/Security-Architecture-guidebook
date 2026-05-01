# Аудит в PostgreSQL: отслеживание действий пользователей

## Что такое аудит в PostgreSQL?

Представь магазин без камер. Кража — и непонятно, кто, когда, как. С камерами — всё видно: кто вошёл, что взял, когда ушёл.

**Аудит в PostgreSQL** — это «камеры наблюдения» для базы данных. Записывает: кто подключался, что читал, что изменил, когда. Без аудита — утечка данных обнаруживается через недели или месяцы. С аудитом — за минуты.

## Эволюция и мотивация

Ранние версии PostgreSQL (до 9.5) имели базовое логирование через `log_statement`. Но это было грубо: логировать всё или ничего, без структуры, без фильтрации.

Версия 9.5 (2016) ввела **pgaudit** — расширение для детального аудита. Версия 13 улучшила встроенное логирование. Современная PostgreSQL поддерживает: логирование соединений, логирование DDL, логирование DML через pgaudit, триггер-based аудит, JSON-формат логов.

Регуляторы требуют аудит: **HIPAA** §164.312(b) — audit controls; **PCI DSS** 10.2 — audit trails; **SOX** — аудит финансовых транзакций; **GDPR** — Article 30 — record of processing activities.

## Зачем это нужно?

### Сценарий 1: Расследование инцидента

Клиенты жалуются: «Мои данные изменились!» Без аудита — непонятно кто виноват. С аудитом — видно: `User:app` выполнил `UPDATE users SET email = 'hacker@evil.com' WHERE id = 12345` в 14:35:22.

### Сценарий 2: Compliance (SOX)

Аудиторы требуют: покажите, кто имел доступ к финансовым данным. Аудит-логи показывают: `User:accounting-app` читал `transactions`, `User:reporting-app` читал `quarterly_reports`, `User:admin` изменял `audit_log`... Wait, admin changed audit log? That's a red flag!

### Сценарий 3: Обнаружение инсайдера

Сотрудник увольняется. В его последний день аудит показывает: он читал таблицы, к которым раньше не обращался. `SELECT * FROM salaries`, `SELECT * FROM customer_pii`. Запрос данных «на память» перед уходом.

## Основные концепции

### Встроенное логирование

**log_statement:**
```
# postgresql.conf
log_statement = 'all'     -- Всё
log_statement = 'ddl'     -- Только DDL (CREATE, ALTER, DROP)
log_statement = 'mod'     -- DML (INSERT, UPDATE, DELETE)
```

**Проблема:** логирует весь SQL, включая чувствительные данные (`INSERT INTO users (ssn) VALUES ('123-45-6789')`).

**log_connections / log_disconnections:**
```
# postgresql.conf
log_connections = on
log_disconnections = on

# Результат:
LOG:  connection authorized: user=alice database=mydb
LOG:  disconnection: session time: 0:05:12 user=alice
```

**log_line_prefix:**
```
# Формат строки лога
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,client=%h '

# 2024-01-15 10:30:00 [12345]: [1-1] user=alice,db=mydb,client=10.0.0.5
```

### pgaudit

**pgaudit** — расширение для структурированного аудита.

**Установка:**
```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;

# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
pgaudit.log_parameter = on
pgaudit.log_rows = on
```

**Уровни логирования:**

| Параметр | Что логируется |
|----------|----------------|
| `none` | Ничего |
| `read` | SELECT, COPY FROM |
| `write` | INSERT, UPDATE, DELETE |
| `ddl` | CREATE, ALTER, DROP |
| `role` | GRANT, REVOKE |
| `function` | EXECUTE |
| `all` | Всё |

**Примеры логов:**
```
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.users,
CREATE TABLE users (id serial PRIMARY KEY, name text)

AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,
INSERT INTO users (name) VALUES ('alice')

AUDIT: SESSION,1,1,WRITE,UPDATE,TABLE,public.users,,
UPDATE users SET status = 'active'
ROWS: 150
```

### Триггер-based аудит

**Триггер для аудита изменений:**
```sql
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
```

### Централизация в SIEM

**Syslog:**
```
# postgresql.conf
log_destination = 'syslog'
syslog_facility = 'LOCAL0'
syslog_ident = 'postgres'
```

**Filebeat + ELK:**
```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/postgresql/*.log
    fields:
      service: postgresql

output.elasticsearch:
  hosts: ["http://localhost:9200"]
```

## Сравнение подходов аудита

| Подход | Детальность | Производительность | Хранение | Сложность |
|--------|-------------|-------------------|----------|-----------|
| **log_statement** | Высокая | Низкая | Большой объём | Низкая |
| **pgaudit** | Высокая | Средняя | Средний | Средняя |
| **Trigger-based** | Строка | Низкая | Большой объём | Средняя |
| **pg_stat_statements** | Агрегат | Минимальная | Малый | Низкая |

## Уроки из реальных инцидентов

### Инцидент 1: Удаление audit logs (2019)

Злоумышленник получил доступ к PostgreSQL как `User:admin`. Перед уходом выполнил: `DELETE FROM audit_log WHERE changed_at > current_date - interval '7 days'`.

**Хронология:** Аудит-лог хранился в той же базе. Злоумышленник удалил следы. Расследование осложнено.

**Ущерб:** Невозможность расследования. Неизвестен масштаб компрометации.

**Как защита изменила бы исход:** Аудит-лог в отдельной базе или внешней системе (SIEM). Immutable audit log (append-only). Регулярный экспорт. `pgAudit` логи в файловую систему, не в таблицу.

### Инцидент 2: Отсутствие аудита DDL (2020)

Администратор случайно удалил таблицу `orders`. Никто не знал, когда и кто. Backup был раз в сутки — потеря 18 часов данных.

**Хронология:** `log_statement = 'mod'` — DDL не логировался. `DROP TABLE orders` прошёл незамеченным.

**Ущерб:** Потеря данных. Восстановление из backup. Простой 6 часов.

**Как защита изменила бы исход:** `log_statement = 'ddl'` или `pgaudit.log = 'ddl'`. Алерт на DDL в production. Backup чаще (или continuous archiving).

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **pgaudit** | Структурированный аудит | Production | NIST AU-6 |
| **log_statement** | Базовое логирование | Production | NIST AU-6 |
| **Trigger-based** | Аудит изменений | Разработка | NIST AU-6 |
| **ELK / Splunk** | Централизация | Runtime | NIST AU-6 |
| **Filebeat** | Сбор логов | Runtime | NIST AU-6 |

## Архитектурные решения

### Defense-in-Depth для аудита

| Слой | Механизм | Контроль |
|------|----------|----------|
| Database | pgaudit | Все операции |
| Application | Trigger-based | Изменения данных |
| OS | Syslog | Централизация |
| SIEM | ELK / Splunk | Анализ, алерты |
| Backup | Immutable log | Невозможность удаления |

### Метрики аудита

| Метрика | Целевое значение | Проверка |
|---------|-----------------|----------|
| Audit coverage | 100% DDL, 100% write | pgaudit config |
| Log centralization | < 1 мин lag | SIEM dashboard |
| Failed logins | Алерт | pgaudit / log_connections |
| DDL changes | Алерт | pgaudit |
| Audit log deletion | 0 | Immutable storage |

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «log_statement = 'all' достаточно для аудита».

`log_statement = 'all'` логирует весь SQL, включая чувствительные данные. Это риск утечки через логи. Лучше использовать `pgaudit` с фильтрацией.

**Ошибка 2:** «Аудит-лог можно хранить в той же базе».

Если злоумышленник получает доступ к базе — он может удалить аудит-лог. Аудит должен быть immutable и внешним.

**Ошибка 3:** «Аудит не влияет на производительность».

Аудит влияет. pgaudit добавляет 5-15% overhead. Trigger-based — 10-25%. Нужно балансировать coverage и performance.

### Три ситуационных вопроса

**Вопрос 1:** «Как аудитировать SELECT в PostgreSQL без огромных логов?»

Ответ: **pgaudit** с `log = 'read'` — но это много данных. Альтернатива: **pg_stat_statements** — агрегированная статистика (кто, как часто, какие таблицы). **Sample-based аудит** — логировать 1% SELECT. **Trigger на sensitive таблицах** — только для критичных данных.

**Вопрос 2:** «Как защитить audit log от удаления?»

Ответ: **Immutable storage** — append-only файлы. **SIEM** — логи отправляются внешне, не хранятся локально. **Separate database** — аудит-лог в другой instance. **Read-only replica** — репликация аудит-лога на read-only standby.

**Вопрос 3:** «Как соответствовать GDPR через аудит PostgreSQL?»

Ответ: **pgaudit** для всех операций с персональными данными. **Article 30** — record of processing: кто обрабатывал, когда, зачем. **Right to erasure** — аудит подтверждает удаление. **Data breach notification** — аудит показывает, кто имел доступ при breach.

## Чек-лист понимания

- [ ] Что такое **pgaudit**?
- [ ] Как настроить **log_statement**?
- [ ] Как создать **триггер для аудита**?
- [ ] Как централизовать **логи**?
- [ ] Какие **уровни** pgaudit?
- [ ] Как защитить **audit log**?
- [ ] Как аудитировать **DDL**?
- [ ] Как аудитировать **DML**?
- [ ] Как **мониторить** failed logins?
- [ ] Как **экспортировать** аудит?

### Ответы на чек-лист

1. **pgaudit** — расширение PostgreSQL для структурированного аудита. Логирует операции с детализацией.

2. **log_statement:** `log_statement = 'ddl'` — только DDL. `log_statement = 'mod'` — DML. `log_statement = 'all'` — всё (осторожно).

3. **Триггер:** `CREATE TRIGGER ... AFTER INSERT OR UPDATE OR DELETE ... EXECUTE FUNCTION audit_trigger();`. Функция записывает old/new данные в audit table.

4. **Централизация:** Syslog, Filebeat → ELK/Splunk. Или через `log_destination = 'csvlog'` + сбор.

5. **Уровни pgaudit:** none, read, write, ddl, role, function, all.

6. **Защита:** Immutable storage, SIEM, separate database, read-only replica.

7. **DDL:** `pgaudit.log = 'ddl'` или `log_statement = 'ddl'`.

8. **DML:** `pgaudit.log = 'write'` или триггеры.

9. **Failed logins:** `log_connections = on`, анализ `pg_stat_activity`, Fail2ban.

10. **Экспорт:** `COPY (SELECT * FROM audit_log) TO '/path/audit.csv';` или через SIEM.

## Ключевые выводы для собеседования

- **Аудит — обязателен** — для compliance и расследований
- **pgaudit — лучший инструмент** — структурированный, гибкий
- **Аудит-лог вне базы** — immutable, централизованный
- **Баланс** — coverage vs performance vs хранение
- **DDL всегда логировать** — структурные изменения критичны

---

_Статья создана на основе анализа материалов по PostgreSQL_
