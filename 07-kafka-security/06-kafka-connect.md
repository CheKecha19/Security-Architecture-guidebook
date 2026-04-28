# Безопасность Kafka Connect

## Аутентификация

| Параметр | Описание |
|----------|----------|
| sasl.mechanism | SCRAM-SHA-512 |
| security.protocol | SASL_SSL |
| ssl.endpoint.identification.algorithm | HTTPS |

## Credentials

| Способ | Описание |
|--------|----------|
| Config file | Не хранить пароли |
| External secrets | Vault, AWS Secrets Manager |
| ConfigProvider | Динамическая загрузка |

`
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider
`

## Connector ACL

| Ресурс | Права |
|--------|-------|
| Source topics | Read |
| Sink topics | Write |
| Configs | Describe, Alter |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Отдельный пользователь | Для Connect |
| Минимум прав | Только необходимые топики |
| Secrets | Вне конфигурации |
| Monitoring | Логи, метрики |

---
_Статья создана на основе анализа материалов Habr по Kafka_
