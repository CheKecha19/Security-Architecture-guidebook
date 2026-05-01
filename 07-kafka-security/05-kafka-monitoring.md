# Мониторинг и аудит безопасности Kafka

## Часть 1: Зачем мониторить Kafka?

Представь ночной клуб. Входная дверь, бар, танцпол, VIP-зона. Без охраны и камер — хаос. С охраной — порядок. Камеры записывают: кто вошёл, куда пошёл, что делал. Охрана реагирует: «Стоп, этот человек не должен быть в VIP».

**Мониторинг Kafka** — это «камеры и охрана» для твоего брокера сообщений. Кто подключается? Какие топики читает? Есть ли неудачные попытки входа? Всё это должно быть видно.

## Зачем нужен мониторинг безопасности?

### Ситуация 1: Неудачные попытки аутентификации

В логах: `Failed authentication with SSL for user: admin`. Повторяется 1000 раз за минуту. Это не «забыли пароль». Это brute-force атака.

### Ситуация 2: Несанкционированный доступ к топику

Пользователь `app-consumer` пытается читать топик `payments`. У него нет прав. В логах: `Principal = User:app-consumer is Denied Operation = Describe from host = 10.0.0.5`. Это попытка получить доступ к чувствительным данным.

### Ситуация 3: Изменение ACL

Кто-то добавил правило: `User:hacker` имеет `ALL` на топик `*`. Это полный доступ ко всему. Если не видеть такие изменения — система скомпрометирована.

## Часть 2: Что мониторить

### Метрики JMX для безопасности

| Метрика | Что показывает | Порог тревоги |
|---------|---------------|---------------|
| kafka.network.RequestMetrics.RequestQueueTimeMs | Время в очереди | > 500ms |
| kafka.network.RequestMetrics.TotalTimeMs | Общее время запроса | > 1000ms |
| kafka.server.BrokerTopicMetrics.MessagesInPerSec | Сообщений в секунду | Резкое изменение |
| kafka.server.BrokerTopicMetrics.BytesInPerSec | Байт в секунду | Резкое изменение |
| kafka.server.ReplicaManager.LeaderCount | Количество лидеров | Изменение |
| kafka.controller.ControllerStats.LeaderElectionRateAndTimeMs | Выборы лидера | > 100ms |

### Логи безопасности Kafka

| Лог-файл | Что содержит | Где найти |
|----------|-------------|-----------|
| server.log | Основные события, аутентификация, авторизация | logs/server.log |
| controller.log | События controller | logs/controller.log |
| state-change.log | Изменения состояний партиций | logs/state-change.log |
| kafka-authorizer.log | Решения авторизации | logs/kafka-authorizer.log |
| kafka-request.log | Детали запросов | logs/kafka-request.log |

### Что искать в логах

| Паттерн | Значение | Действие |
|---------|----------|----------|
| `Failed authentication` | Неудачный вход | Алерт при > 5 за минуту |
| `is Denied Operation` | Отказано в доступе | Алерт при > 10 за минуту |
| `Added ACL` | Новое правило доступа | Алерт на все изменения |
| `Removed ACL` | Удалено правило | Алерт на все изменения |
| `Created topic` | Новый топик | Информация |
| `Deleted topic` | Удалён топик | Алерт |

## Часть 3: Инструменты мониторинга

### Prometheus + Grafana

    # JMX Exporter для Kafka
    java -javaagent:jmx_prometheus_javaagent.jar=7071:kafka-jmx.yml -jar kafka-server.jar
    
    # kafka-jmx.yml — конфигурация метрик
    rules:
      - pattern: kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec><>Count
        name: kafka_messages_in_total
        type: COUNTER
      - pattern: kafka.network<type=RequestMetrics, name=RequestsPerSec, request=(Produce|Fetch)><>Count
        name: kafka_requests_total
        labels:
          request: "$1"

### ELK Stack (Elasticsearch + Logstash + Kibana)

    # Filebeat для сбора логов Kafka
    filebeat.inputs:
      - type: log
        enabled: true
        paths:
          - /var/log/kafka/*.log
        fields:
          service: kafka
        multiline.pattern: '^\['
        multiline.negate: true
        multiline.match: after
    
    output.elasticsearch:
      hosts: ["http://localhost:9200"]

### Splunk / Datadog / New Relic

Коммерческие решения с готовыми дашбордами для Kafka:
- Преднастроенные алерты
- Корреляция событий
- Интеграция с SIEM

## Часть 4: Алерты безопасности

### Примеры алертов

| Условие | Приоритет | Действие |
|---------|-----------|----------|
| Failed auth > 10 за 5 минут | Critical | Блокировать IP, оповестить SOC |
| Denied operation > 50 за 10 минут | High | Проверить пользователя, возможно скомпрометирован |
| ACL изменён вне окна обслуживания | Critical | Проверить, кто и зачем |
| Новый пользователь SASL/SCRAM | High | Проверить легитимность |
| Необычный объём данных | Warning | Проверить, не утечка ли |
| Producer без аутентификации | Critical | Немедленная блокировка |

### Пример алерта в Prometheus AlertManager

    groups:
      - name: kafka_security
        rules:
          - alert: KafkaFailedAuthentication
            expr: increase(kafka_server_replica_manager_failedisrupdatespersec[5m]) > 10
            for: 1m
            labels:
              severity: critical
            annotations:
              summary: "Multiple failed authentications in Kafka"
              description: "{{ $value }} failed authentications in last 5 minutes"

## Часть 5: Аудит

### Включение аудита

    # server.properties
    # Логирование всех запросов
    log4j.logger.kafka.request.logger=WARN, requestAppender
    log4j.additivity.kafka.request.logger=false
    
    # Логирование авторизации
    log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender
    log4j.additivity.kafka.authorizer.logger=false

### Аудит с KAFKA-12180 (Kafka 2.5+)

Встроенный аудит для AdminClient операций:

    // Включить аудит в конфигурации
    log4j.logger.kafka.audit.logger=INFO, auditAppender

### Что аудировать

| Событие | Уровень | Хранение |
|---------|---------|----------|
| Аутентификация (успех/неудача) | INFO | 1 год |
| Авторизация (разрешено/отказано) | INFO | 1 год |
| Изменения ACL | INFO | 2 года |
| Создание/удаление топиков | INFO | 1 год |
| Изменения конфигурации | INFO | 2 года |
| Подключение/отключение клиентов | DEBUG | 3 месяца |

## Часть 6: Runbook для инцидентов

### Сценарий: Массовые неудачные аутентификации

    Шаг 1: Проверить логи
    grep "Failed authentication" /var/log/kafka/server.log | tail -100
    
    Шаг 2: Определить источник
    grep "Failed authentication" /var/log/kafka/server.log | awk '{print $NF}' | sort | uniq -c | sort -rn
    
    Шаг 3: Если IP вне whitelist — заблокировать на firewall
    iptables -A INPUT -s <IP> -j DROP
    
    Шаг 4: Проверить, не скомпрометирован ли пользователь
    # Проверить последние успешные входы
    grep "Successfully authenticated" /var/log/kafka/server.log | grep "User:admin"
    
    Шаг 5: При необходимости — сменить пароль SCRAM
    kafka-configs.sh --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=new-password]' --entity-type users --entity-name admin

### Сценарий: Подозрительный доступ к топику

    Шаг 1: Найти все попытки доступа к топику
    grep "Topic=payments" /var/log/kafka/kafka-authorizer.log
    
    Шаг 2: Проверить ACL для топика
    kafka-acls.sh --bootstrap-server localhost:9092 --list --topic payments
    
    Шаг 3: Если есть лишние права — удалить
    kafka-acls.sh --bootstrap-server localhost:9092 --remove --allow-principal User:suspicious-user --operation All --topic payments
    
    Шаг 4: Проверить другие топики
    kafka-acls.sh --bootstrap-server localhost:9092 --list | grep "User:suspicious-user"

## Вывод

Мониторинг и аудит Kafka:
1. **Собирать логи** — все события аутентификации и авторизации
2. **Анализировать метрики** — JMX для производительности и безопасности
3. **Настроить алерты** — на неудачные входы, отказы в доступе, изменения ACL
4. **Хранить историю** — минимум 1 год для безопасности
5. **Иметь runbook** — что делать при инциденте

Без мониторинга — ты слеп. С мониторингом — ты видишь угрозы.

---
_Статья создана на основе анализа материалов по безопасности Kafka_