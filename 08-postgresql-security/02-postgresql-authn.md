# Аутентификация в PostgreSQL

## pg_hba.conf

| Колонка | Описание | Пример |
|---------|----------|--------|
| TYPE | local, host, hostssl | hostssl |
| DATABASE | База | all, mydb |
| USER | Пользователь | all, alice |
| ADDRESS | Сеть | 10.0.0.0/8 |
| METHOD | Метод | scram-sha-256 |

### Методы

| Метод | Описание | Безопасность |
|-------|----------|--------------|
| trust | Без пароля | Нет |
| reject | Запретить | — |
| md5 | MD5 хеш | Средняя |
| password | Открытый текст | Нет |
| scram-sha-256 | SCRAM | Высокая |
| ldap | LDAP | Высокая |
| gss | Kerberos | Высокая |
| sspi | Windows SSPI | Высокая |
| ident | OS user | Низкая |
| peer | Local OS | Низкая |
| radius | RADIUS | Средняя |
| cert | SSL certificate | Высокая |

### Примеры

`
# Локальные подключения
local   all             all                                     scram-sha-256

# Удалённые по SSL
hostssl all             all             10.0.0.0/8              scram-sha-256

# Запретить всё остальное
host    all             all             0.0.0.0/0               reject
`

## SSL

`
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_crl_file = 'root.crl'
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| scram-sha-256 | Предпочтительный метод |
| hostssl | Только SSL |
| Ограничение сети | Не 0.0.0.0/0 |
| Регулярная проверка | pg_hba.conf |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
