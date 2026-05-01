# Backup и Recovery PostgreSQL

## Часть 1: Зачем нужен бэкап?

Представь, что ты ведёшь дневник всю жизнь. Однажды дом сгорел. Дневник сгорел. Воспоминаний нет.

А теперь представь: у тебя есть копии дневника в сейфе у друга, в банковской ячейке, в облаке. Дом сгорел — ничего страшного.

**Backup** — это копии твоих данных. **Recovery** — это восстановление из копий.

## Зачем нужен бэкап?

### Ситуация 1: Удаление данных

Разработчик случайно выполнил `DROP TABLE users CASCADE;`. Всё удалилось. Без бэкапа — данные потеряны навсегда.

### Ситуация 2: Повреждение диска

Жёсткий диск вышел из строя. Данные на нём нечитаемы. Без бэкапа — конец.

### Ситуация 3: Ransomware

Злоумышленник зашифровал файлы базы данных. Требует выкуп. Без бэкапа — платить или потерять всё.

### Ситуация 4: Логическая ошибка

Приложение записало некорректные данные. Нужно откатиться на время до ошибки. Без WAL-архивирования — невозможно.

## Часть 2: Логический бэкап (pg_dump)

### Что такое логический бэкап

Экспорт данных в виде SQL-команд. Можно прочитать, отредактировать, восстановить на другую версию PostgreSQL.

### pg_dump

    # Простой SQL-бэкап
    pg_dump -h localhost -U postgres -d mydb > backup.sql
    
    # Сжатый формат (custom)
    pg_dump -h localhost -U postgres -d mydb -Fc -f backup.dump
    
    # Только структура
    pg_dump -h localhost -U postgres -d mydb -s > schema.sql
    
    # Только данные
    pg_dump -h localhost -U postgres -d mydb -a > data.sql
    
    # Определённые таблицы
    pg_dump -h localhost -U postgres -d mydb -t users -t orders > tables.dump
    
    # Параллельный бэкап (быстрее)
    pg_dump -h localhost -U postgres -d mydb -Fd -j 4 -f backup_dir/

### pg_dumpall

    # Бэкап всех баз + глобальные объекты (роли, tablespaces)
    pg_dumpall -h localhost -U postgres > all_databases.sql
    
    # Только роли и привилегии
    pg_dumpall -h localhost -U postgres --roles-only > roles.sql

### Восстановление из логического бэкапа

    # Из SQL
    psql -h localhost -U postgres -d mydb < backup.sql
    
    # Из custom формата
    pg_restore -h localhost -U postgres -d mydb backup.dump
    
    # Восстановление отдельной таблицы
    pg_restore -h localhost -U postgres -d mydb -t users backup.dump
    
    # Параллельное восстановление
    pg_restore -h localhost -U postgres -d mydb -j 4 backup.dump

## Часть 3: Физический бэкап (pg_basebackup)

### Что такое физический бэкап

Копия файлов данных PostgreSQL. Быстрее логического, но привязана к версии.

### pg_basebackup

    # Простой бэкап
    pg_basebackup -h primary -D /backup/base -U replica -P -v
    
    # С WAL-архивированием
    pg_basebackup -h primary -D /backup/base -U replica -P -v -X fetch
    
    # Потоковое WAL (лучше)
    pg_basebackup -h primary -D /backup/base -U replica -P -v -X stream
    
    # Сжатый бэкап
    pg_basebackup -h primary -D /backup/base -U replica -P -v --gzip --compress=9
    
    # Бэкап с реплики
    pg_basebackup -h standby -D /backup/base -U replica -P -v

### Настройка WAL-архивирования

    # postgresql.conf
    archive_mode = on
    archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'
    
    # Или через rsync
    archive_command = 'rsync -a %p backup-server:/backup/wal/%f'
    
    # Или в облако
    archive_command = 'aws s3 cp %p s3://mybucket/wal/%f'

### Восстановление из физического бэкапа

    # Остановить PostgreSQL
    pg_ctl stop -D /var/lib/postgresql/data
    
    # Удалить старые данные (осторожно!)
    rm -rf /var/lib/postgresql/data/*
    
    # Распаковать бэкап
    tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/data
    
    # Создать recovery.signal
    touch /var/lib/postgresql/data/recovery.signal
    
    # Настроить восстановление
    cat >> /var/lib/postgresql/data/postgresql.auto.conf << EOF
    restore_command = 'cp /backup/wal/%f %p'
    recovery_target_time = '2024-01-15 10:00:00'
    EOF
    
    # Запустить PostgreSQL
    pg_ctl start -D /var/lib/postgresql/data

## Часть 4: Шифрование бэкапов

### GPG

    # Создать ключ
    gpg --gen-key
    
    # Зашифровать бэкап
    pg_dump -h localhost -U postgres -d mydb | gpg --encrypt --recipient admin@example.com > backup.sql.gpg
    
    # Расшифровать
    gpg --decrypt backup.sql.gpg > backup.sql

### LUKS (шифрование диска)

    # Создать зашифрованный раздел для бэкапов
    cryptsetup luksFormat /dev/sdb1
    cryptsetup luksOpen /dev/sdb1 backup
    mkfs.ext4 /dev/mapper/backup
    mount /dev/mapper/backup /backup

### Облачное шифрование

| Провайдер | Решение | Управление ключами |
|-----------|---------|-------------------|
| AWS | S3 SSE-S3, SSE-KMS | AWS KMS |
| Azure | Storage Service Encryption | Azure Key Vault |
| GCP | Customer-Supplied Encryption Keys | Cloud KMS |

    # AWS S3 с шифрованием
    aws s3 cp backup.dump s3://mybucket/backups/ --sse aws:kms --sse-kms-key-id alias/postgres-backup

## Часть 5: Автоматизация и стратегии

### Стратегия 3-2-1

| Правило | Описание |
|---------|----------|
| 3 копии | Основная + 2 резервные |
| 2 разных носителя | Диск + лента/облако |
| 1 оффсайт | В другом месте |

### Инструменты автоматизации

| Инструмент | Описание |
|------------|----------|
| pgBackRest | Продвинутый бэкап и восстановление |
| Barman | Управление бэкапами для PostgreSQL |
| WAL-G | Бэкап в облако (S3, GCS, Azure) |
| pg_probackup | Форк pgBackRest от PostgresPro |

### pgBackRest

    # Установка
    sudo apt install pgbackrest
    
    # Конфигурация /etc/pgbackrest/pgbackrest.conf
    [global]
    repo1-path=/backup/pgbackrest
    repo1-retention-full=2
    repo1-retention-diff=4
    
    [mydb]
    pg1-path=/var/lib/postgresql/13/main
    
    # Полный бэкап
    pgbackrest --stanza=mydb backup --type=full
    
    # Инкрементальный бэкап
    pgbackrest --stanza=mydb backup --type=diff
    
    # Восстановление
    pgbackrest --stanza=mydb restore
    
    # Список бэкапов
    pgbackrest --stanza=mydb info

### Планирование (cron)

    # Ежедневный инкрементальный бэкап в 2:00
    0 2 * * * pgbackrest --stanza=mydb backup --type=diff
    
    # Еженедельный полный бэкап в воскресенье 3:00
    0 3 * * 0 pgbackrest --stanza=mydb backup --type=full
    
    # Проверка валидности бэкапа
    0 4 * * * pgbackrest --stanza=mydb verify

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Регулярность | Ежедневно минимум |
| Тестирование | Восстановление раз в месяц |
| Шифрование | Все бэкапы |
| Хранение | Offsite + onsite |
| Версионность | Несколько копий |
| WAL | Архивировать обязательно |
| Автоматизация | pgBackRest/WAL-G |
| Мониторинг | Алерты на ошибки бэкапа |
| Документация | Процедуры восстановления |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| Бэкап настроен и работает | ⬜ |
| WAL-архивирование включено | ⬜ |
| Бэкапы шифруются | ⬜ |
| Есть offsite-копия | ⬜ |
| Восстановление тестировалось | ⬜ |
| Автоматизация настроена | ⬜ |
| Мониторинг бэкапов | ⬜ |
| Документация по recovery | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Резервное копирование данных карт | Шифрованные бэкапы |
| SOX | Восстановление финансовых данных | Тестирование recovery |
| HIPAA | Защита PHI в бэкапах | Шифрование + доступ |
| ISO 27001 | Регулярное резервирование | Стратегия 3-2-1 |

## Вывод

Backup и Recovery — критически важны:
1. **pg_dump** — логический, гибкий
2. **pg_basebackup** — физический, быстрый
3. **WAL** — точечное восстановление
4. **Шифрование** — защита копий
5. **Автоматизация** — pgBackRest/WAL-G

Правило: **если бэкап не тестировался — его нет.**

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_