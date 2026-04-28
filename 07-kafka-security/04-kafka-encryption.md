# Шифрование в Kafka

## TLS/SSL

### Настройка брокера

`
listeners=SASL_SSL://:9093
ssl.keystore.location=/path/to/server.keystore.jks
ssl.keystore.password=secret
ssl.key.password=secret
ssl.truststore.location=/path/to/server.truststore.jks
ssl.truststore.password=secret
ssl.client.auth=required
ssl.endpoint.identification.algorithm=HTTPS
`

### Версии TLS

| Версия | Рекомендация |
|--------|--------------|
| TLS 1.0 | Запретить |
| TLS 1.1 | Запретить |
| TLS 1.2 | Минимум |
| TLS 1.3 | Рекомендуется |

`
ssl.enabled.protocols=TLSv1.3
ssl.protocol=TLSv1.3
`

### SASL_SSL

| Комбинация | Описание |
|------------|----------|
| SASL_SSL | Аутентификация SASL + шифрование TLS |
| SSL | Только TLS (mTLS) |

## Шифрование данных

### At Rest

| Способ | Описание |
|--------|----------|
| OS-level | LUKS, dm-crypt |
| Cloud | EBS encryption, Azure Disk |
| Kafka | Нет встроенного |

### In Transit

| Способ | Описание |
|--------|----------|
| TLS | Всегда включать |
| SASL_SSL | Аутентификация + шифрование |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| TLS 1.3 | Максимум |
| mTLS | Для высокой безопасности |
| Сертификаты | Регулярная ротация |
| HSTS | Strict-Transport-Security |
| Monitoring | Проверка шифрования |

---
_Статья создана на основе анализа материалов Habr по Kafka_
