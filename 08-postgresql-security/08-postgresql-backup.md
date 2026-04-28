# Backup и Recovery PostgreSQL

## pg_dump

`ash
pg_dump -h host -U user -d mydb -f backup.sql
pg_dump -h host -U user -d mydb -Fc -f backup.dump
`

## pg_basebackup

`ash
pg_basebackup -h primary -D /var/lib/postgresql/standby -U replica -P -X stream
`

## WAL Archiving

`
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'
`

## Шифрование backup

| Способ | Описание |
|--------|----------|
| GPG | gpg --encrypt |
| OS | LUKS |
| Cloud | S3 encryption |

## Recovery

`
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2024-01-01 00:00:00'
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Регулярно | Ежедневно |
| Тестировать | Восстановление |
| Шифровать | Backup |
| Хранить | Offsite |
| WAL | Архивировать |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
