# Репликация PostgreSQL

## Часть 1: Зачем нужна репликация?

Представь библиотеку. Один экземпляр книги — если её украдут или сожгут, книги больше нет. Но если есть копии в разных филиалах — книга сохранена.

**Репликация PostgreSQL** — создание копий базы данных на других серверах. Зачем:
- **Отказоустойчивость** — если primary упал, standby продолжает работать
- **Масштабирование чтения** — читаем с реплик, снижаем нагрузку на primary
- **Резервное копирование** — бэкап с реплики не нагружает primary
- **Географическая распределённость** — реплики ближе к пользователям

## Часть 2: Физическая репликация (Streaming Replication)

### Как работает

1. Primary пишет изменения в WAL (Write-Ahead Log)
2. WAL передаётся на standby в реальном времени
3. Standby применяет WAL, воспроизводя изменения

### Настройка Primary

    # postgresql.conf
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
    max_replication_slots = 5
    
    # pg_hba.conf
    hostssl replication replica 10.0.0.0/24 scram-sha-256

### Создание пользователя репликации

    CREATE ROLE replica WITH LOGIN REPLICATION PASSWORD 'strong-password';
    
    -- Проверить права
    SELECT rolname, rolreplication FROM pg_roles WHERE rolname = 'replica';

### Настройка Standby

    # postgresql.conf
    hot_standby = on
    
    # recovery.conf (до PostgreSQL 12) или standby.signal (PostgreSQL 12+)
    # primary_conninfo
    primary_conninfo = 'host=primary.example.com port=5432 user=replica password=strong-password sslmode=require'
    
    # Создать standby.signal для PostgreSQL 12+
    touch $PGDATA/standby.signal

### Проверка репликации

    -- На primary: состояние репликации
    SELECT 
        client_addr,
        state,
        sent_lsn,
        write_lsn,
        flush_lsn,
        replay_lsn,
        write_lag,
        flush_lag,
        replay_lag
    FROM pg_stat_replication;
    
    -- На standby: состояние воспроизведения
    SELECT 
        status,
        receive_start_lsn,
        received_lsn,
        latest_end_lsn,
        last_msg_send_time,
        last_msg_receipt_time
    FROM pg_stat_wal_receiver;

### Рекомендации по безопасности

| Практика | Реализация |
|----------|------------|
| Отдельный пользователь | `CREATE ROLE replica REPLICATION` |
| SSL обязателен | `sslmode=require` в primary_conninfo |
| Ограничение IP | pg_hba.conf: только подсеть standby |
| scram-sha-256 | Метод аутентификации |
| Минимум прав | Только REPLICATION, не SUPERUSER |

## Часть 3: Логическая репликация

### Отличие от физической

| Физическая | Логическая |
|------------|------------|
| Байт-в-байт копия | Только данные, не структура |
| Все базы | Выбранные таблицы |
| Нельзя писать на standby | Можно (конфликты) |
| PostgreSQL → PostgreSQL | PostgreSQL → PostgreSQL, но гибче |
| DDL реплицируется | DDL не реплицируется |

### Настройка публикации

    -- На primary
    CREATE PUBLICATION users_pub FOR TABLE users;
    
    -- Публикация всех таблиц
    CREATE PUBLICATION all_pub FOR ALL TABLES;
    
    -- Публикация с фильтром (PostgreSQL 15+)
    CREATE PUBLICATION filtered_pub FOR TABLE users WHERE (status = 'active');
    
    -- Просмотр публикаций
    SELECT * FROM pg_publication;

### Настройка подписки

    -- На standby
    CREATE SUBSCRIPTION users_sub
    CONNECTION 'host=primary.example.com port=5432 user=replica password=strong-password sslmode=require dbname=mydb'
    PUBLICATION users_pub;
    
    -- Проверить статус
    SELECT * FROM pg_subscription;
    SELECT * FROM pg_stat_subscription;

### Безопасность логической репликации

    -- Пользователь для логической репликации
    CREATE ROLE logical_replica WITH LOGIN PASSWORD 'strong-password';
    GRANT SELECT ON users TO logical_replica;
    
    -- REPLICATION не нужен для логической репликации
    -- Нужны права на чтение таблиц
    
    -- Ограничить в publication
    CREATE PUBLICATION secure_pub FOR TABLE users WITH (publish = 'insert, update, delete');

## Часть 4: Слоты репликации

### Что такое слот?

Слот гарантирует, что WAL не будет удалён, пока standby его не получит.

    -- Создать слот
    SELECT pg_create_physical_replication_slot('standby_slot', true);
    
    -- Использовать в primary_conninfo
    primary_slot_name = 'standby_slot'
    
    -- Логические слоты
    SELECT pg_create_logical_replication_slot('logical_slot', 'pgoutput');
    
    -- Удалить слот
    SELECT pg_drop_replication_slot('standby_slot');

### Мониторинг слотов

    SELECT 
        slot_name,
        plugin,
        slot_type,
        database,
        active,
        restart_lsn,
        confirmed_flush_lsn
    FROM pg_replication_slots;

## Часть 5: Failover и высокая доступность

### Ручной failover

    -- На standby: промоут в primary
    pg_ctl promote -D /var/lib/postgresql/data
    
    -- Или через SQL (PostgreSQL 12+)
    SELECT pg_promote();

### Автоматический failover (Patroni)

**Patroni** — популярное решение для автоматического failover:

| Компонент | Роль |
|-----------|------|
| Patroni | Управление кластером |
| etcd/ZooKeeper/Kubernetes | Хранение состояния |
| HAProxy/pgBouncer | Балансировка |

### Архитектура высокой доступности

    [Application]
         |
    [HAProxy / pgBouncer]
         |
    [Primary]  <--streaming--  [Standby 1]
         |                       [Standby 2]
         |                       [Standby 3]
    [Synchronous standby]

### Синхронная репликация

    # postgresql.conf
    synchronous_commit = remote_apply
    synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
    
    -- Гарантия: транзакция подтверждена только после записи на standby
    -- Риск: задержка, но нет потери данных

## Лучшие практики

| Практика | Описание |
|----------|----------|
| SSL | Обязательно для репликации |
| Отдельный пользователь | REPLICATION только для репликации |
| Мониторинг лага | pg_stat_replication.replay_lag |
| Автоматический failover | Patroni или аналог |
| Регулярные проверки | Переключение primary/standby |
| Бэкап с standby | pg_basebackup с реплики |
| Слоты | Для гарантии получения WAL |
| Разные дата-центры | Географическая избыточность |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| SSL на репликации | ⬜ |
| Отдельный пользователь replica | ⬜ |
| Мониторинг лага | ⬜ |
| Автоматический failover настроен | ⬜ |
| Слоты созданы | ⬜ |
| Реплика в другом ДЦ | ⬜ |
| Тест failover регулярно | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Высокая доступность | Репликация + failover |
| SOX | Защита от потери данных | Синхронная репликация |
| HIPAA | Отказоустойчивость | HA кластер |
| ISO 27001 | Резервирование | Репликация в разных ДЦ |

## Вывод

Репликация — обязательный элемент production:
1. **Физическая** — для отказоустойчивости
2. **Логическая** — для гибкости и миграций
3. **Слоты** — для гарантии доставки
4. **Failover** — для автоматизации

Без репликации — single point of failure.

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_