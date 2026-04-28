# Мониторинг Kafka

## JMX Metrics

| Метрика | Описание |
|---------|----------|
| MessagesInPerSec | Сообщений в секунду |
| BytesInPerSec | Байт в секунду |
| BytesOutPerSec | Байт из |
| RequestsPerSec | Запросов в секунду |
| LogFlushRateAndTimeMs | Скорость flush |

## Логи

| Тип | Описание |
|-----|----------|
| server.log | Основной |
| controller.log | Controller |
| state-change.log | Изменения состояний |
| kafka-authorizer.log | Авторизация |
| kafka-request.log | Запросы |

## Security Monitoring

| Что мониторить | Описание |
|----------------|----------|
| Неудачные аутентификации | WARN |
| Неудачные авторизации | WARN |
| Изменения ACL | INFO |
| Создание топиков | INFO |
| Подключения | INFO |

## Алерты

| Условие | Действие |
|---------|----------|
| Failed auth > 10/min | Alert |
| Failed auth > 100/min | Block |
| Unknown client | Alert |
| ACL change | Alert |

## Инструменты

| Инструмент | Описание |
|------------|----------|
| Prometheus + Grafana | Метрики |
| ELK | Логи |
| Confluent Control Center | GUI |
| Burrow | Consumer lag |

---
_Статья создана на основе анализа материалов Habr по Kafka_
