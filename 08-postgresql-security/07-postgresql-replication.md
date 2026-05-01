# Репликация и высокая доступность PostgreSQL

## Что такое репликация?

Представь библиотеку. Один экземпляр книги — если украдут, книги больше нет. Но если есть копии в разных филиалах — книга сохранена.

**Репликация PostgreSQL** — создание копий базы на других серверах. Зачем: отказоустойчивость, масштабирование чтения, бэкап без нагрузки, географическая распределённость.

## Зачем это нужно?

### Сценарий 1: Отказоустойчивость

Primary сервер падает. Без репликации — downtime до восстановления из backup. С репликацией — standby становится primary за секунды.

### Сценарий 2: Масштабирование чтения

Приложение читает данные чаще, чем пишет. Primary перегружен. С репликацией — чтение направляется на standby.

### Сценарий 3: Географическая распределённость

Пользователи в Европе и Азии. Primary в Европе. Азиатские пользователи испытывают latency. С репликой в Азии — чтение локально.

## Основные концепции

### Физическая репликация

**Streaming Replication:** primary пишет WAL, standby применяет.

**Настройка primary:**
```
# postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
max_replication_slots = 5

# pg_hba.conf
hostssl replication replicator 10.0.0.0/24 scram-sha-256
```

**Пользователь репликации:**
```sql
CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'strong-password';
```

**Настройка standby:**
```
# primary_conninfo
primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=strong-password sslmode=require'

# PostgreSQL 12+
touch $PGDATA/standby.signal
```

**Проверка:**
```sql
-- На primary
SELECT * FROM pg_stat_replication;

-- На standby
SELECT * FROM pg_stat_wal_receiver;
```

### Логическая репликация

**Публикация и подписка:**
```sql
-- На primary
CREATE PUBLICATION users_pub FOR TABLE users;

-- На standby
CREATE SUBSCRIPTION users_sub
CONNECTION 'host=primary port=5432 user=replicator password=strong-password sslmode=require dbname=mydb'
PUBLICATION users_pub;
```

**Отличие от физической:**

| Физическая | Логическая |
|------------|------------|
| Байт-в-байт | Только данные |
| Все базы | Выбранные таблицы |
| DDL реплицируется | DDL не реплицируется |
| PostgreSQL → PostgreSQL | Гибче |

### Слоты репликации

**Гарантируют:** WAL не удалится, пока standby не получит.

```sql
-- Создать слот
SELECT pg_create_physical_replication_slot('standby_slot', true);

-- Использовать
primary_slot_name = 'standby_slot'
```

### Failover

**Ручной:**
```
pg_ctl promote -D /var/lib/postgresql/data
-- или
SELECT pg_promote();
```

**Автоматический (Patroni):**

| Компонент | Роль |
|-----------|------|
| Patroni | Управление кластером |
| etcd/ZooKeeper | Хранение состояния |
| HAProxy/pgBouncer | Балансировка |

### Синхронная репликация

```
# postgresql.conf
synchronous_commit = remote_apply
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
```

## Уроки из инцидентов

### Инцидент 1: Рассинхронизация реплик (2019)

Primary и standby потеряли связь. WAL на primary удалился (retention). Standby не смог догнать. При failover — потеря данных.

**Решение:** Увеличить `wal_keep_size`. Использовать replication slots.

### Инцидент 2: Неправильный failover (2020)

Администратор промоутировал standby вручную, не проверив синхронизацию. Данные на standby были на 2 часа старше. Потеря транзакций.

**Решение:** Проверять `pg_stat_replication` перед failover. Использовать Patroni.

## Чек-лист понимания

- [ ] Что такое **WAL**?
- [ ] Как работает **streaming replication**?
- [ ] Чем **физическая** отличается от **логической**?
- [ ] Что такое **слот репликации**?
- [ ] Как выполнить **failover**?
- [ ] Что такое **synchronous_commit**?
- [ ] Как **мониторить** репликацию?
- [ ] Как **Patroni** работает?
- [ ] Какие **риски** у репликации?
- [ ] Как **защитить** репликацию?

## Выводы

- **Репликация — обязательна** для production
- **Physical** — для DR, **Logical** — для гибкости
- **Slots** — для гарантии доставки
- **Patroni** — для автоматического failover
- **Мониторинг** — lag, состояние, ошибки

---

_Статья создана на основе анализа материалов по PostgreSQL_
