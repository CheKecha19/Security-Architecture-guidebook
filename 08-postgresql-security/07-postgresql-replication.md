# Репликация и высокая доступность PostgreSQL

## Часть 1: Зачем нужна репликация?

Представь библиотеку. Один экземпляр книги — если украдут или сожгут, книги больше нет. Но если есть копии в разных филиалах — книга сохранена.

**Репликация PostgreSQL** — создание копий базы на других серверах:
- **Отказоустойчивость** — primary упал, standby работает
- **Масштабирование чтения** — читаем с реплик
- **Бэкап** — без нагрузки на primary
- **Географическая распределённость** — реплики ближе к пользователям

## Часть 2: Физическая репликация

### Streaming Replication

1. Primary пишет изменения в WAL
2. WAL передаётся на standby в реальном времени
3. Standby применяет WAL

### Настройка Primary

    # postgresql.conf
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
    max_replication_slots = 5
    
    # pg_hba.conf
    hostssl replication replicator 10.0.0.0/24 scram-sha-256

### Создание пользователя репликации

    CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'strong-password';

### Настройка Standby

    # postgresql.conf
    hot_standby = on
    
    # primary_conninfo
    primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=strong-password sslmode=require'
    
    # Для PostgreSQL 12+
    touch $PGDATA/standby.signal

### Проверка

    -- На primary
    SELECT * FROM pg_stat_replication;
    
    -- На standby
    SELECT * FROM pg_stat_wal_receiver;

## Часть 3: Логическая репликация

### Отличие от физической

| Физическая | Логическая |
|------------|------------|
| Байт-в-байт копия | Только данные |
| Все базы | Выбранные таблицы |
| DDL реплицируется | DDL не реплицируется |
| PostgreSQL → PostgreSQL | Гибче |

### Публикация и подписка

    -- На primary
    CREATE PUBLICATION users_pub FOR TABLE users;
    
    -- На standby
    CREATE SUBSCRIPTION users_sub
    CONNECTION 'host=primary port=5432 user=replicator password=strong-password sslmode=require dbname=mydb'
    PUBLICATION users_pub;

### Безопасность логической репликации

    -- Пользователь для логической репликации
    CREATE ROLE logical_replica WITH LOGIN PASSWORD 'strong-password';
    GRANT SELECT ON users TO logical_replica;

## Часть 4: Слоты репликации

### Что такое слот

Гарантирует, что WAL не удалится, пока standby не получит его.

    -- Создать слот
    SELECT pg_create_physical_replication_slot('standby_slot', true);
    
    -- Использовать в primary_conninfo
    primary_slot_name = 'standby_slot'

## Часть 5: Failover

### Ручной failover

    -- На standby
    pg_ctl promote -D /var/lib/postgresql/data
    -- Или
    SELECT pg_promote();

### Patroni (автоматический)

| Компонент | Роль |
|-----------|------|
| Patroni | Управление кластером |
| etcd/ZooKeeper | Хранение состояния |
| HAProxy/pgBouncer | Балансировка |

### Синхронная репликация

    # postgresql.conf
    synchronous_commit = remote_apply
    synchronous_standby_names = 'FIRST 1 (standby1, standby2)'

## Вывод

Репликация PostgreSQL:
1. **Физическая** — для отказоустойчивости
2. **Логическая** — для гибкости
3. **Слоты** — для гарантии доставки
4. **Failover** — ручной или автоматический
5. **Мониторинг** — lag, состояние

Без репликации — single point of failure.

---
_Статья создана на основе анализа материалов по PostgreSQL_