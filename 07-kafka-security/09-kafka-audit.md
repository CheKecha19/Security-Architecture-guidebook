# Аудит в Kafka

## Логирование

| Событие | Уровень |
|---------|---------|
| Аутентификация | INFO |
| Авторизация | INFO |
| ACL изменение | INFO |
| Топик создан | INFO |
| Группа присоединена | INFO |

## Authorizer Logging

`
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Log
[2024-01-01 00:00:00] INFO Principal = User:alice is Allowed Operation = Read from host = 10.0.0.1 on resource = Topic:my-topic
`

## Audit Events

| Событие | Описание |
|---------|----------|
| Authentication successful | Пользователь вошёл |
| Authentication failed | Неудача |
| Authorization denied | Доступ запрещён |
| Topic created | Новый топик |
| ACL changed | Изменение прав |

## Compliance

| Стандарт | Требование |
|----------|------------|
| PCI DSS | Аудит доступа |
| HIPAA | Кто видел PHI |
| GDPR | Кто обрабатывал ПДн |
| SOC 2 | Логи, мониторинг |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Централизованные логи | ELK, Splunk |
| Неизменяемость | Нельзя удалить |
| Анализ | SIEM |
| Ротация | Хранение по compliance |
| Алерты | На критичные события |

---
_Статья создана на основе анализа материалов Habr по Kafka_
