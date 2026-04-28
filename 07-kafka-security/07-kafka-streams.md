# Безопасность Kafka Streams

## Аутентификация

| Параметр | Описание |
|----------|----------|
| application.id | ID приложения |
| sasl.mechanism | SCRAM-SHA-512 |
| security.protocol | SASL_SSL |

## ACL для Streams

| Ресурс | Права |
|--------|-------|
| Input topics | Read |
| Output topics | Write |
| Internal topics | All |
| Consumer group | Read |

## KSQL

| Параметр | Описание |
|----------|----------|
| ksql.authentication.plugin | Плагин аутентификации |
| ksql.access.validator | Валидатор доступа |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Отдельный пользователь | Для Streams |
| ACL на internal topics | Автоматически |
| Monitoring | Лаг, ошибки |
| Isolation | Application ID |

---
_Статья создана на основе анализа материалов Habr по Kafka_
