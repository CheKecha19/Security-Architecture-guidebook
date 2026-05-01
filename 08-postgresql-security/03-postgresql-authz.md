# Авторизация в PostgreSQL

## Часть 1: Что такое авторизация в PostgreSQL?

Представь, что ты прошёл мимо охранника (аутентификация) и вошёл в офисное здание. Но это не значит, что ты можешь войти в любой кабинет. У тебя есть пропуск на 3-й этаж, но не на 5-й. Ты можешь войти в переговорную, но не в серверную.

**Аутентификация** — это «кто ты?» **Авторизация** — это «что тебе можно?»

В PostgreSQL аутентификация решается в `pg_hba.conf` (вход в здание), а авторизация — системой ролей и привилегий (доступ к комнатам).

## Зачем нужна авторизация?

### Ситуация 1: Разделение прав разработчиков

Разработчик Алиса работает с таблицей `resumes`. Она должна читать и редактировать записи. Но она не должна:
- Удалять таблицу
- Менять структуру
- Видеть таблицу `salaries`
- Видеть чужие резюме (только свои)

### Ситуация 2: Приложение с ограниченными правами

Приложение подключается к базе с пользователем `app_user`. Этот пользователь:
- Может читать и писать в рабочие таблицы
- Не может создавать/удалять таблицы
- Не может выполнять DDL
- Не может читать системные таблицы

### Ситуация 3: Compliance (PCI DSS)

Стандарт требует: доступ к данным карт только тем, кому нужно для работы. Кассир может создавать транзакции, но не видеть полные номера карт. Аналитик видит обезличенные данные, но не может создавать транзакции.

## Часть 2: Система ролей

### Роли vs Пользователи

В PostgreSQL нет различия между «пользователем» и «ролью». Технически это одно и то же: `CREATE ROLE` с флагом `LOGIN` — это пользователь, без `LOGIN` — групповая роль.

| Тип | Описание | Пример |
|-----|----------|--------|
| Пользователь | Роль с LOGIN | `alice`, `app_user` |
| Групповая роль | Роль без LOGIN | `managers`, `developers` |
| Суперпользователь | Роль с SUPERUSER | `postgres` |

### Создание и настройка ролей

    -- Пользователь для приложения
    CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password';
    
    -- Групповая роль для менеджеров
    CREATE ROLE managers NOLOGIN;
    
    -- Суперпользователь (осторожно!)
    CREATE ROLE admin WITH LOGIN PASSWORD 'admin_pass' SUPERUSER;

### Атрибуты ролей

| Атрибут | Описание | Риск |
|---------|----------|------|
| LOGIN | Может подключаться | — |
| SUPERUSER | Все права | Критический |
| CREATEDB | Создавать базы | Высокий |
| CREATEROLE | Создавать роли | Высокий |
| REPLICATION | Репликация | Высокий |
| BYPASSRLS | Обход RLS | Высокий |
| INHERIT | Наследовать права группы | Средний |

### Наследование ролей

    -- Алиса входит в группу managers
    GRANT managers TO alice;
    
    -- Теперь Алиса имеет все права группы managers
    -- Проверить:
    SELECT rolname, member, grantor
    FROM pg_auth_members
    JOIN pg_roles ON pg_roles.oid = pg_auth_members.roleid;

**Важно:** `INHERIT` по умолчанию включён. Если отключить (`NOINHERIT`), пользователь должен явно активировать роль через `SET ROLE`.

## Часть 3: Привилегии

### Уровни привилегий

PostgreSQL работает с привилегиями на четырёх уровнях:

| Уровень | Что контролируется | Примеры команд |
|---------|-------------------|----------------|
| База данных | Подключение, создание схем | CONNECT, CREATE, TEMPORARY |
| Схема | Использование, создание объектов | USAGE, CREATE |
| Таблица/представление | Доступ к данным | SELECT, INSERT, UPDATE, DELETE |
| Колонка | Доступ к конкретным колонкам | SELECT (колонка), UPDATE (колонка) |

### Таблица привилегий

| Привилегия | Описание | Пример риска |
|------------|----------|--------------|
| SELECT | Читать данные | Чтение чужих записей |
| INSERT | Добавлять строки | Заполнение мусором |
| UPDATE | Изменять строки | Подмена данных |
| DELETE | Удалять строки | Потеря данных |
| TRUNCATE | Очистить таблицу полностью | Полная потеря |
| REFERENCES | Создавать внешние ключи | Связывание таблиц |
| TRIGGER | Создавать триггеры | Скрытая логика |
| EXECUTE | Выполнять функции | Выполнение произвольного кода |
| USAGE | Использовать схемы/последовательности | Доступ к объектам |
| CREATE | Создавать объекты | Расширение поверхности атаки |
| ALL | Все права | Полный доступ |

### Назначение привилегий

    -- Дать Алисе права на чтение таблицы
    GRANT SELECT ON resumes TO alice;
    
    -- Дать право на чтение и вставку
    GRANT SELECT, INSERT ON resumes TO alice;
    
    -- Дать все права
    GRANT ALL ON resumes TO alice;
    
    -- Дать право на определённые колонки
    GRANT SELECT (id, name, email) ON users TO analyst;
    
    -- Отозвать права
    REVOKE DELETE ON resumes FROM alice;
    
    -- Запретить передачу прав другим
    GRANT SELECT ON resumes TO alice WITH GRANT OPTION;
    REVOKE GRANT OPTION FOR SELECT ON resumes FROM alice;

### Привилегии по умолчанию

    -- Установить права по умолчанию для новых таблиц
    ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT ON TABLES TO app_user;
    
    -- Проверить текущие привилегии
    SELECT * FROM information_schema.table_privileges
    WHERE grantee = 'alice';

## Часть 4: Row-Level Security (RLS)

### Зачем нужен RLS?

Представь HR-систему. Менеджер по найму должен видеть все резюме. Но кандидат должен видеть только своё резюме. Рекрутер — только назначенные ему. Обычные GRANT/REVOKE не решают эту задачу: они работают на уровне таблиц.

**Row-Level Security** позволяет контролировать доступ к строкам на основе политик.

### Включение RLS

    ALTER TABLE resumes ENABLE ROW LEVEL SECURITY;
    
    -- Проверить статус
    SELECT relname, relrowsecurity
    FROM pg_class
    WHERE relname = 'resumes';

### Создание политик

    -- Политика: пользователь видит только свои резюме
    CREATE POLICY resume_owner ON resumes
        FOR ALL
        TO public
        USING (owner_id = current_user_id());
    
    -- Политика: менеджеры видят всё
    CREATE POLICY resume_managers ON resumes
        FOR ALL
        TO managers
        USING (true);
    
    -- Политика только для SELECT
    CREATE POLICY resume_select ON resumes
        FOR SELECT
        TO analysts
        USING (status = 'published');

### Функции для RLS

    -- Функция: получить ID текущего пользователя
    CREATE OR REPLACE FUNCTION current_user_id() RETURNS INT AS $$
    BEGIN
        RETURN (SELECT id FROM users WHERE username = current_user);
    END;
    $$ LANGUAGE plpgsql SECURITY DEFINER;
    
    -- Политика с использованием функции
    CREATE POLICY resume_user ON resumes
        FOR ALL
        USING (user_id = current_user_id());

### Обход RLS

**Внимание:** Суперпользователи (`SUPERUSER`) и роли с `BYPASSRLS` обходят RLS!

    -- Проверить, кто обходит RLS
    SELECT rolname, rolbypassrls
    FROM pg_roles
    WHERE rolbypassrls = true;
    
    -- Опасно: дать обход RLS
    ALTER ROLE alice BYPASSRLS;  -- Только если необходимо!

## Часть 5: Лучшие практики и аудит

### Принцип наименьших привилегий

| Плохо | Хорошо |
|-------|--------|
| `GRANT ALL ON ALL TABLES TO app_user;` | `GRANT SELECT, INSERT ON specific_table TO app_user;` |
| Суперпользователь для приложения | Отдельная роль с минимальными правами |
| `public` имеет права на схемы | Отзывать права у `public` |

### Отзыв прав у PUBLIC

    -- По умолчанию PUBLIC имеет права на схему public
    REVOKE ALL ON SCHEMA public FROM PUBLIC;
    
    -- Дать права только нужным ролям
    GRANT USAGE ON SCHEMA public TO app_user;
    GRANT CREATE ON SCHEMA public TO admin;

### Аудит привилегий

    -- Кто имеет права на таблицу
    SELECT grantee, privilege_type
    FROM information_schema.table_privileges
    WHERE table_name = 'resumes';
    
    -- Какие роли существуют
    SELECT rolname, rolsuper, rolcreaterole, rolcreatedb
    FROM pg_roles
    WHERE rolname NOT LIKE 'pg_%';
    
    -- Кто наследует какие роли
    SELECT r.rolname AS role, m.rolname AS member
    FROM pg_auth_members
    JOIN pg_roles r ON roleid = r.oid
    JOIN pg_roles m ON member = m.oid;

### Чек-лист безопасности

| Проверка | Статус |
|----------|--------|
| Нет суперпользователей для приложений | ⬜ |
| Права `public` отозваны | ⬜ |
| RLS включён где нужно | ⬜ |
| Нет `BYPASSRLS` без необходимости | ⬜ |
| Привилегии на уровне колонок где нужно | ⬜ |
| Регулярный аудит ролей | ⬜ |
| Ротация паролей | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Ролевая модель, минимальные привилегии | Роли, GRANT/REVOKE |
| SOX | Контроль доступа, аудит | RLS, логирование |
| GDPR | Ограничение доступа к персональным данным | RLS, колоночные привилегии |
| HIPAA | Аудит доступа к PHI | Аудит, RLS |

## Вывод

Авторизация в PostgreSQL — гибкая система от ролей до строк. Ключевые принципы:
1. **Минимальные привилегии** — только то, что нужно
2. **Групповые роли** — управлять правами централизованно
3. **RLS** — защита на уровне строк
4. **Аудит** — регулярная проверка

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_