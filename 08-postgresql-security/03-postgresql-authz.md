# Авторизация в PostgreSQL: роли, привилегии и Row-Level Security

## Что такое авторизация в PostgreSQL?

Представь офисное здание. Ты прошёл охрану (аутентификация) и получил бейдж. Но бейдж открывает не все двери. Ты можешь войти на свой этаж, в переговорную, в кафетерий. Но не в серверную. Не в архив. Не в кабинет директора. Если бейдж открывает всё — это не бейдж, это мастер-ключ.

**Авторизация в PostgreSQL** — это именно такая «система доступа к комнатам». После аутентификации (кто ты?) нужно решить: что тебе можно? Можешь ли ты читать таблицу `orders`? Можешь ли ты писать в `payments`? Можешь ли ты удалять таблицы? Создавать новых пользователей?

PostgreSQL реализует **Discretionary Access Control (DAC)** — владелец объекта решает, кто имеет к нему доступ. Это гибко, но требует внимательного управления.

## Эволюция и мотивация

Ранние версии PostgreSQL (до 8.0) имели простую модель: пользователи и группы. Права: SELECT, INSERT, UPDATE, DELETE, RULE, REFERENCES, TRIGGER. Всё.

Версия 8.1 (2005) ввела **ROLE** — унифицированную сущность для пользователей и групп. Это упростило управление: роль может быть LOGIN (пользователь) или NOLOGIN (группа). Версия 9.0 добавила **Column-Level Privileges** — права на конкретные колонки. Версия 9.5 (2016) революционизировала безопасность, введя **Row-Level Security (RLS)** — защиту на уровне строк.

Эта эволюция отражает рост требований: от «у каждого своя база» до «множество пользователей работают с одними данными, но видят разное».

## Зачем это нужно?

### Сценарий 1: Разделение окружений

Компания имеет три окружения: `dev`, `staging`, `production`. Разработчики должны писать в `dev`, читать из `staging`, не иметь доступа к `production`. Без авторизации — разработчик случайно подключается к продакшену, запускает `DELETE FROM orders WHERE 1=1` — и стирает все заказы.

С авторизацией: `User:dev-team` имеет **ALL** на `Schema:dev_*`, **SELECT** на `Schema:staging_*`, нет прав на `Schema:production_*`. Попытка доступа отклонена. Лог: «ERROR: permission denied for relation orders».

### Сценарий 2: Разделение по командам

Команда `payments` работает с таблицами `payments_*`. Команда `analytics` — с `analytics_*` и `metrics`. Без авторизации — аналитик подключается к `payments_received` и получает данные всех транзакций. Это PCI DSS violation.

С авторизацией: `User:analytics-app` имеет **SELECT** только на `Table:analytics_*` и `Table:metrics`. Попытка чтения `payments_received` — отклонена. Для анализа платежей — создаётся анонимизированное представление `payments_anonymized`.

### Сценарий 3: Row-Level Security (RLS)

HR-система. Менеджер видит все резюме. Кандидат — только своё. Рекрутер — назначенные ему. Без RLS — или все видят всё (утечка), или отдельные таблицы на каждого (немасштабируемо).

С RLS: таблица `resumes` имеет политику «видеть только строки, где `owner_id = current_user_id()`». Менеджер имеет bypass? Нет — менеджер видит все через отдельную роль с `BYPASSRLS`? Нет — менеджер использует отдельное представление с агрегированными данными.

## Основные концепции

### Роли

В PostgreSQL **роль = пользователь = группа**. Это единая сущность.

| Атрибут | Описание |
|---------|----------|
| **LOGIN** | Может подключаться (это «пользователь») |
| **SUPERUSER** | Все права, обходит проверки |
| **CREATEDB** | Может создавать базы |
| **CREATEROLE** | Может создавать роли |
| **REPLICATION** | Может делать репликацию |
| **BYPASSRLS** | Обходит Row-Level Security |
| **INHERIT** | Наследует права групп (по умолчанию) |

**Создание ролей:**
```sql
-- Пользователь
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong-password';

-- Групповая роль
CREATE ROLE developers NOLOGIN;

-- Суперпользователь (осторожно!)
CREATE ROLE admin WITH LOGIN PASSWORD 'very-strong' SUPERUSER;

-- Проверить
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb FROM pg_roles;
```

**Наследование:**
```sql
-- Добавить пользователя в группу
GRANT developers TO app_user;

-- Теперь app_user имеет права developers
-- Проверить членство
SELECT r.rolname AS role, m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON am.roleid = r.oid
JOIN pg_roles m ON am.member = m.oid;
```

### Привилегии

| Уровень | Объект | Права |
|---------|--------|-------|
| **База данных** | Database | CONNECT, CREATE, TEMP |
| **Схема** | Schema | USAGE, CREATE |
| **Таблица** | Table | SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER |
| **Колонка** | Column | SELECT, UPDATE |
| **Последовательность** | Sequence | USAGE, SELECT, UPDATE |
| **Функция** | Function | EXECUTE |

**Назначение привилегий:**
```sql
-- Права на базу
GRANT CONNECT ON DATABASE mydb TO app_user;

-- Права на схему
GRANT USAGE ON SCHEMA public TO app_user;
GRANT CREATE ON SCHEMA public TO admin;

-- Права на таблицу
GRANT SELECT, INSERT ON orders TO app_user;
GRANT ALL ON orders TO admin;

-- Права на колонки
GRANT SELECT (id, name, email) ON users TO analyst;

-- Права по умолчанию для новых объектов
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO analyst;
```

### Row-Level Security (RLS)

**RLS** — защита на уровне строк. Позволяет контролировать: какие строки видит пользователь.

**Включение:**
```sql
ALTER TABLE resumes ENABLE ROW LEVEL SECURITY;

-- Проверить
SELECT relname, relrowsecurity FROM pg_class WHERE relname = 'resumes';
```

**Создание политик:**
```sql
-- Политика: пользователь видит только свои строки
CREATE POLICY resume_owner ON resumes
  FOR ALL
  TO public
  USING (owner_id = current_user_id());

-- Политика для администраторов
CREATE POLICY resume_admin ON resumes
  FOR ALL
  TO admin
  USING (true);  -- Видят всё

-- Только SELECT для аналитиков
CREATE POLICY resume_analyst ON resumes
  FOR SELECT
  TO analyst
  USING (status = 'published');
```

**Функции для RLS:**
```sql
CREATE OR REPLACE FUNCTION current_user_id() RETURNS INT AS $$
BEGIN
  RETURN (SELECT id FROM users WHERE username = current_user);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Важно:** SUPERUSER и BYPASSRLS обходят RLS! Проверить:
```sql
SELECT rolname, rolbypassrls FROM pg_roles WHERE rolbypassrls = true;
```

### Default Privileges

```sql
-- Установить права по умолчанию для новых таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO analyst;

-- Для объектов, созданных конкретным пользователем
ALTER DEFAULT PRIVILEGES FOR ROLE app_user IN SCHEMA public
GRANT SELECT ON TABLES TO analyst;
```

## Сравнение подходов авторизации

| Подход | Гранулярность | Сложность | Производительность |
|--------|---------------|-----------|-------------------|
| **Database per tenant** | Высокая | Высокая | Накладные расходы |
| **Schema per tenant** | Высокая | Средняя | Меньше накладных |
| **Table per tenant** | Средняя | Средняя | Меньше накладных |
| **RLS** | Строка | Низкая | Небольшие накладные |
| **Column-level grants** | Колонка | Низкая | Минимальные |

## Уроки из реальных инцидентов

### Инцидент 1: Утечка через неправильные default privileges (2018)

Администратор настроил: `ALTER DEFAULT PRIVILEGES GRANT ALL ON TABLES TO public`. Все новые таблицы автоматически доступны всем. Аналитик создал таблицу `salaries_temp` — и все увидели зарплаты.

**Хронология:** Default privileges — мощный инструмент, но опасный. `public` — это псевдороль, включающая всех пользователей. `GRANT ALL TO public` = открытый доступ.

**Ущерб:** Утечка конфиденциальных данных. Юридические последствия.

**Как защита изменила бы исход:** Никаких `GRANT TO public` без явного approval. Ревью default privileges. Аудит: кто создал таблицу, какие права.

### Инцидент 2: Обход RLS через функцию (2020)

Разработчик создал функцию:
```sql
CREATE FUNCTION get_all_resumes() RETURNS TABLE (...) AS $$
  SELECT * FROM resumes;
$$ LANGUAGE SQL SECURITY DEFINER;
```

Функция выполняется с правами владельца (владелец имеет BYPASSRLS). Пользователь без прав на `resumes` вызывает функцию — и получает все строки.

**Хронология:** `SECURITY DEFINER` — функция выполняется с правами создателя, не вызывающего. Если создатель — superuser или имеет BYPASSRLS — RLS обходится.

**Ущерб:** Несанкционированный доступ к данным.

**Как защита изменила бы исход:** `SECURITY INVOKER` по умолчанию. Если нужен `SECURITY DEFINER` — создавать отдельную роль с минимальными правами, без BYPASSRLS. Аудит функций: кто создал, с какими правами.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **GRANT/REVOKE** | Управление привилегиями | Регулярно | NIST AC-3 |
| **RLS Policies** | Защита на уровне строк | Разработка | NIST AC-3 |
| **Column-level grants** | Защита на уровне колонок | Разработка | NIST AC-3 |
| **Default Privileges** | Автоматизация прав | Разработка | NIST AC-3 |
| **pg_roles / pg_auth_members** | Аудит ролей | Регулярно | NIST AC-2 |
| **information_schema** | Переносимый аудит | Регулярно | NIST AC-2 |

## Архитектурные решения

### Defense-in-Depth для авторизации

| Слой | Механизм | Контроль |
|------|----------|----------|
| Database | CONNECT | Кто может подключиться |
| Schema | USAGE | Кто может видеть объекты |
| Table | SELECT/INSERT/UPDATE/DELETE | Кто может работать с данными |
| Column | Column-level grants | Кто может видеть колонки |
| Row | RLS | Кто может видеть строки |
| Function | EXECUTE + SECURITY INVOKER | Кто может выполнять |

### Метрики авторизации

| Метрика | Целевое значение | Проверка |
|---------|-----------------|----------|
| Superuser connections | Минимум | `pg_stat_activity` |
| Objects with PUBLIC access | 0 | `information_schema.table_privileges` |
| RLS policies | На все sensitive таблицы | `pg_policies` |
| Functions with SECURITY DEFINER | Аудит | `pg_proc` |
| Stale roles | 0 | `pg_roles` (не использовались 90 дней) |

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «RLS заменяет авторизацию».

RLS — дополнительный слой, не замена. RLS проверяет строки, но не заменяет проверку таблиц. Если пользователь имеет `DELETE` на таблицу — RLS может не защитить (если нет соответствующей политики). Нужна комбинация: ACL + RLS.

**Ошибка 2:** «SUPERUSER безопасен, потому что это администратор».

SUPERUSER обходит все проверки: RLS, ownership, ACL. Он может читать `pg_authid`, менять конфигурацию, удалять всё. Принцип: минимум суперпользователей, регулярный аудит их действий.

**Ошибка 3:** «Default privileges упрощают управление».

Default privileges — двойной меч. Они упрощают, но создают риск: новые объекты автоматически получают права. Если `public` — проблема. Если не `public` — нужно управлять.

### Три ситуационных вопроса

**Вопрос 1:** «Как реализовать multi-tenancy в PostgreSQL?»

Ответ: **Три подхода:**
1. **Database per tenant** — полная изоляция, но накладные расходы на соединения.
2. **Schema per tenant** — изоляция на уровне схем, общая база.
3. **RLS per tenant** — одна таблица, политики `USING (tenant_id = current_tenant_id())`. Минимум накладных расходов, но требует внимательного управления.

**Вопрос 2:** «Как защититься от SQL Injection через RLS?»

Ответ: RLS **не защищает от SQL Injection**. SQL Injection — это внедрение кода в запрос. RLS — это фильтрация результатов. Защита от SQL Injection: **Prepared Statements**, **ORM с параметризацией**, **Input validation**. RLS ограничивает ущерб: даже при SQL Injection злоумышленник видит только строки, доступные его роли.

**Вопрос 3:** «Как аудитировать права в PostgreSQL?»

Ответ: **Запросы:**
```sql
-- Кто имеет права на таблицу
SELECT grantee, privilege_type FROM information_schema.table_privileges WHERE table_name = 'orders';

-- Роли и их атрибуты
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb FROM pg_roles WHERE rolname NOT LIKE 'pg_%';

-- Политики RLS
SELECT * FROM pg_policies;

-- Функции с SECURITY DEFINER
SELECT proname, prosecdef FROM pg_proc WHERE prosecdef = true;
```

## Чек-лист понимания

- [ ] Чем **роль** отличается от **пользователя**?
- [ ] Какие **атрибуты** роли есть?
- [ ] Как назначить **привилегии**?
- [ ] Что такое **RLS**?
- [ ] Как создать **политику RLS**?
- [ ] Что такое **default privileges**?
- [ ] Как **аудитировать** права?
- [ ] Чем **GRANT** отличается от **ALTER DEFAULT PRIVILEGES**?
- [ ] Какие риски у **SECURITY DEFINER**?
- [ ] Как **отозвать** привилегии?

### Ответы на чек-лист

1. В PostgreSQL **роль = пользователь = группа**. Разница только в атрибуте LOGIN: с LOGIN — пользователь, без — группа.

2. **Атрибуты:** LOGIN, SUPERUSER, CREATEDB, CREATEROLE, REPLICATION, BYPASSRLS, INHERIT.

3. **GRANT:** `GRANT SELECT ON table TO user;`. Можно на таблицу, схему, базу, колонку, последовательность, функцию.

4. **RLS** (Row-Level Security) — защита на уровне строк. Контролирует, какие строки видит пользователь.

5. **Политика:** `CREATE POLICY name ON table FOR operation TO role USING (condition);`. Condition — boolean expression.

6. **Default privileges** — права, автоматически назначаемые новым объектам. `ALTER DEFAULT PRIVILEGES ...`

7. **Аудит:** `information_schema.table_privileges`, `pg_roles`, `pg_auth_members`, `pg_policies`, `pg_proc`.

8. **GRANT** — на существующий объект. **ALTER DEFAULT PRIVILEGES** — на будущие объекты.

9. **SECURITY DEFINER** — функция выполняется с правами создателя. Риск: обход RLS, повышение привилегий.

10. **REVOKE:** `REVOKE SELECT ON table FROM user;`. Отзывает ранее выданные права.

## Ключевые выводы для собеседования

- **PostgreSQL — DAC-модель** — владелец решает, кто имеет доступ
- **RLS — мощный инструмент** — но не замена ACL
- **SUPERUSER — опасен** — минимум, аудит, никаких приложений
- **SECURITY DEFINER — проверять** — кто создал, с какими правами
- **Default privileges — внимательно** — не public без причины

---

_Статья создана на основе анализа материалов по PostgreSQL_
