# Мониторинг PostgreSQL

## Часть 1: Зачем мониторить?

Представь электростанцию без приборов. Не знаешь температуру, давление, радиацию. Превышение — авария.

**Мониторинг PostgreSQL** — приборная панель для базы данных. Здорова ли база? Не перегружается ли? Нет ли проблем?

## Часть 2: Встроенная статистика

### pg_stat_activity

    -- Текущие подключения
    SELECT pid, usename, application_name, client_addr, state, query
    FROM pg_stat_activity WHERE state != 'idle';
    
    -- Долгие запросы (> 5 сек)
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

### pg_stat_statements

    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    
    -- Самые медленные запросы
    SELECT query, calls, mean_exec_time, total_exec_time
    FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
    
    -- Самые частые
    SELECT query, calls FROM pg_stat_statements ORDER BY calls DESC LIMIT 10;

### pg_stat_database

    SELECT datname, numbackends, xact_commit, xact_rollback,
           blks_read, blks_hit,
           round(blks_hit::numeric/(blks_hit + blks_read) * 100, 2) as cache_hit_ratio
    FROM pg_stat_database;

## Часть 3: Ключевые метрики

### Производительность

| Метрика | Хорошо | Плохо |
|---------|--------|-------|
| connections | < 80% max | > 90% |
| cache_hit_ratio | > 99% | < 95% |
| query_time_avg | < 100ms | > 1s |

### Ресурсы

| Метрика | Хорошо | Плохо |
|---------|--------|-------|
| CPU | < 70% | > 90% |
| Memory | < 80% | > 95% |
| Disk space | < 70% | > 85% |

### Безопасность

| Метрика | Описание |
|---------|----------|
| failed_logins | Неудачные попытки |
| ssl_connections | Должно быть 100% |
| superuser_queries | Алерт на неожиданные |
| ddl_changes | Изменения структуры |

## Часть 4: Инструменты

### Prometheus + Grafana

    -- postgres_exporter для метрик
    -- Grafana для дашбордов
    -- AlertManager для алертов

### pgAdmin / pganalyze

| Инструмент | Тип |
|------------|-----|
| pgAdmin | Бесплатный GUI |
| pganalyze | Коммерческий, облачный |

## Часть 5: Алерты

| Условие | Приоритет |
|---------|-----------|
| connections > 80% | Critical |
| disk_space > 85% | Critical |
| cache_hit_ratio < 90% | Warning |
| failed_logins > 5 за 5 мин | High |
| query_time_avg > 1s | Warning |

## Вывод

Мониторинг PostgreSQL:
1. **Метрики** — что измеряем
2. **Алерты** — когда реагируем
3. **Инструменты** — Prometheus, Grafana, pgAdmin
4. **Runbook** — как реагируем

Без мониторинга — слепота. С мониторингом — контроль.

---
_Статья создана на основе анализа материалов по PostgreSQL_