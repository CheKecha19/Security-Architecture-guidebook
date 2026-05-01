# Авторизация в PostgreSQL: роли и привилегии

## Часть 1: Что такое авторизация?

Представь офисное здание. Ты прошёл охрану (аутентификация) и получил бейдж. Но бейдж открывает не все двери. Ты можешь войти на свой этаж, но не в серверную. Не в архив. Не в кабинет директора.

**Авторизация** — это «система доступа к комнатам». После аутентификации (кто ты?) нужно решить: что тебе можно?

В PostgreSQL:
- **Аутентификация** — `pg_hba.conf` (вход в здание)
- **Авторизация** — роли и привилегии (доступ к комнатам)

## Часть 2: Роли

### Что такое роль

В PostgreSQL роль = пользователь = группа. Это единая сущность.

| Атрибут | Описание |
|---------|----------|
| LOGIN | Может подключаться (это «пользователь») |
| SUPERUSER | Все права, обходит проверки |
| CREATEDB | Может создавать базы |
| CREATEROLE | Может создавать роли |
| REPLICATION | Может делать репликацию |
| BYPASSRLS | Обходит Row-Level Security |
| INHERIT | Наследует права групп (по умолчанию) |

### Создание ролей

    -- Пользователь (с LOGIN)
    CREATE ROLE app_user WITH LOGIN PASSWORD 'strong-password';
    
    -- Групповая роль (без LOGIN)
    CREATE ROLE developers NOLOGIN;
    
    -- Суперпользователь (осторожно!)
    CREATE ROLE admin WITH LOGIN PASSWORD 'very-strong' SUPERUSER;
    
    -- Проверить
    SELECT rolname, rolsuper, rolcreaterole, rolcreatedb 
    FROM pg_roles;

### Наследование

    -- Добавить пользователя в группу
    GRANT developers TO app_user;
    
    -- Теперь app_user имеет права developers
    -- Проверить членство
    SELECT r.rolname AS role, m.rolname AS member
    FROM pg_auth_members am
    JOIN pg_roles r ON am.roleid = r.oid
    JOIN pg_roles m ON am.member = m.oid;

## Часть 3: Привилегии

### Уровни привилегий

| Уровень | Объект | Примеры |
|---------|--------|---------|
| База данных | База | CONNECT, CREATE, TEMP |
| Схема | Schema | USAGE, CREATE |
| Таблица | Table | SELECT, INSERT, UPDATE, DELETE, TRUNCATE |
| Колонка | Column | SELECT, UPDATE (на колонку) |
| Последовательность | Sequence | USAGE, SELECT, UPDATE |
| Функция | Function | EXECUTE |

### Назначение привилегий

    -- Дать права на базу
    GRANT CONNECT ON DATABASE mydb TO app_user;
    
    -- Дать права на схему
    GRANT USAGE ON SCHEMA public TO app_user;
    GRANT CREATE ON SCHEMA public TO admin;
    
    -- Дать права на таблицу
    GRANT SELECT, INSERT ON orders TO app_user;
    GRANT ALL ON orders TO admin;
    
    -- Права на колонки
    GRANT SELECT (id, name, email) ON users TO analyst;
    
    -- Права по умолчанию для новых объектов
    ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO analyst;

### Отзыв привилегий

    REVOKE DELETE ON orders FROM app_user;
    REVOKE ALL ON orders FROM app_user;

## Часть 4: Row-Level Security (RLS)

### Зачем нужен RLS

Представь HR-систему. Менеджер видит все резюме. Кандидат — только своё. Рекрутер — назначенные ему.

RLS позволяет контролировать доступ к **строкам** таблицы.

### Включение RLS

    ALTER TABLE resumes ENABLE ROW LEVEL SECURITY;
    
    -- Проверить
    SELECT relname, relrowsecurity FROM pg_class WHERE relname = 'resumes';

### Создание политик

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

### Функции для RLS

    CREATE OR REPLACE FUNCTION current_user_id() RETURNS INT AS $$
    BEGIN
        RETURN (SELECT id FROM users WHERE username = current_user);
    END;
    $$ LANGUAGE plpgsql SECURITY DEFINER;

### Обход RLS

**Внимание:** SUPERUSER и BYPASSRLS обходят RLS!

    -- Проверить, кто обходит
    SELECT rolname, rolbypassrls FROM pg_roles WHERE rolbypassrls = true;
    
    -- Опасно:
    ALTER ROLE app_user BYPASSRLS;  -- Только если необходимо!

## Часть 5: Практические примеры

### Пример 1: Веб-приложение

    -- Роль для приложения
    CREATE ROLE webapp WITH LOGIN PASSWORD 'secret';
    
    -- Права
    GRANT CONNECT ON DATABASE shop TO webapp;
    GRANT USAGE ON SCHEMA public TO webapp;
    GRANT SELECT, INSERT, UPDATE ON products, orders, order_items TO webapp;
    GRANT SELECT, INSERT ON customers TO webapp;
    GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO webapp;
    
    -- Нет права:
    -- DELETE (soft delete через UPDATE status)
    -- CREATE TABLE
    -- DROP TABLE

### Пример 2: Аналитик

    CREATE ROLE analyst WITH LOGIN PASSWORD 'secret';
    
    -- Только чтение, только определённые колонки
    GRANT SELECT (id, name, category, price) ON products TO analyst;
    GRANT SELECT (id, total, status, created_at) ON orders TO analyst;
    -- Нет: customer_details, payment_info

### Пример 3: Администратор

    CREATE ROLE dbadmin WITH LOGIN PASSWORD 'very-secret';
    GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO dbadmin;
    GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO dbadmin;

## Часть 6: Проверка привилегий

    -- Кто имеет права на таблицу
    SELECT grantee, privilege_type
    FROM information_schema.table_privileges
    WHERE table_name = 'orders';
    
    -- Роли и их атрибуты
    SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin
    FROM pg_roles
    WHERE rolname NOT LIKE 'pg_%';
    
    -- Политики RLS
    SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual
    FROM pg_policies;

## Вывод

Авторизация в PostgreSQL:
1. **Роли** — пользователи и группы
2. **Привилегии** — на базы, схемы, таблицы, колонки
3. **RLS** — защита на уровне строк
4. **Принцип** — минимальные привилегии
5. **Аудит** — регулярная проверка прав

Без авторизации — все имеют доступ ко всему. Это как бейдж, открывающий все двери.

---
_Статья создана на основе анализа материалов по PostgreSQL_