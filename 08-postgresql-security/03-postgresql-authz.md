# Авторизация в PostgreSQL

## Роли

### Создание

`sql
CREATE ROLE alice WITH LOGIN PASSWORD 'secret';
CREATE ROLE bob WITH LOGIN PASSWORD 'secret';
CREATE ROLE managers NOLOGIN;
`

### Права

| Право | Описание |
|-------|----------|
| SELECT | Чтение |
| INSERT | Вставка |
| UPDATE | Обновление |
| DELETE | Удаление |
| TRUNCATE | Очистка |
| REFERENCES | Внешние ключи |
| TRIGGER | Триггеры |
| EXECUTE | Функции |
| USAGE | Схемы, последовательности |
| CREATE | Создание |
| CONNECT | Подключение |
| TEMPORARY | Временные таблицы |

### GRANT

`sql
GRANT SELECT, INSERT ON resumes TO alice;
GRANT managers TO alice;
GRANT USAGE ON SCHEMA public TO alice;
`

### REVOKE

`sql
REVOKE DELETE ON resumes FROM alice;
`

## ROW LEVEL SECURITY (RLS)

`sql
ALTER TABLE resumes ENABLE ROW LEVEL SECURITY;

CREATE POLICY resume_owner ON resumes
    FOR ALL
    TO users
    USING (owner = current_user);
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Least privilege | Минимум прав |
| Роли | Группировка |
| RLS | Защита строк |
| Проверка | \dp, \du |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
