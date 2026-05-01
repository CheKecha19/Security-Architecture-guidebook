# Авторизация в Kafka: управление доступом через ACL и RBAC

## Что такое авторизация в Kafka?

Представь офисное здание. Ты прошёл охрану (аутентификация) и получил бейдж. Но бейдж открывает не все двери. Ты можешь войти на свой этаж, в переговорную, в кафетерий. Но не в серверную. Не в архив. Не в кабинет директора. Если бейдж открывает всё — это не бейдж, это мастер-ключ, и он опасен.

**Авторизация в Kafka** — это именно такая «система доступа к комнатам» после прохождения охраны. После аутентификации (кто ты?) нужно решить: что тебе можно? Можешь ли ты читать топик `payments`? Можешь ли ты писать в `orders`? Можешь ли ты удалять топики? Создавать новые? Управлять ACL?

Без авторизации — аутентифицированный пользователь может всё. Это как бейдж, открывающий все двери: серверную, архив, сейф. Однажды такой бейдж попадёт не в те руки — и последствия будут катастрофическими.

## Эволюция и мотивация

Ранние версии Kafka (до 0.9) вообще не имели механизма авторизации. Если ты мог подключиться — ты мог делать всё. Это было время «доверенных сетей», где весь perimeter security сводился к firewall.

Версия 0.9 (2015) ввела **SimpleAclAuthorizer** — первый механизм управления доступом на основе ACL (Access Control Lists). Но он был ограничен: только whitelist, нет deny правил, сложное управление в масштабе.

Версия 0.11 улучшила ACL: добавила **ResourceType** (Topic, Group, Cluster, TransactionalId, DelegationToken), **Operation** (Read, Write, Create, Delete, Alter, Describe, ClusterAction), **Principal** (User или Group), **Host** (IP или wildcard).

Confluent Platform добавил **RBAC** (Role-Based Access Control) — абстракцию над ACL, позволяющую управлять доступом через роли. Это решило проблему масштабирования: вместо сотен ACL-записей на каждого пользователя — десяток ролей, которые назначаются группам.

Эта эволюция отражает рост сложности: от «все своим доверяем» до «каждый имеет ровно те права, которые нужны для работы».

## Зачем это нужно?

### Сценарий 1: Разделение окружений

Компания имеет три окружения: `dev`, `staging`, `production`. Разработчики должны писать в `dev`, читать из `staging`, и не иметь доступа к `production`. Без авторизации — разработчик случайно подключается к продакшену, запускает тест — и удаляет реальные данные клиентов.

С ACL: `User:dev-team` имеет **Write** на `Topic:dev-*` и **Read** на `Topic:staging-*`. Нет прав на `Topic:production-*`. Попытка подключения отклонена. Лог: «Principal User:dev-team is Denied Operation Read from host 10.0.0.15 on resource Topic:production-orders».

### Сценарий 2: Разделение по командам

Команда `payments` работает с топиками `payments-*`. Команда `analytics` — с `analytics-*` и `metrics`. Команда `logistics` — с `shipping-*`. Без авторизации — аналитик из любопытства подключается к `payments-received`, получает данные всех транзакций. Это PCI DSS violation, коммерческая тайна, риск утечки.

С ACL: `User:analytics-app` имеет **Read** только на `Topic:analytics-*` и `Topic:metrics`. Попытка чтения `payments-received` — отклонена. Аналитик видит ошибку в логах, обращается к администратору. Администратор создаёт специальный anonymized топик `payments-anonymized` с маскированными данными — и даёт на него права.

### Сценарий 3: Compliance и аудит

Стандарт PCI DSS требует: только определённые сервисы могут обрабатывать данные карт. Авторизация гарантирует: только `payment-processor` может **Read** и **Write** в `Topic:card-transactions`. `User:fraud-detection` имеет **Read** на `Topic:card-transactions` — для анализа, но не может писать. `User:notification-service` не имеет доступа вообще.

При аудите: администратор выгружает ACL и показывает: «Вот список субъектов, вот их права, вот объекты. Только payment-processor может модифицировать данные карт.» Это доказательство compliance.

## Основные концепции

### ACL — Access Control Lists

ACL — это записи в формате: **Principal** + **Operation** + **ResourceType** + **ResourceName** + **Host** + **PermissionType** (Allow/Deny).

| Компонент | Описание | Примеры |
|-----------|----------|---------|
| **Principal** | Кто | User:alice, Group:admins |
| **Operation** | Что делает | Read, Write, Create, Delete, Alter, Describe, ClusterAction |
| **ResourceType** | С чем работает | Topic, Group, Cluster, TransactionalId, DelegationToken |
| **ResourceName** | Имя ресурса | orders, payments, * (wildcard) |
| **Host** | Откуда | 10.0.0.5, * (любой) |
| **PermissionType** | Разрешить или запретить | Allow, Deny |

**Пример ACL:**
```
Principal: User:payments-app
Operation: Write
ResourceType: Topic
ResourceName: payments-received
Host: *
PermissionType: Allow
```

### Resource Types

| Тип | Описание | Операции |
|-----|----------|----------|
| **Topic** | Топик сообщений | Read, Write, Create, Delete, Alter, Describe |
| **Group** | Consumer group | Read, Describe, Delete |
| **Cluster** | Кластер целиком | Create, ClusterAction, Describe, Alter |
| **TransactionalId** | Транзакция | Describe, Write |
| **DelegationToken** | Токен делегирования | Describe |

### Operations

| Операция | Описание | Риск |
|----------|----------|------|
| **Read** | Чтение сообщений | Утечка данных |
| **Write** | Запись сообщений | Подделка, мусор |
| **Create** | Создание ресурса | Расширение поверхности |
| **Delete** | Удаление ресурса | Потеря данных |
| **Alter** | Изменение конфигурации | Нестабильность |
| **Describe** | Чтение метаданных | Разведка |
| **ClusterAction** | Административные | Полный контроль |

### Wildcards и Patterns

- **Exact match** — `Topic:orders` — только этот топик
- **Prefixed** — `Topic:payments-` — все топики с префиксом
- **Wildcard** — `Topic:*` — все топики (опасно!)

**Deny имеет приоритет над Allow.** Если одна запись Allow, а другая Deny — Deny побеждает.

### Управление ACL через CLI

```
# Добавить ACL
kafka-acls --bootstrap-server kafka:9093 \
  --add --allow-principal User:alice \
  --operation Read --topic orders \
  --command-config admin.properties

# Удалить ACL
kafka-acls --bootstrap-server kafka:9093 \
  --remove --allow-principal User:alice \
  --operation Read --topic orders

# Список ACL для топика
kafka-acls --bootstrap-server kafka:9093 \
  --list --topic orders

# Deny (запретить)
kafka-acls --bootstrap-server kafka:9093 \
  --add --deny-principal User:hacker \
  --operation All --topic *

# Префиксный ACL
kafka-acls --bootstrap-server kafka:9093 \
  --add --allow-principal User:payments-app \
  --operation Write --topic payments- \
  --resource-pattern-type Prefixed
```

### Встроенный авторизатор

**AclAuthorizer** — стандартный авторизатор Kafka:

```
# server.properties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Запретить доступ по умолчанию (если нет ACL)
allow.everyone.if.no.acl.found=false

# Суперпользователи (обходят ACL)
super.users=User:admin;User:ops-team
```

**Принцип наименьших привилегий:**
- Producer: **Write, Create** на топик — но не **Read, Delete, Alter**
- Consumer: **Read, Describe** на топик — но не **Write, Create, Delete**
- Admin: **All** на Cluster — только операционная команда
- Monitoring: **Describe** на всё — для метрик, не для данных

## RBAC — Role-Based Access Control

### Что такое RBAC

Вместо управления правами для каждого пользователя — создаём **роли** и назначаем пользователей в роли. Это абстракция, упрощающая управление в масштабе.

| Роль | Права | Пользователи |
|------|-------|--------------|
| **Producer** | Write на payments-* | payments-app, billing-app |
| **Consumer** | Read на payments-* | analytics-app, reporting-app |
| **Admin** | All на Cluster | ops-team, security-team |
| **ReadOnly** | Read, Describe на * | monitoring, audit |

### RBAC в Confluent Platform

```
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
```

### RBAC vs ACL

| Критерий | ACL | RBAC |
|----------|-----|------|
| **Управление** | По пользователю/ресурсу | По роли |
| **Масштабирование** | Сложно (N×M записей) | Просто (роли × пользователи) |
| **Аудит** | Сложнее | Проще (роли — высокоуровнево) |
| **Изменение прав** | Много записей | Одна роль |
| **Kafka Open Source** | ✅ Да | ❌ Нет (Confluent) |
| **Стандарт** | Kafka native | Реализация Confluent |

**Рекомендация:** Для Kafka Open Source использовать ACL с naming convention и автоматизацией. Для Confluent Platform — RBAC для упрощения.

## Сравнение подходов авторизации

| Подход | Масштаб | Сложность | Аудит | Гибкость |
|--------|---------|-----------|-------|----------|
| **No authorization** | Нет | Низкая | Нет | Максимальная |
| **ACL (native)** | Средний | Средняя | Хороший | Хорошая |
| **RBAC (Confluent)** | Большой | Низкая | Отличный | Хорошая |
| **OPA (Open Policy Agent)** | Любой | Высокая | Отличный | Максимальная |
| **IAM integration** | Большой | Средняя | Отличный | Зависит от IAM |

## Уроки из реальных инцидентов

### Инцидент 1: Удаление топика через ACL-лазейку (2019)

Компания настроила ACL: `User:app` имел **Write** на `Topic:orders`. Но администратор забыл ограничить **Delete**. Тестовый скрипт с учётными данными `User:app` (утёкшие в Git) выполнил `--delete --topic orders`.

**Хронология:** Утёчка credentials через публичный GitHub репозиторий. Злоумышленник нашёл username/password SCRAM. Подключился. Выполнил delete. Топик удалён. Данные потеряны (retention был 7 дней, backup — раз в сутки, последний backup — 12 часов назад).

**Ущерб:** Потеря 12 часов заказов. Ручное восстановление. Штрафы клиентам за задержку. $200,000 ущерб.

**Как защита изменила бы исход:** Принцип наименьших привилегий — `User:app` должен иметь только **Write**, не **Delete**. Административные операции — только для `User:admin` с **ClusterAction**. Аудит изменений ACL — «кто дал права Delete?». Git secret scanning — предотвращение утечки credentials.

### Инцидент 2: Несанкционированный доступ через wildcard ACL (2021)

Администратор для удобства создал ACL: `User:analytics-team` → **Read** → `Topic:*`. Вся команда аналитики могла читать все топики, включая `salaries`, `customer-pii`, `financial-reports`.

**Хронология:** Аналитик, уволившийся через месяц, забрал с собой данные. Обнаружилось при exit interview, когда он случайно упомянул зарплаты коллег.

**Ущерб:** Утечка конфиденциальных данных. Юридические последствия. Потеря доверия сотрудников.

**Как защита изменила бы исход:** Никаких wildcard ACL без явного approval. Разделение по префиксам: `analytics-*` — для аналитики, `payments-*` — для платежей. Аналитика получает **Read** только на `Topic:analytics-*`. Для sensitive данных — отдельный процесс с anonymization.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **ACL (native)** | Авторизация на уровне Kafka | Настройка кластера | NIST AC-3 |
| **RBAC (Confluent)** | Ролевая модель | Enterprise | NIST AC-2 |
| **OPA** | Policy-as-code | GitOps, Kubernetes | NIST AC-3 |
| **Apache Ranger** | Централизованная авторизация | Enterprise Hadoop/Kafka | NIST AC-2 |
| **Git + Terraform** | IaC для ACL | CI/CD | NIST CM-3 |
| **SIEM** | Аудит изменений ACL | Runtime | NIST AU-6 |

## Архитектурные решения

### Defense-in-Depth для авторизации

| Слой | Механизм | Контроль |
|------|----------|----------|
| Network | Firewall, VPC | Только нужные IP |
| Authentication | SASL_SSL | Каждый идентифицирован |
| Authorization | ACL/RBAC | Только необходимые права |
| Data | Encryption at-rest | Даже при доступе к диску |
| Audit | Логирование | Кто, когда, что |

### GitOps для ACL

```
# Определяем ACL в YAML
acl:
  - principal: User:payments-app
    operation: Write
    resource:
      type: Topic
      name: payments-received
      pattern: Prefixed

# Terraform / Ansible применяет
# Git history = audit trail
# PR = change approval
# CI/CD = automated deployment
```

### Метрики авторизации

| Метрика | Целевое значение | Действие |
|---------|-----------------|----------|
| Denied operations | < 10 за час | Алерт при превышении |
| Wildcard ACL | 0 | Немедленное удаление |
| Superuser operations | Логировать все | Ревью еженедельно |
| ACL changes | Алерт | Кто и зачем изменил |

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «ACL и RBAC — одно и то же».

ACL — низкоуровневые записи (Principal → Operation → Resource). RBAC — абстракция (Role → Permissions → Users). RBAC проще масштабировать, ACL более гибкий. В Kafka Open Source — только ACL.

**Ошибка 2:** «Allow everyone по умолчанию безопасно».

`allow.everyone.if.no.acl.found=true` означает: если нет ACL для ресурса — доступ открыт всем. Это опасно. Должно быть `false` — deny по умолчанию.

**Ошибка 3:** «Deny ACL не нужен, достаточно не давать Allow».

Deny имеет приоритет над Allow. Если дать **Allow All** группе и **Deny Write** конкретному пользователю — пользователь может всё, кроме Write. Без Deny — пришлось бы перечислять все Allow для всех, кроме него.

### Три ситуационных вопроса

**Вопрос 1:** «Как управлять ACL в крупной организации с сотнями топиков и тысячами пользователей?»

Ответ: **Naming convention** — `team-purpose-env` (например, `payments-received-prod`). **Prefixed ACL** — дать команде права на `payments-*-prod`. **RBAC** — если Confluent. **GitOps** — ACL как код, версионирование, approval через PR. **Automation** — скрипты создания ACL при создании топика. **Regular audit** — скрипт проверки wildcard, избыточных прав, stale ACL.

**Вопрос 2:** «Как защититься от privilege escalation через новые топики?»

Ответ: **Default deny** — новый топик не доступен никому, пока ACL не созданы. **Auto-ACL** — при создании топика через GitOps автоматически создаются ACL для owning team. **Create permission restricted** — только `User:topic-admin` может создавать топики. **Review process** — создание топика требует approval.

**Вопрос 3:** «Как аудитор может проверить, что authorization работает корректно?»

Ответ: Аудитор запрашивает: список всех ACL (`kafka-acls --list`), список суперпользователей, логи изменений ACL, тестовые подключения с разными credentials. Проверяет: есть ли wildcard ACL, есть ли stale ACL (на уволенных сотрудников), есть ли overprivileged users. Сравнивает с политикой least privilege.

## Чек-лист понимания

- [ ] Чем **авторизация** отличается от **аутентификации**?
- [ ] Какие **ResourceType** существуют в Kafka?
- [ ] Какие **Operation** можно разрешать?
- [ ] Что такое **Prefixed ACL**?
- [ ] Имеет ли **Deny** приоритет над **Allow**?
- [ ] Что такое **super.users**?
- [ ] Чем **RBAC** отличается от **ACL**?
- [ ] Как управлять ACL в **Kafka Open Source**?
- [ ] Какие риски у **wildcard ACL**?
- [ ] Как **audit** изменения ACL?

### Ответы на чек-лист

1. **Авторизация** — проверка прав (что можно?). **Аутентификация** — проверка личности (кто ты?). Сначала аутентификация, потом авторизация.

2. **ResourceType:** Topic, Group, Cluster, TransactionalId, DelegationToken.

3. **Operation:** Read, Write, Create, Delete, Alter, Describe, ClusterAction.

4. **Prefixed ACL** — права на все ресурсы с определённым префиксом. Например, `Topic:payments-*` — все топики, начинающиеся с "payments-".

5. **Да**, Deny имеет приоритет над Allow. Если есть и Allow, и Deny — Deny побеждает.

6. **super.users** — список пользователей, обходящих ACL. Они имеют права на всё. Использовать минимально, только для операционной команды.

7. **RBAC** — управление через роли. **ACL** — низкоуровневые записи principal-operation-resource. RBAC проще масштабировать, ACL более гибкий. RBAC — Confluent, ACL — Open Source.

8. В **Open Source** — через `kafka-acls` CLI, JAAS, или программно через AdminClient API. Рекомендуется автоматизация через GitOps/Terraform.

9. **Wildcard ACL** (`Topic:*`) дают доступ ко всему. Риск: неожиданный доступ к новым/чужим топикам. Рекомендация — никаких wildcard без явного approval.

10. Аудит изменений ACL: логировать команды `kafka-acls`, использовать Git history для ACL-as-code, настроить SIEM алерты на изменения.

## Ключевые выводы для собеседования

- **Авторизация — вторая линия обороны** — после аутентификации
- **Deny по умолчанию** — `allow.everyone.if.no.acl.found=false`
- **Least privilege** — только необходимые права, никаких wildcard без причины
- **Separate admin roles** — не давать приложениям прав на ClusterAction
- **ACL as code** — версионирование, review, automated deployment

---

_Статья создана на основе анализа материалов по безопасности Kafka_
