# Backup и Recovery PostgreSQL

## Что такое бэкап?

Представь дневник, который ты ведёшь всю жизнь. Дом сгорел — дневник сгорел. Но если есть копии в сейфе — дневник сохранён.

**Backup** — копии данных. **Recovery** — восстановление из копий.

## Зачем это нужно?

### Сценарий 1: Удаление данных

`DROP TABLE users CASCADE;` — всё удалилось. Без бэкапа — потеряно навсегда.

### Сценарий 2: Повреждение диска

Жёсткий диск вышел из строя. Данные нечитаемы. Без бэкапа — конец.

### Сценарий 3: Ransomware

Файлы зашифрованы. Требуют выкуп. Без бэкапа — платить или потерять всё.

## Основные концепции

### Логический бэкап (pg_dump)

```bash
# SQL-бэкап
pg_dump -h localhost -U postgres -d mydb > backup.sql

# Сжатый формат
pg_dump -h localhost -U postgres -d mydb -Fc -f backup.dump

# Восстановление
pg_restore -h localhost -U postgres -d mydb backup.dump
```

### Физический бэкап (pg_basebackup)

```bash
# Бэкап
pg_basebackup -h primary -D /backup/base -U replicator -P -v

# WAL-архивирование
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'
```

### Point-in-Time Recovery

```bash
# recovery.signal
touch $PGDATA/recovery.signal

# Настройка восстановления
echo "restore_command = 'cp /backup/wal/%f %p'" >> $PGDATA/postgresql.auto.conf
echo "recovery_target_time = '2024-01-15 10:00:00'" >> $PGDATA/postgresql.auto.conf
```

### Шифрование бэкапов

```bash
# GPG
pg_dump -h localhost -U postgres -d mydb | gpg --encrypt --recipient admin@example.com > backup.sql.gpg

# Облако
aws s3 cp backup.dump s3://mybucket/backups/ --sse aws:kms
```

### Автоматизация (pgBackRest)

```bash
# Полный бэкап
pgbackrest --stanza=mydb backup --type=full

# Инкрементальный
pgbackrest --stanza=mydb backup --type=diff
```

### Стратегия 3-2-1

| Правило | Описание |
|---------|----------|
| 3 копии | Основная + 2 резервные |
| 2 разных носителя | Диск + облако |
| 1 оффсайт | В другом месте |

## Уроки из инцидентов

### Инцидент 1: Незашифрованный backup (2019)

Backup на NFS без шифрования. Злоумышленник получил доступ — скачал backup. Все данные открытым текстом.

**Решение:** Шифрование backup. Ограничение доступа.

### Инцидент 2: Непроверенное восстановление (2020)

Backup делался, но восстановление не тестировалось. При инциденте — backup оказался corrupted.

**Решение:** Регулярное тестирование восстановления.

## Чек-лист понимания

- [ ] Что такое **pg_dump**?
- [ ] Что такое **pg_basebackup**?
- [ ] Что такое **WAL-архивирование**?
- [ ] Как **восстановить** базу?
- [ ] Что такое **Point-in-Time Recovery**?
- [ ] Как **шифровать** backup?
- [ ] Что такое **pgBackRest**?
- [ ] Какая **стратегия** 3-2-1?
- [ ] Как **тестировать** backup?
- [ ] Как **автоматизировать**?

## Выводы

- **Backup — обязателен** — pg_dump + pg_basebackup
- **WAL** — для point-in-time recovery
- **Шифрование** — защита копий
- **Тестирование** — если не тестировал — backup не работает
- **3-2-1** — минимальная стратегия

---

_Статья создана на основе анализа материалов по PostgreSQL_
