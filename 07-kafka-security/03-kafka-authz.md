# Авторизация в Kafka: ACL и RBAC

## Часть 1: Зачем нужна авторизация?

Представь бизнес-центр. Ты прошёл охрану (аутентификация) и получил бейдж. Но бейдж открывает не все двери. Ты можешь войти в переговорную на 3-м этаже, но не в серверную на 5-м. Не в офис директора. Не в архив.

**Авторизация в Kafka** — это именно такая «система доступа к комнатам». После аутентификации (кто ты?) нужно решить: что тебе можно? Читать топик `orders`? Писать в `payments`? Удалять топики? Создавать ACL?

## Зачем нужна авторизация?

### Ситуация 1: Разделение окружений

Есть три окружения: `dev`, `staging`, `production`. Разработчики должны писать в `dev`, но только читать из `staging`. В `production` — только операционная команда.

### Ситуация 2: Разделение по командам

Команда `payments` работает с топиками `payments-*`. Команда `analytics` — с `analytics-*`. Без авторизации — analytics может случайно (или намеренно) писать в `payments-received`.

### Ситуация 3: Compliance

Стандарт требует: только определённые сервисы могут обрабатывать данные карт. Авторизация гарантирует: только `payment-processor` может писать в `card-transactions`.

## Часть 2: ACL (Access Control Lists)

### Что такое ACL

ACL — список правил: кто может что делать с каким ресурсом.

    # Формат ACL
    Principal    | Operation | ResourceType | ResourceName | Host
    -------------|-----------|--------------|--------------|------
    User:alice   | Read      | Topic        | orders       | *
    User:bob     | Write     | Topic        | payments     | 10.0.0.5
    User:admin   | All       | Cluster      | *            | *

### Типы ресурсов

| ResourceType | Описание | Примеры |
|--------------|----------|---------|
| Topic | Топик | `orders`, `payments` |
| Group | Consumer group | `order-processor` |
| Cluster | Кластер целиком | Управление брокерами |
| TransactionalId | Транзакция | `payment-tx-123` |
| DelegationToken | Токен делегирования | `token-abc` |

### Операции

| Operation | Описание | Применение |
|-----------|----------|------------|
| Read | Чтение | Consumer, описание |
| Write | Запись | Producer |
| Create | Создание | Новый топик, группа |
| Delete | Удаление | Топик, группа |
| Alter | Изменение | Конфигурация |
| Describe | Описание | Метаданные |
| ClusterAction | Административные | Переназначение партиций |
| All | Всё | Полный доступ |

### Управление ACL

    # Добавить ACL
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:alice \
        --operation Read --topic orders \
        --command-config admin.properties
    
    # Удалить ACL
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --remove --allow-principal User:alice \
        --operation Read --topic orders
    
    # Список ACL
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --list --topic orders
    
    # ACL для группы
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:alice \
        --operation Read --group order-processors

### Префиксные ACL

    # Дать доступ ко всем топикам с префиксом "payments-"
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:payments-app \
        --operation Read --topic payments- \
        --resource-pattern-type Prefixed

### Запрещающие ACL (Deny)

    # Запретить доступ (Deny имеет приоритет над Allow)
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --deny-principal User:hacker \
        --operation All --topic *

## Часть 3: Встроенный авторизатор

### AclAuthorizer

Встроенный авторизатор Kafka:

    # server.properties
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    
    # Разрешить доступ, если нет ACL (опасно!)
    # allow.everyone.if.no.acl.found=true  # ❌ НЕТ
    
    # Запретить по умолчанию (безопасно)
    allow.everyone.if.no.acl.found=false  # ✅ ДА
    
    # Суперпользователи
    super.users=User:admin;User:ops-team

### Принцип наименьших привилегий

| Пользователь | Нужные права | Не нужно |
|--------------|-------------|----------|
| Producer | Write, Create на топик | Read, Delete, Alter |
| Consumer | Read, Describe на топик | Write, Create, Delete |
| Admin | All на Cluster | — |
| Monitoring | Describe на всё | Write, Delete |

## Часть 4: RBAC (Role-Based Access Control)

### Что такое RBAC

Вместо управления правами для каждого пользователя — создаём роли и назначаем пользователей в роли.

| Роль | Права | Пользователи |
|------|-------|--------------|
| Producer | Write на payments-* | payments-app, billing-app |
| Consumer | Read на payments-* | analytics-app, reporting-app |
| Admin | All на Cluster | ops-team, security-team |
| ReadOnly | Read на * | monitoring, audit |

### RBAC в Confluent Platform

Confluent добавляет RBAC поверх Kafka:

    # Создать роль
    confluent iam rbac role create PaymentsProducer
    
    # Назначить права роли
    confluent iam rbac role-binding create \
        --role PaymentsProducer \
        --principal User:payments-app \
        --resource Topic:payments-* \
        --operation Write
    
    # Назначить пользователя в роль
    confluent iam rbac role-binding create \
        --role PaymentsProducer \
        --principal User:payments-app

### RBAC vs ACL

| | ACL | RBAC |
|---|-----|------|
| Управление | По пользователю | По роли |
| Масштабирование | Сложно при росте | Просто |
| Аудит | Сложнее | Проще |
| Kafka Open Source | ✅ Да | ❌ Нет (только Confluent) |

## Часть 5: Авторизация между брокерами

### Inter-broker authorization

Брокеры тоже аутентифицируются и авторизуются друг относительно друга:

    # server.properties
    # Брокер аутентифицируется как "broker"
    # Нужны права на Cluster для репликации
    
    # ACL для брокера
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:broker \
        --operation All --cluster

## Часть 6: Практические примеры

### Пример 1: Микросервисная архитектура

    # Сервис: order-service (producer)
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:order-service \
        --operation Write --topic orders \
        --operation Create --topic orders
    
    # Сервис: payment-service (consumer + producer)
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:payment-service \
        --operation Read --topic orders \
        --operation Write --topic payments \
        --operation Read --group payment-processors
    
    # Сервис: analytics-service (consumer)
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:analytics-service \
        --operation Read --topic orders,payments \
        --operation Read --group analytics-group

### Пример 2: Admin права

    # Только ops-team может управлять топиками
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:ops-team \
        --operation All --topic * \
        --operation All --cluster
    
    # Только security-team может управлять ACL
    kafka-acls.sh --bootstrap-server localhost:9093 \
        --add --allow-principal User:security-team \
        --operation Alter --cluster

## Вывод

Авторизация в Kafka:
1. **ACL** — базовый механизм, работает в Open Source
2. **RBAC** — удобнее для больших организаций (Confluent)
3. **Принцип:** запретить всё по умолчанию, разрешить минимум
4. **Суперпользователи** — только операционная команда
5. **Аудит** — все изменения ACL логировать

Без авторизации — аутентифицированный пользователь может всё. Это как бейдж, открывающий все двери.

---
_Статья создана на основе анализа материалов по безопасности Kafka_