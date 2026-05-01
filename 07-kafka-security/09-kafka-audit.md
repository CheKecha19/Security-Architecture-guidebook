# Аудит безопасности Kafka

## Часть 1: Зачем аудит?

Представь магазин без камер. Кража — и непонятно, кто, когда, как. С камерами — всё видно.

**Аудит Kafka** — это «камеры» для брокера сообщений. Кто подключался? Что читал? Что писал? Когда?

## Зачем нужен аудит?

### Ситуация 1: Расследование утечки

Данные из топика `payments` утекли. Без аудита — непонятно, кто имел доступ. С аудитом — видно: `User:analytics-app` читал топик каждый день. Время утечки совпадает.

### Ситуация 2: Compliance (SOX)

Аудиторы требуют: покажите, кто имел доступ к финансовым данным. Аудит-логи показывают: `User:trading-app` писал в `trades`, `User:reporting-app` читал из `trades`.

### Ситуация 3: Обнаружение инсайдера

Сотрудник увольняется. В его последний день анализ логов показывает: он читал топики, к которым раньше не обращался. Запрос данных «на память».

## Часть 2: Что аудировать

### Аутентификация

| Событие | Что записывать | Уровень |
|---------|---------------|---------|
| Успешный вход | User, time, IP, mechanism | INFO |
| Неудачный вход | User, time, IP, reason | WARN |
| SSL handshake | Certificate CN, time | DEBUG |
| SASL handshake | Mechanism, result | DEBUG |

### Авторизация

| Событие | Что записывать | Уровень |
|---------|---------------|---------|
| Разрешён доступ | User, resource, operation, time | INFO |
| Отказано в доступе | User, resource, operation, time | WARN |
| ACL изменён | Who, what, when | INFO |

### Операции

| Событие | Что записывать | Уровень |
|---------|---------------|---------|
| Создание топика | Who, topic name, config | INFO |
| Удаление топика | Who, topic name | WARN |
| Изменение config | Who, what changed | INFO |
| Producer записал | User, topic, partition, offset (метаданные) | DEBUG |
| Consumer прочитал | User, topic, partition, offset | DEBUG |
| Group join | Member ID, group ID | DEBUG |

## Часть 3: Настройка аудита

### Log4j для аудита

    # server.properties — логирование
    log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender
    log4j.additivity.kafka.authorizer.logger=false
    
    log4j.logger.kafka.request.logger=WARN, requestAppender
    log4j.additivity.kafka.request.logger=false
    
    # Конфигурация appender
    log4j.appender.authorizerAppender=org.apache.log4j.RollingFileAppender
    log4j.appender.authorizerAppender.File=/var/log/kafka/authorizer.log
    log4j.appender.authorizerAppender.MaxFileSize=100MB
    log4j.appender.authorizerAppender.MaxBackupIndex=10
    log4j.appender.authorizerAppender.layout=org.apache.log4j.PatternLayout
    log4j.appender.authorizerAppender.layout.ConversionPattern=[%d] %p %m (%c)%n

### Структурированный JSON-лог

    # Использовать JSON layout
    log4j.appender.KAFKA_JSON=org.apache.log4j.RollingFileAppender
    log4j.appender.KAFKA_JSON.File=/var/log/kafka/kafka-json.log
    log4j.appender.KAFKA_JSON.layout=net.logstash.log4j.JSONEventLayoutV1
    
    # Результат:
    # {
    #   "timestamp": "2024-01-15T10:30:00Z",
    #   "level": "INFO",
    #   "logger": "kafka.authorizer.logger",
    #   "message": "Principal = User:app is Allowed Operation = Read from host = 10.0.0.5"
    # }

### Authorizer Audit (kafka-authorizer.log)

Встроенный authorizer (AclsAuthorizer) логирует:

    # Разрешено
    Principal = User:payments-app is Allowed Operation = Write from host = 10.0.0.5 on resource = Topic:LITERAL:payments
    
    # Отказано
    Principal = User:hacker is Denied Operation = Read from host = 192.168.1.100 on resource = Topic:LITERAL:payments

## Часть 4: Сбор и анализ логов

### ELK Stack

    # Filebeat для Kafka
    filebeat.inputs:
      - type: log
        enabled: true
        paths:
          - /var/log/kafka/authorizer.log
          - /var/log/kafka/server.log
        fields:
          service: kafka
          cluster: production
        fields_under_root: true
    
    output.elasticsearch:
      hosts: ["http://elasticsearch:9200"]
      index: "kafka-audit-%{+yyyy.MM.dd}"
    
    # Kibana dashboard:
    # - Количество отказов по времени
    # - Топ пользователей с отказами
    # - Географическая карта IP
    # - Тренды по операциям

### Splunk

    # Splunk Universal Forwarder
    [monitor:///var/log/kafka]
    index = kafka_security
    sourcetype = kafka_logs
    
    # Splunk alerts:
    # - Failed auth > 10 за 5 минут
    # - Denied operations из нового IP
    # - ACL changes вне окна обслуживания

### SIEM Integration

| SIEM | Способ | Алерты |
|------|--------|--------|
| Splunk | Forwarder | Готовые dashboards |
| QRadar | Syslog | Custom rules |
| ArcSight | SmartConnector | Correlation rules |
| Azure Sentinel | Log Analytics agent | KQL queries |
| Chronicle (Google) | Forwarder | YARA-L rules |

## Часть 5: Анализ и расследование

### Типичные запросы

    # Поиск отказов в доступе за последний час
    grep "is Denied" /var/log/kafka/authorizer.log | grep "$(date -d '1 hour ago' '+%Y-%m-%d %H')"
    
    # Статистика по пользователям
    grep "is Denied" /var/log/kafka/authorizer.log | awk -F'Principal = ' '{print $2}' | awk '{print $1}' | sort | uniq -c | sort -rn
    
    # Поиск по IP
    grep "192.168.1.100" /var/log/kafka/authorizer.log | grep "is Denied"
    
    # Изменения ACL
    grep "Added ACL" /var/log/kafka/server.log
    grep "Removed ACL" /var/log/kafka/server.log

### Корреляция событий

    # Сценарий: brute-force + успешный вход + доступ к данным
    # 1. Много failed auth
    # 2. Один successful auth
    # 3. Немедленный доступ к чувствительному топику
    
    # Elasticsearch query:
    {
      "query": {
        "bool": {
          "must": [
            {"match": {"message": "Failed authentication"}},
            {"range": {"timestamp": {"gte": "now-1h"}}}
          ]
        }
      }
    }

## Часть 6: Хранение и ротация

### Политика хранения

| Тип лога | Минимум | Рекомендация |
|----------|---------|--------------|
| Аутентификация (failed) | 1 год | 2 года |
| Авторизация (denied) | 1 год | 2 года |
| Авторизация (allowed) | 3 месяца | 1 год |
| Изменения ACL | 2 года | 3 года |
| Producer/consumer debug | 1 месяц | 3 месяца |

### Ротация

    # logrotate
    /var/log/kafka/*.log {
        daily
        rotate 365
        compress
        delaycompress
        missingok
        notifempty
        create 640 kafka kafka
    }

## Вывод

Аудит Kafka:
1. **Собирать** — все события аутентификации и авторизации
2. **Структурировать** — JSON для машинной обработки
3. **Централизовать** — ELK, Splunk, SIEM
4. **Анализировать** — корреляция, anomaly detection
5. **Хранить** — по compliance-требованиям
6. **Реагировать** — runbook для инцидентов

Без аудита — нет visibility. Без visibility — нет безопасности.

---
_Статья создана на основе анализа материалов по безопасности Kafka_