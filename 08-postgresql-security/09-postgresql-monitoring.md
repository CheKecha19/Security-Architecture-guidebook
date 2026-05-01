# Мониторинг PostgreSQL

## Что такое мониторинг?

Представь электростанцию без приборов. Не знаешь температуру, давление, радиацию. Превышение — авария.

**Мониторинг PostgreSQL** — приборная панель для базы данных. Здорова ли база? Не перегружается ли? Нет ли проблем?

## Зачем это нужно?

### Сценарий 1: Медленные запросы

Пользователи жалуются: «Сайт тормозит». Без мониторинга — непонятно почему. С мониторингом — видно: один запрос занимает 30 секунд, блокирует таблицу.

### Сценарий 2: Заканчивается место

Диск заполнен на 95%. Без мониторинга — база падает. С мониторингом — алерт при 80%, время на реакцию.

### Сценарий 3: Подозрительная активность

В 3 часа ночи — необычно много запросов. Мониторинг показывает: кто и откуда подключался.

## Основные концепции

### pg_stat_activity

```sql
-- Текущие подключения
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity WHERE state != 'idle';

-- Долгие запросы
SELECT pid, usename, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 seconds';

-- Блокировки
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### pg_stat_statements

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Самые медленные запросы
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- Самые частые
SELECT query, calls FROM pg_stat_statements ORDER BY calls DESC LIMIT 10;
```

### pg_stat_database

```sql
SELECT datname, numbackends, xact_commit, xact_rollback,
       blks_read, blks_hit,
       round(blks_hit::numeric/(blks_hit + blks_read) * 100, 2) as cache_hit_ratio
FROM pg_stat_database;
```

### Инструменты

| Инструмент | Тип |
|------------|-----|
| **Prometheus + Grafana** | Open source |
| **pgAdmin** | Бесплатный GUI |
| **pganalyze** | Коммерческий |

### Алерты

| Условие | Приоритет |
|---------|-----------|
| connections > 80% | Critical |
| disk_space > 85% | Critical |
| cache_hit_ratio < 90% | Warning |
| failed_logins > 5 за 5 мин | High |
| query_time_avg > 1s | Warning |

## Уроки из инцидентов

### Инцидент 1: Нет мониторинга lag (2019)

Репликация отстала на 2 часа. Никто не заметил. При failover — потеря 2 часов данных.

**Решение:** Мониторинг `pg_stat_replication.replay_lag`.

### Инцидент 2: Нет мониторинга deadlocks (2020)

Deadlock каждые 5 минут. Приложение падает. Причина — race condition в коде.

**Решение:** Мониторинг `pg_stat_database.deadlocks`.

## Чек-лист понимания

- [ ] Что такое **pg_stat_activity**?
- [ ] Что такое **pg_stat_statements**?
- [ ] Как найти **блокировки**?
- [ ] Какие **метрики** мониторить?
- [ ] Какие **инструменты** использовать?
- [ ] Как настроить **алерты**?
- [ ] Что такое **cache_hit_ratio**?
- [ ] Как мониторить **replication lag**?
- [ ] Как мониторить **deadlocks**?
- [ ] Как централизовать **логи**?

## Выводы

- **Мониторинг — обязателен** — иначе слепота
- **pg_stat_statements** — медленные и частые запросы
- **pg_stat_activity** — текущие подключения
- **Алерты** — на критичные метрики
- **SIEM** — централизация и анализ

---

_Статья создана на основе анализа материалов по PostgreSQL_
