# Авторизация в Kafka

## Что такое авторизация?

Представь библиотеку. Ты прошёл паспортный контроль (аутентификация). Теперь библиотекарь спрашивает: "Какой у тебя пропуск?" Студент — читальный зал. Преподаватель — абонемент. Администратор — всё.

**Авторизация в Kafka** — это именно такой "пропуск". Брокер знает, кто ты. Теперь решает: какие топики можешь читать, какие — писать, какие — администрировать.

## ACL (Access Control Lists)

### Структура

| Поле | Описание | Пример |
|------|----------|--------|
| Principal | Кто | User:alice |
| Host | Откуда | * |
| Operation | Что делать | Read, Write |
| ResourceType | Что | Topic, Group |
| ResourceName | Название | my-topic |
| PermissionType | Разрешить/запретить | Allow, Deny |

### Команды

```bash
# Дать права на чтение
kafka-acls --bootstrap-server localhost:9092 \
  --add --allow-principal User:alice \
  --operation Read --topic my-topic

# Дать права на запись
kafka-acls --bootstrap-server localhost:9092 \
  --add --allow-principal User:bob \
  --operation Write --topic my-topic

# Запретить
kafka-acls --bootstrap-server localhost:9092 \
  --add --deny-principal User:mallory \
  --operation Read --topic my-topic

# Показать все ACL
kafka-acls --bootstrap-server localhost:9092 --list
```

### Операции

| Операция | Описание |
|----------|----------|
| Read | Чтение |
| Write | Запись |
| Create | Создание |
| Delete | Удаление |
| Alter | Изменение |
| Describe | Описание |
| ClusterAction | Кластерные |
| DescribeConfigs | Конфиги |
| AlterConfigs | Изменение конфигов |
| IdempotentWrite | Идемпотентная запись |

### Ресурсы

| Ресурс | Описание |
|--------|----------|
| Topic | Топик |
| Group | Consumer group |
| Cluster | Кластер |
| TransactionalId | Транзакция |
| DelegationToken | Токен |

## RBAC (Role-Based Access Control)

### Confluent RBAC

| Роль | Описание |
|------|----------|
| SystemAdmin | Всё |
| ClusterAdmin | Кластер |
| DeveloperRead | Чтение |
| DeveloperWrite | Запись |
| DeveloperManage | Управление |
| ResourceOwner | Владелец ресурса |

### Назначение ролей

```bash
confluent iam rbac role-binding create \
  --principal User:alice \
  --role DeveloperRead \
  --resource Topic:my-topic \
  --kafka-cluster-id my-cluster
```

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Принцип наименьших привилегий | Только необходимое |
| Группировка по ролям | Не индивидуально |
| Регулярный аудит | Проверка ACL |
| Запрет по умолчанию | allow.everyone.if.no.acl.found=false |
| Мониторинг | Кто что делает |

## Чек-лист понимания

- [ ] Что такое ACL?
- [ ] Какая структура ACL?
- [ ] Какие операции?
- [ ] Какие ресурсы?
- [ ] Как дать права?
- [ ] Как запретить?
- [ ] Что такое RBAC?
- [ ] Какие роли?
- [ ] Какие лучшие практики?
- [ ] Почему least privilege?

### Ответы на чек-лист

1. **ACL** — Access Control List. Список правил доступа.

2. **Структура**: Principal, Host, Operation, ResourceType, ResourceName, PermissionType.

3. **Операции**: Read, Write, Create, Delete, Alter, Describe, ClusterAction.

4. **Ресурсы**: Topic, Group, Cluster, TransactionalId, DelegationToken.

5. **Дать права**: kafka-acls --add --allow-principal User:X --operation Read --topic Y.

6. **Запретить**: kafka-acls --add --deny-principal User:X --operation Read --topic Y.

7. **RBAC** — Role-Based Access Control. Доступ через роли.

8. **Роли**: SystemAdmin, ClusterAdmin, DeveloperRead, DeveloperWrite, DeveloperManage, ResourceOwner.

9. **Лучшие практики**: least privilege, группировка, аудит, запрет по умолчанию, мониторинг.

10. **Least privilege** — минимум прав. Меньше прав = меньше риск.

---

_Статья создана на основе анализа материалов Habr по Kafka_
