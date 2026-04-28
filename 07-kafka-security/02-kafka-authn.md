# Аутентификация в Kafka

## Зачем нужна аутентификация?

Представь почтовое отделение. Каждый может зайти и забрать любое письмо? Нет. Нужен паспорт. Почтальон проверяет: ты — адресат?

**Аутентификация в Kafka** — это именно такая проверка паспорта. Каждый клиент (producer, consumer) должен доказать, кто он. Брокер не принимает анонимов.

## SASL (Simple Authentication and Security Layer)

### PLAIN

| Параметр | Описание |
|----------|----------|
| Механизм | Логин/пароль |
| Безопасность | Низкая без TLS |
| Использование | Только с SASL_SSL |

```
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="alice" password="alice-secret";
```

### SCRAM-SHA-256 / SCRAM-SHA-512

| Параметр | Описание |
|----------|----------|
| Механизм | Salted Challenge Response |
| Безопасность | Высокая |
| Использование | Рекомендуется |

```
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="alice" password="alice-secret";
```

### GSSAPI (Kerberos)

| Параметр | Описание |
|----------|----------|
| Механизм | Kerberos |
| Безопасность | Высокая |
| Использование | Enterprise |

```
sasl.mechanism=GSSAPI
sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
  useKeyTab=true keyTab="/path/to/alice.keytab" principal="alice@EXAMPLE.COM";
```

### OAUTHBEARER

| Параметр | Описание |
|----------|----------|
| Механизм | OAuth2 |
| Безопасность | Высокая |
| Использование | Современные системы |

## SSL / TLS

### Односторонний SSL

| Что | Описание |
|-----|----------|
| Брокер | Сертификат |
| Клиент | Проверяет сертификат |
| Аутентификация | Нет |

### Двусторонний SSL (mTLS)

| Что | Описание |
|-----|----------|
| Брокер | Сертификат |
| Клиент | Сертификат |
| Аутентификация | Да |

```
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=secret
ssl.key.password=secret
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=secret
ssl.client.auth=required
```

## Настройка брокера

```
listeners=SASL_SSL://:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

## Лучшие практики

| Практика | Описание |
|----------|----------|
| SCRAM-SHA-512 | Предпочтительный механизм |
| TLS | Всегда |
| mTLS | Для высокой безопасности |
| Не PLAIN без TLS | Пароль в открытом виде |
| Ротация | Регулярная смена паролей |
| Хранение | Не в коде, в secrets |

## Чек-лист понимания

- [ ] Что такое SASL?
- [ ] Какие механизмы?
- [ ] Чем SCRAM лучше PLAIN?
- [ ] Что такое mTLS?
- [ ] Как настроить SCRAM?
- [ ] Как настроить Kerberos?
- [ ] Как настроить OAuth?
- [ ] Какой механизм выбрать?
- [ ] Какие лучшие практики?
- [ ] Почему не PLAIN?

### Ответы на чек-лист

1. **SASL** — Simple Authentication and Security Layer. Фреймворк аутентификации.

2. **Механизмы**: PLAIN, SCRAM-SHA-256/512, GSSAPI, OAUTHBEARER.

3. **SCRAM лучше PLAIN**, потому что пароль не передаётся в открытом виде.

4. **mTLS** — mutual TLS. Клиент и сервер проверяют друг друга сертификатами.

5. **SCRAM настройка**: sasl.mechanism=SCRAM-SHA-512, sasl.jaas.config.

6. **Kerberos**: GSSAPI, keytab, principal.

7. **OAuth**: OAUTHBEARER, токен.

8. **Выбор**: SCRAM-SHA-512 для простоты, mTLS для высокой безопасности, Kerberos для enterprise.

9. **Лучшие практики**: SCRAM-SHA-512, TLS, mTLS, не PLAIN, ротация, secrets.

10. **Не PLAIN**, потому что пароль в открытом виде без TLS.

---

_Статья создана на основе анализа материалов Habr по Kafka_
