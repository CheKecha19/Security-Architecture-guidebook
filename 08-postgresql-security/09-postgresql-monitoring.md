# Мониторинг PostgreSQL

## Часть 1: Зачем нужен мониторинг?

Представь, что ты управляешь атомной электростанцией. Нет приборов — не знаешь, что происходит. Есть приборы — видишь температуру, давление, радиацию. При превышении — сигнализация.

**Мониторинг PostgreSQL** — это «приборная панель» для базы данных. Показывает: здорова ли база, не перегружается ли, нет ли проблем.

## Зачем нужен мониторинг?

### Ситуация 1: Медленные запросы

Пользователи жалуются: «Сайт тормозит!». Без мониторинга — непонятно, в чём проблема. С мониторингом — видно: один запрос занимает 30 секунд, блокирует таблицу.

### Ситуация 2: Заканчивается место

Диск заполнен на 95%. Без мониторинга — база упадёт. С мониторингом — алерт при 80%, время на реакцию.

### Ситуация 3: Репликация отстала

Standby отстаёт на 2 часа. Если primary упадёт — потеряем 2 часа данных. Мониторинг показывает лаг.

## Часть 2: Встроенная статистика

### pg_stat_activity

    -- Текущие подключения
    SELECT 
        pid,
        usename,
        application_name,
        client_addr,
        state,
        query_start,
        state_change,
        query
    FROM pg_stat_activity
    WHERE state != 'idle';
    
    -- Долгие запросы (> 5 секунд)
    SELECT 
        pid,
        usename,
        query_start,
        now() - query_start as duration,
        query
    FROM pg_stat_activity
    WHERE state = 'active'
    AND now() - query_start > interval '5 seconds';
    
    -- Блокировки
    SELECT 
        blocked_locks.pid AS blocked_pid,
        blocked_activity.usename AS blocked_user,
        blocking_locks.pid AS blocking_pid,
        blocking_activity.usename AS blocking_user,
        blocked_activity.query AS blocked_query,
        blocking_activity.query AS blocking_query
    FROM pg_catalog.pg_locks blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
    WHERE NOT blocked_locks.granted;

### pg_stat_statements

    -- Установка
    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    
    -- Самые медленные запросы
    SELECT 
        query,
        calls,
        mean_exec_time,
        total_exec_time,
        rows
    FROM pg_stat_statements
    ORDER BY total_exec_time DESC
    LIMIT 10;
    
    -- Самые частые запросы
    SELECT 
        query,
        calls,
        mean_exec_time
    FROM pg_stat_statements
    ORDER BY calls DESC
    LIMIT 10;
    
    -- Сброс статистики
    SELECT pg_stat_statements_reset();

### pg_stat_database

    -- Статистика по базам
    SELECT 
        datname,
        numbackends,
        xact_commit,
        xact_rollback,
        blks_read,
        blks_hit,
        round(blks_hit::numeric/(blks_hit + blks_read) * 100, 2) as cache_hit_ratio
    FROM pg_stat_database
    WHERE datname NOT IN ('template0', 'template1');

### pg_stat_user_tables

    -- Статистика по таблицам
    SELECT 
        schemaname,
        relname,
        seq_scan,
        idx_scan,
        n_tup_ins,
        n_tup_upd,
        n_tup_del,
        n_live_tup,
        n_dead_tup,
        last_vacuum,
        last_autovacuum,
        last_analyze
    FROM pg_stat_user_tables
    ORDER BY n_live_tup DESC
    LIMIT 10;

## Часть 3: Ключевые метрики

### Производительность

| Метрика | Хорошо | Плохо | Действие |
|---------|--------|-------|----------|
| connections | < 80% от max | > 90% | Увеличить max_connections или пул |
| cache_hit_ratio | > 99% | < 95% | Увеличить shared_buffers |
| index_hit_ratio | > 95% | < 90% | Проверить индексы |
| transactions/sec | Зависит от нагрузки | Резкое падение | Искать блокировки |
| query_time_avg | < 100ms | > 1s | Оптимизировать запросы |

### Репликация

| Метрика | Хорошо | Плохо | Действие |
|---------|--------|-------|----------|
| replication_lag | < 1s | > 1min | Проверить сеть, нагрузку |
| wal_lag | < 100MB | > 1GB | Проверить standby |
| slots_active | Да | Нет | Проверить слоты |

### Ресурсы

| Метрика | Хорошо | Плохо | Действие |
|---------|--------|-------|----------|
| CPU | < 70% | > 90% | Масштабировать |
| Memory | < 80% | > 95% | Увеличить RAM |
| Disk I/O | < 80% | > 95% | SSD, оптимизация |
| Disk space | < 70% | > 85% | Очистка, расширение |

### Безопасность

| Метрика | Описание | Действие |
|---------|----------|----------|
| failed_logins | Неудачные попытки входа | Алерт при > 5 за 5 минут |
| ssl_connections | Подключения по SSL | Должно быть 100% |
| superuser_queries | Запросы от superuser | Алерт на неожиданные |
| ddl_changes | Изменения структуры | Алерт на неожиданные |

## Часть 4: Инструменты мониторинга

### pgAdmin

| Плюс | Минус |
|------|-------|
| Бесплатно | Только GUI |
| Встроен в PostgreSQL | Нет алертов |
| Простой | Нет истории |

### Prometheus + Grafana

    -- Установка postgres_exporter
    -- Сбор метрик из pg_stat_*
    -- Визуализация в Grafana
    -- Алерты через AlertManager

    # Пример запроса Prometheus
    pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)

### pganalyze

| Плюс | Минус |
|------|-------|
| Автоматический анализ | Платно |
| Рекомендации | Облачный |
| История | Зависимость от вендора |

### Nagios / Zabbix

    # Проверка подключений
    check_pgsql -H localhost -U nagios -d mydb
    
    # Проверка репликации
    check_pgsql_replication -H standby -U nagios

## Часть 5: Алерты и реагирование

### Примеры алертов

| Условие | Приоритет | Действие |
|---------|-----------|----------|
| connections > 80% | Critical | Добавить пул, увеличить max_connections |
| replication_lag > 1min | Critical | Проверить standby, сеть |
| disk_space > 85% | Critical | Очистить WAL, архивы |
| deadlocks > 0 | High | Перезапустить приложение |
| cache_hit_ratio < 90% | Warning | Увеличить shared_buffers |
| failed_logins > 5 за 5 мин | High | Проверить brute-force |
| query_time_avg > 1s | Warning | Оптимизировать запросы |

### Runbook для инцидентов

    -- 1. База не отвечает
    -- Проверить процесс
    ps aux | grep postgres
    
    -- Проверить диск
    df -h
    
    -- Проверить логи
    tail -f /var/log/postgresql/postgresql-*.log
    
    -- 2. Медленные запросы
    -- Найти и завершить
    SELECT pg_terminate_backend(pid) 
    FROM pg_stat_activity 
    WHERE now() - query_start > interval '30 seconds';
    
    -- 3. Место на диске заканчивается
    -- Очистить WAL
    pg_archivecleanup /backup/wal $(pg_controldata | grep "Latest checkpoint's REDO WAL file" | awk '{print $NF}')
    
    -- Очистить логи
    find /var/log/postgresql -name "*.log" -mtime +7 -delete

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Мониторить всё | Метрики, алерты, дашборды |
| Алерты | Приоритизировать |
| Runbook | Документировать реакцию |
| История | Хранить метрики > 1 год |
| Тестировать | Алерты должны срабатывать |
| Автоматизация | Авто-ремедиация |
| Capacity planning | Прогнозировать рост |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| pg_stat_statements установлен | ⬜ |
| Метрики собираются | ⬜ |
| Дашборд настроен | ⬜ |
| Алерты настроены | ⬜ |
| Runbook документирован | ⬜ |
| История хранится | ⬜ |
| Тестирование алертов | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Мониторинг доступа к данным карт | pg_stat_activity + SIEM |
| SOX | Мониторинг изменений | pg_stat_statements + алерты |
| HIPAA | Мониторинг PHI | Аудит + метрики |
| ISO 27001 | Мониторинг событий безопасности | Полный мониторинг |

## Вывод

Мониторинг — глаза и уши администратора:
1. **Метрики** — что измеряем
2. **Алерты** — когда реагируем
3. **Runbook** — как реагируем
4. **История** — для анализа

Без мониторинга — слепота. С мониторингом — контроль.

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_