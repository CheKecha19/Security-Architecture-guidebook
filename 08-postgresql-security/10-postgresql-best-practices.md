# Лучшие практики PostgreSQL

## Чек-лист

### Аутентификация

- [ ] scram-sha-256
- [ ] hostssl
- [ ] Ограничение сети
- [ ] Не trust

### Авторизация

- [ ] Least privilege
- [ ] Роли
- [ ] RLS для sensitive
- [ ] Регулярный аудит

### Шифрование

- [ ] TLS 1.3
- [ ] pgcrypto для columns
- [ ] OS-level для диска
- [ ] KMS для ключей

### Аудит

- [ ] pgaudit
- [ ] log_connections
- [ ] log_disconnections
- [ ] SIEM

### Hardening

- [ ] CIS Benchmarks
- [ ] Минимум портов
- [ ] Обновление
- [ ] Backup

### Мониторинг

- [ ] pg_stat_statements
- [ ] Replication lag
- [ ] Deadlocks
- [ ] Алерты

## Архитектура

`
Client → SSL → pg_hba.conf → scram-sha-256 → Role → ACL → Table → RLS → Row
`

## Итог

| Уровень | Защита |
|---------|--------|
| Сеть | SSL, firewall |
| Аутентификация | scram-sha-256 |
| Авторизация | GRANT, RLS |
| Шифрование | TLS, pgcrypto |
| Аудит | pgaudit, логи |
| Backup | Шифрованный, offsite |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
