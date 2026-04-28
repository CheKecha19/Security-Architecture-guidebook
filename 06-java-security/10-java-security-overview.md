# Обзор архитектуры Java Security

## Что мы изучили?

В этом разделе мы прошли путь от JVM до Spring Security. От базовых механизмов до современных угроз.

## Карта знаний

### 1. Основы (Статья 01)

| Тема | Ключевое |
|------|----------|
| JVM | Bytecode verification |
| ClassLoader | Иерархия загрузки |
| SecurityManager | Policy, permissions |
| AccessController | Доступ к ресурсам |

### 2. Spring Security (Статья 02)

| Тема | Ключевое |
|------|----------|
| Фильтры | SecurityFilterChain |
| Аутентификация | Форма, JWT, OAuth2 |
| Авторизация | @PreAuthorize, RBAC |
| Защита | CSRF, XSS, clickjacking |

### 3. OWASP Top 10 (Статья 03)

| Тема | Ключевое |
|------|----------|
| A01 | Broken Access Control |
| A03 | Injection |
| A06 | Vulnerable Components |
| A10 | SSRF |

### 4. Аутентификация и авторизация (Статья 04)

| Тема | Ключевое |
|------|----------|
| MFA | Многофакторная |
| JWT | Stateless токены |
| OAuth2 | Delegated access |
| SSO | Single Sign-On |

### 5. Управление сессиями (Статья 05)

| Тема | Ключевое |
|------|----------|
| Сессии | Cookie, token |
| Угрозы | Fixation, hijacking |
| CSRF | Токены, SameSite |
| Spring Session | Redis, JDBC |

### 6. Валидация (Статья 06)

| Тема | Ключевое |
|------|----------|
| Whitelist | Разрешить хорошее |
| Bean Validation | @NotNull, @Size |
| SQL Injection | PreparedStatement |
| XSS | Output encoding |

### 7. Безопасное программирование (Статья 07)

| Тема | Ключевое |
|------|----------|
| Defense in Depth | Многослойность |
| Fail Secure | Безопасный отказ |
| Zero Trust | Не доверяй |
| Инструменты | SonarQube, SpotBugs |

### 8. Криптография (Статья 08)

| Тема | Ключевое |
|------|----------|
| Хеширование | SHA-256, BCrypt |
| Шифрование | AES-256-GCM |
| Асимметрия | RSA-2048+ |
| TLS | Версии 1.2, 1.3 |

### 9. Зависимости (Статья 09)

| Тема | Ключевое |
|------|----------|
| SCA | Dependency Check |
| Транзитивные | Проверять глубину |
| Обновление | Регулярно |
| Лицензии | Проверять |

## Сравнение подходов

| Подход | Когда | Пример |
|--------|-------|--------|
| Валидация | Всегда | Bean Validation |
| Аутентификация | Вход | Spring Security |
| Авторизация | Каждый запрос | @PreAuthorize |
| Шифрование | Данные | AES, BCrypt |
| Логирование | Всегда | SLF4J |
| Мониторинг | Всегда | SIEM |

## Архитектурные решения

### Безопасное Java-приложение

```
Client → WAF → Load Balancer → Spring Security → Application → Database
              HTTPS              JWT/Session         Validation    Encrypted
              TLS                RBAC                Sanitization  Backup
```

### Security Pipeline

```
Code → SAST (SonarQube) → Build → SCA (Dependency Check) → Deploy → DAST (ZAP)
```

## Чек-лист понимания

- [ ] Какие механизмы JVM?
- [ ] Как Spring Security работает?
- [ ] Какие OWASP угрозы?
- [ ] Как реализовать MFA?
- [ ] Как защитить сессии?
- [ ] Как валидировать входные данные?
- [ ] Какие принципы безопасного кода?
- [ ] Какие алгоритмы шифрования?
- [ ] Как проверять зависимости?
- [ ] Как построить pipeline?

### Ответы на чек-лист

1. **JVM**: bytecode verification, ClassLoader, SecurityManager, AccessController.

2. **Spring Security**: фильтры, аутентификация, авторизация, защита от CSRF/XSS.

3. **OWASP**: Broken Access Control, Injection, Vulnerable Components, SSRF.

4. **MFA**: Spring Security + TOTP, SMS, WebAuthn.

5. **Сессии**: HttpOnly, Secure, SameSite, короткий срок, изменение ID.

6. **Валидация**: whitelist, Bean Validation, PreparedStatement, output encoding.

7. **Принципы**: defense in depth, fail secure, least privilege, zero trust.

8. **Шифрование**: BCrypt (пароли), AES-256-GCM (данные), RSA-2048+ (асимметрия), TLS 1.3 (связь).

9. **Зависимости**: OWASP Dependency Check, Snyk, Dependabot, регулярное обновление.

10. **Pipeline**: SAST → Build → SCA → Deploy → DAST.

---

_Статья создана на основе анализа материалов Habr по Java Security_
