# Аутентификация и авторизация в Java

## Что такое аутентификация и авторизация?

Представь клуб. На входе стоит охранник. Он спрашивает у тебя: "Кто ты?" Ты показываешь паспорт. Охранник сверяет: фото совпадает? Да. Это **аутентификация** — доказательство, что ты — это ты.

Потом охранник спрашивает: "У тебя есть приглашение?" Ты показываешь билет. На нём написано: "VIP". Охранник говорит: "Можешь в VIP-зону. В закрытую комнату — нет. На танцпол — да". Это **авторизация** — определение, что тебе разрешено.

**Аутентификация** = кто ты?
**Авторизация** = что тебе можно?

## Зачем нужны аутентификация и авторизация?

### Ситуация 1: Вход в HR-платформу

Пользователь вводит логин и пароль. Система проверяет: правильный ли пароль? (аутентификация). Затем проверяет: HR-менеджер или кандидат? (авторизация). HR-менеджер видит все резюме. Кандидат — только своё.

### Ситуация 2: API для партнёров

Партнёр вызывает API. Нужно проверить: действительно ли это партнёр? (аутентификация). Имеет ли он право на этот endpoint? (авторизация).

### Ситуация 3: SSO

Сотрудник входит в 10 систем. Не вводит 10 паролей. Вводит один раз. Системы доверяют друг другу.

## Аутентификация

### Способы аутентификации

| Способ | Описание | Пример |
|--------|----------|--------|
| Пароль | Знание секрета | login/password |
| Токен | Владение секретом | JWT, API key |
| Biometric | Что ты есть | Отпечаток, лицо |
| Certificate | Криптографический | TLS client cert |
| OTP | Одноразовый код | SMS, TOTP |
| Social | OAuth2 | Google, Facebook |

### Многофакторная аутентификация (MFA)

| Фактор | Примеры |
|--------|---------|
| Знание | Пароль, PIN |
| Владение | Телефон, токен |
| Наследие | Отпечаток, лицо |
| Место | Геолокация |
| Время | Время суток |

### Spring Security Authentication

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http.formLogin(form -> form
            .loginPage("/login")
            .defaultSuccessUrl("/dashboard")
        );
        return http.build();
    }
}
```

## Авторизация

### RBAC

| Роль | Права |
|------|-------|
| USER | Читать своё |
| HR | Читать все |
| ADMIN | Всё |

### ABAC

| Атрибут | Условие |
|---------|---------|
| department | == "HR" |
| time | 9:00 - 18:00 |
| location | Внутри офиса |

### Spring Security Authorization

```java
@PreAuthorize("hasRole('ADMIN') or hasRole('HR')")
public List<Resume> getAllResumes() { ... }

@PreAuthorize("@resumeService.isOwner(#id, authentication.name)")
public Resume getResume(@PathVariable Long id) { ... }
```

## OAuth2

### Роли

| Роль | Описание |
|------|----------|
| Resource Owner | Пользователь |
| Client | Приложение |
| Authorization Server | Выдаёт токены |
| Resource Server | API с данными |

### Flows

| Flow | Когда |
|------|-------|
| Authorization Code | Веб-приложения |
| PKCE | Мобильные, SPA |
| Client Credentials | Server-to-server |
| Device Code | ТВ, консоли |

### JWT

| Часть | Описание |
|-------|----------|
| Header | Алгоритм |
| Payload | Данные (claims) |
| Signature | Подпись |

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyIiwicm9sZSI6IkFEU0RJTiJ9.xxx
```

## SSO (Single Sign-On)

### Протоколы

| Протокол | Описание |
|----------|----------|
| SAML | XML, enterprise |
| OAuth2 | Токены, современный |
| OpenID Connect | Identity layer над OAuth2 |
| Kerberos | Windows, сеть |

### Spring Security + OIDC

```java
http.oauth2Login(oauth2 -> oauth2
    .loginPage("/login")
    .defaultSuccessUrl("/dashboard")
);
```

## Лучшие практики

| Практика | Описание |
|----------|----------|
| MFA | Включить всегда |
| HTTPS | Никогда не передавать пароли по HTTP |
| BCrypt | Для хеширования паролей |
| JWT | Короткий срок жизни (15-30 мин) |
| Refresh token | Для обновления |
| Session fixation | Менять ID после логина |
| Logout | Инвалидировать токены |
| Rate limiting | На попытки входа |

## Чек-лист понимания

- [ ] Чем аутентификация отличается от авторизации?
- [ ] Какие способы аутентификации?
- [ ] Что такое MFA?
- [ ] Какие факторы MFA?
- [ ] Что такое RBAC?
- [ ] Что такое ABAC?
- [ ] Какие flow в OAuth2?
- [ ] Что такое JWT?
- [ ] Что такое SSO?
- [ ] Какие лучшие практики?

### Ответы на чек-лист

1. **Аутентификация** — кто ты? **Авторизация** — что тебе можно?

2. **Способы**: пароль, токен, биометрия, сертификат, OTP, social login.

3. **MFA** — многофакторная аутентификация. Несколько факторов подтверждения.

4. **Факторы**: знание (пароль), владение (телефон), наследие (отпечаток), место, время.

5. **RBAC** — Role-Based Access Control. Доступ на основе ролей.

6. **ABAC** — Attribute-Based Access Control. Доступ на основе атрибутов.

7. **OAuth2 flows**: Authorization Code, PKCE, Client Credentials, Device Code.

8. **JWT** — JSON Web Token. Содержит Header, Payload, Signature.

9. **SSO** — Single Sign-On. Один вход во множество систем.

10. **Лучшие практики**: MFA, HTTPS, BCrypt, короткий JWT, refresh token, session fixation, logout, rate limiting.

---

_Статья создана на основе анализа материалов Habr по аутентификации и авторизации_
