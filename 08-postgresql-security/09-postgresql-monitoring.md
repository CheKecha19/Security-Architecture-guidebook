# Мониторинг PostgreSQL

## pg_stat_statements

`sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
`

## Метрики

| Метрика | Описание |
|---------|----------|
| connections | Активные подключения |
| transactions | Транзакции в секунду |
| cache_hit_ratio | Эффективность кэша |
| lock_waits | Ожидания блокировок |
| replication_lag | Лаг репликации |
| deadlocks | Взаимоблокировки |

## Алерты

| Условие | Действие |
|---------|----------|
| connections > 80% | Alert |
| replication_lag > 1min | Alert |
| deadlocks > 0 | Alert |
| cache_hit_ratio < 90% | Warning |

## Инструменты

| Инструмент | Описание |
|------------|----------|
| pgAdmin | GUI |
| pganalyze | Cloud |
| Grafana + Prometheus | Метрики |
| Nagios | Алерты |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
