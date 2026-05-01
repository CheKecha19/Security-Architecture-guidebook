# Лучшие практики безопасности Kafka

## Часть 1: Архитектура безопасности Kafka

Представь банк. Не просто сейф, а система:
- Охрана у входа
- Камеры в холле
- Сейфовые ячейки
- Ограниченный доступ в хранилище
- Бронированные машины для перевозки
- Аудит каждой операции

**Kafka** — это цифровой банк для данных. Данные — валюта. Безопасность — обязательна.

## Часть 2: Чек-лист безопасности

### Сеть

| Проверка | Статус | Как |
|----------|--------|-----|
| Kafka не доступна из интернета | ⬜ | Firewall, Security Group |
| Только TLS для внешних подключений | ⬜ | security.protocol=SSL |
| Внутри сети — TLS или SASL_SSL | ⬜ | В зависимости от зоны |
| Ограничение портов | ⬜ | Только 9093 (SSL) |
| Network segmentation | ⬜ | Отдельная VLAN для Kafka |

### Аутентификация

| Проверка | Статус | Как |
|----------|--------|-----|
| SASL/SSL или mTLS | ⬜ | Не PLAINTEXT |
| SCRAM-SHA-256 или SCRAM-SHA-512 | ⬜ | Не PLAIN, не MD5 |
| Отдельные пользователи для каждого приложения | ⬜ | Не shared аккаунт |
| Регулярная ротация паролей SCRAM | ⬜ | kafka-configs --alter |
| mTLS с проверкой CN | ⬜ | ssl.client.auth=required |

### Авторизация

| Проверка | Статус | Как |
|----------|--------|-----|
| ACL включён | ⬜ | authorizer.class.name=kafka.security.authorizer.AclAuthorizer |
| Топик default — deny | ⬜ | allow.everyone.if.no.acl.found=false |
| Минимальные права | ⬜ | Read для consumer, Write для producer |
| Префиксные ACL для групп | ⬜ | ResourcePatternType.Prefixed |
| Регулярный аудит ACL | ⬜ | kafka-acls --list |

### Шифрование

| Проверка | Статус | Как |
|----------|--------|-----|
| TLS 1.2+ | ⬜ | ssl.protocol=TLSv1.2 |
| Шифрование in-transit | ⬜ | security.protocol=SASL_SSL |
| Шифрование at-rest (диск) | ⬜ | OS-level (LUKS) или cloud encryption |
| Шифрование backups | ⬜ | LUKS или GPG |

### Мониторинг и аудит

| Проверка | Статус | Как |
|----------|--------|-----|
| Логирование авторизации | ⬜ | log4j.logger.kafka.authorizer.logger |
| Логирование неудачных входов | ⬜ | log4j.logger.kafka.network.RequestChannel |
| Централизация логов | ⬜ | Filebeat → ELK/Splunk |
| Алерты на failed auth | ⬜ | Prometheus AlertManager |
| Алерты на denied operations | ⬜ | SIEM rules |
| Хранение логов > 1 год | ⬜ | logrotate + archive |

## Часть 3: Конфигурация по умолчанию

### server.properties — безопасный шаблон

    # === Сеть ===
    listeners=SASL_SSL://:9093
    advertised.listeners=SASL_SSL://kafka.example.com:9093
    security.inter.broker.protocol=SASL_SSL
    
    # === SSL ===
    ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
    ssl.keystore.password=keystore-password
    ssl.key.password=key-password
    ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
    ssl.truststore.password=truststore-password
    ssl.client.auth=required
    ssl.enabled.protocols=TLSv1.2,TLSv1.3
    
    # === SASL ===
    sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
    sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
    
    # === Authorization ===
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    allow.everyone.if.no.acl.found=false
    super.users=User:admin
    
    # === Audit ===
    log4j.logger.kafka.authorizer.logger=INFO
    log4j.logger.kafka.request.logger=WARN
    
    # === Quotas ===
    quota.producer.default=10485760
    quota.consumer.default=10485760
    
    # === Replication ===
    default.replication.factor=3
    min.insync.replicas=2

## Часть 4: Процедуры

### Регулярные задачи

| Период | Задача | Ответственный |
|--------|--------|---------------|
| Ежедневно | Проверка failed auth в логах | Security |
| Еженедельно | Аудит ACL | Security |
| Еженедельно | Проверка уязвимостей Kafka | DBA |
| Ежемесячно | Ротация паролей SCRAM | DBA |
| Ежемесячно | Тестирование восстановления | DBA |
| Ежеквартально | Пентест | Security |
| Ежеквартально | Обзор архитектуры безопасности | Security + Architect |

### Реагирование на инциденты

| Инцидент | Действие |
|----------|----------|
| Массовые failed auth | Блокировать IP, сменить пароль, расследовать |
| Необычный доступ к топику | Проверить ACL, отозвать если нужно |
| Удаление топика | Восстановление из backup, расследование |
| Изменение ACL | Откатить, проверить кто и зачем |
| Подозрительный producer | Проверить сертификат/пользователя |

## Часть 5: Интеграция с инфраструктурой

### Terraform / Ansible

    # Инфраструктура как код
    # Kafka cluster с защитой по умолчанию
    # Не допускать конфигурацию без SSL

### CI/CD для Kafka

    [Code Review] > [Test] > [Security Scan] > [Deploy]
                      |
                      v
              [Проверка ACL changes]
              [Валидация конфигурации]
              [Тесты аутентификации]

### GitOps

    # Конфигурация Kafka в Git
    # PR для изменений
    # Автоматическая валидация
    # Автоматический деплой

## Вывод

Безопасность Kafka — система:
1. **Сеть** — изоляция, firewall
2. **Аутентификация** — SCRAM или mTLS
3. **Авторизация** — ACL, минимальные права
4. **Шифрование** — TLS, at-rest
5. **Аудит** — логи, SIEM
6. **Мониторинг** — алерты, аномалии
7. **Процессы** — регулярные проверки
8. **Автоматизация** — IaC, GitOps

Принцип: **не доверяй сети. Шифруй всё. Аудируй всё.**

---
_Статья создана на основе анализа материалов по безопасности Kafka_