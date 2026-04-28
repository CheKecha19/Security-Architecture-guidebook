# Аудит в PostgreSQL

## pgaudit

### Установка

`sql
CREATE EXTENSION pgaudit;

SET pgaudit.log = 'write, ddl';
SET pgaudit.log_relation = on;
`

### Что логировать

| Параметр | Описание |
|----------|----------|
| none | Ничего |
| all | Всё |
| read | SELECT, COPY |
| write | INSERT, UPDATE, DELETE |
| ddl | CREATE, ALTER, DROP |
| role | GRANT, REVOKE |
| function | EXECUTE |
| misc | SET, SHOW |

### Примеры логов

`
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.users,CREATE TABLE users (id serial PRIMARY KEY, name text)
AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,INSERT INTO users (name) VALUES ('alice')
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| pgaudit | Включить |
| Логи | В защищённое место |
| SIEM | Централизация |
| Ротация | Хранение по compliance |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
