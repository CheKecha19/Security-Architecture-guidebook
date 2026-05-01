# Backup и Recovery PostgreSQL

## Часть 1: Зачем нужен бэкап?

Представь дневник, который ты ведёшь всю жизнь. Дом сгорел — дневник сгорел. Но если есть копии в сейфе у друга, в банке, в облаке — дневник сохранён.

**Backup** — копии данных. **Recovery** — восстановление из копий.

## Зачем нужен бэкап?

### Ситуация 1: Удаление данных

`DROP TABLE users CASCADE;` — всё удалилось. Без бэкапа — потеряно навсегда.

### Ситуация 2: Повреждение диска

Жёсткий диск вышел из строя. Данные нечитаемы. Без бэкапа — конец.

### Ситуация 3: Ransomware

Файлы зашифрованы. Требуют выкуп. Без бэкапа — платить или потерять всё.

## Часть 2: Логический бэкап (pg_dump)

### pg_dump

    # SQL-бэкап
    pg_dump -h localhost -U postgres -d mydb > backup.sql
    
    # Сжатый формат
    pg_dump -h localhost -U postgres -d mydb -Fc -f backup.dump
    
    # Только структура
    pg_dump -h localhost -U postgres -d mydb -s > schema.sql
    
    # Только данные
    pg_dump -h localhost -U postgres -d mydb -a > data.sql
    
    # Параллельный
    pg_dump -h localhost -U postgres -d mydb -Fd -j 4 -f backup_dir/

### pg_dumpall

    # Все базы + глобальные объекты
    pg_dumpall -h localhost -U postgres > all_databases.sql

### Восстановление

    # Из SQL
    psql -h localhost -U postgres -d mydb < backup.sql
    
    # Из custom формата
    pg_restore -h localhost -U postgres -d mydb backup.dump
    
    # Параллельное восстановление
    pg_restore -h localhost -U postgres -d mydb -j 4 backup.dump

## Часть 3: Физический бэкап (pg_basebackup)

### pg_basebackup

    # Простой бэкап
    pg_basebackup -h primary -D /backup/base -U replicator -P -v
    
    # С потоковым WAL
    pg_basebackup -h primary -D /backup/base -U replicator -P -v -X stream
    
    # Сжатый
    pg_basebackup -h primary -D /backup/base -U replicator -P -v --gzip

### WAL-архивирование

    # postgresql.conf
    archive_mode = on
    archive_command = 'cp %p /backup/wal/%f'

### Восстановление (Point-in-Time Recovery)

    # Остановить PostgreSQL
    pg_ctl stop -D $PGDATA
    
    # Удалить старые данные
    rm -rf $PGDATA/*
    
    # Распаковать бэкап
    tar -xzf /backup/base.tar.gz -C $PGDATA
    
    # Настроить восстановление
    touch $PGDATA/recovery.signal
    echo "restore_command = 'cp /backup/wal/%f %p'" >> $PGDATA/postgresql.auto.conf
    echo "recovery_target_time = '2024-01-15 10:00:00'" >> $PGDATA/postgresql.auto.conf
    
    # Запустить
    pg_ctl start -D $PGDATA

## Часть 4: Шифрование бэкапов

### GPG

    pg_dump -h localhost -U postgres -d mydb | gpg --encrypt --recipient admin@example.com > backup.sql.gpg
    
    gpg --decrypt backup.sql.gpg > backup.sql

### Облачное шифрование

    aws s3 cp backup.dump s3://mybucket/backups/ --sse aws:kms --sse-kms-key-id alias/postgres-backup

## Часть 5: Автоматизация

### pgBackRest

    # Полный бэкап
    pgbackrest --stanza=mydb backup --type=full
    
    # Инкрементальный
    pgbackrest --stanza=mydb backup --type=diff
    
    # Восстановление
    pgbackrest --stanza=mydb restore

### Стратегия 3-2-1

| Правило | Описание |
|---------|----------|
| 3 копии | Основная + 2 резервные |
| 2 разных носителя | Диск + облако |
| 1 оффсайт | В другом месте |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Регулярно | Ежедневно минимум |
| Тестировать | Восстановление раз в месяц |
| Шифровать | Все бэкапы |
| Хранить | Offsite |
| WAL | Архивировать |
| Автоматизация | pgBackRest / WAL-G |
| Мониторинг | Алерты на ошибки |

## Вывод

Backup и Recovery:
1. **pg_dump** — логический, гибкий
2. **pg_basebackup** — физический, быстрый
3. **WAL** — точечное восстановление
4. **Шифрование** — защита копий
5. **Автоматизация** — pgBackRest

Правило: **если бэкап не тестировался — его нет.**

---
_Статья создана на основе анализа материалов по PostgreSQL_