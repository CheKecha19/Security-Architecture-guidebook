# Безопасность Schema Registry

## Аутентификация

| Способ | Описание |
|--------|----------|
| Basic Auth | Логин/пароль |
| SASL | SCRAM-SHA-512 |
| mTLS | Сертификаты |
| OAuth | Bearer token |

## Авторизация

| Ресурс | Операции |
|--------|----------|
| Subject | Read, Write, Delete |
| Schema | Read |
| Compatibility | Alter |

## Schema ACL

`ash
curl -X POST http://registry:8081/subjects/my-topic/versions \
  -u alice:secret \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "..."}'
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| HTTPS | Всегда |
| Аутентификация | Обязательно |
| ACL | На каждый subject |
| Регистрация | Только authorized |
| Мониторинг | Кто что регистрирует |

---
_Статья создана на основе анализа материалов Habr по Kafka_
