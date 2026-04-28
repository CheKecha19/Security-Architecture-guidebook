# Hardening PostgreSQL

## CIS Benchmarks

| Пункт | Описание |
|-------|----------|
| 1.1 | Установка из доверенных источников |
| 2.1 | Отключить listen_addresses если не нужно |
| 2.2 | SSL включён |
| 3.1 | Superuser только для администрирования |
| 3.2 | Пароли не в командной строке |
| 4.1 | pgaudit включён |
| 5.1 | log_connections |
| 5.2 | log_disconnections |

## Конфигурация

`
listen_addresses = 'localhost,10.0.0.1'
port = 5432
ssl = on
password_encryption = scram-sha-256
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
shared_preload_libraries = 'pgaudit'
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Минимум портов | Не слушать все |
| SSL | Обязательно |
| Superuser | Ограничить |
| Логи | Всё включить |
| Обновление | Последняя версия |
| Backup | Регулярно |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
