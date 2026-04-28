# OWASP Top 10 для Java

## Что такое OWASP Top 10?

Представь список самых опасных преступлений в городе. Полиция анализирует статистику: какие преступления чаще всего? Какие причиняют больше всего вреда? Составляет топ-10. И говорит: "Вот на что обращайте внимание в первую очередь".

**OWASP Top 10** — это именно такой список для кибербезопасности. Организация OWASP анализирует уязвимости по всему миру. Выбирает 10 самых опасных. Обновляет каждые 3-4 года. Последняя версия — 2021.

## Зачем нужен OWASP Top 10?

### Ситуация 1: Новый проект

Команда начинает разработку HR-платформы на Java. Какие угрозы наиболее вероятны? На что тратить время?

OWASP Top 10 — приоритизированный список.

### Ситуация 2: Аудит

Аудитор проверяет приложение. Использует OWASP Top 10 как чек-лист.

### Ситуация 3: Обучение

Новый разработчик. Нужно быстро понять: чего опасаться?

## OWASP Top 10 2021

### A01: Broken Access Control

| Описание | Пример |
|----------|--------|
| Пользователь может получить доступ к чужим данным | Изменение ID в URL: /resume?id=123 → /resume?id=124 |

**Java-защита:**
- Проверять права на каждый запрос
- Использовать Spring Security @PreAuthorize
- Не полагаться на client-side проверки

### A02: Cryptographic Failures

| Описание | Пример |
|----------|--------|
| Данные передаются/хранятся без шифрования | HTTP вместо HTTPS |

**Java-защита:**
- Всегда HTTPS
- Шифровать sensitive данные
- Не хранить пароли в plaintext (использовать BCrypt)

### A03: Injection

| Описание | Пример |
|----------|--------|
| Вредоносный код внедряется в запрос | SQL injection: "'; DROP TABLE users; --" |

**Java-защита:**
- PreparedStatement вместо конкатенации
- ORM (Hibernate) с параметрами
- Input validation

### A04: Insecure Design

| Описание | Пример |
|----------|--------|
| Проблемы в архитектуре | Бизнес-логика без проверок |

**Java-защита:**
- Threat modeling
- Security by design
- Defense in depth

### A05: Security Misconfiguration

| Описание | Пример |
|----------|--------|
| Неправильные настройки | Debug mode включён в production |

**Java-защита:**
- application.properties: spring.profiles.active=prod
- Отключить stack trace
- Удалить тестовые данные

### A06: Vulnerable and Outdated Components

| Описание | Пример |
|----------|--------|
| Уязвимые зависимости | Log4j 2.14.1 (CVE-2021-44228) |

**Java-защита:**
- OWASP Dependency Check
- Snyk, Dependabot
- Регулярное обновление

### A07: Identification and Authentication Failures

| Описание | Пример |
|----------|--------|
| Проблемы с аутентификацией | Нет MFA, слабые пароли |

**Java-защита:**
- Spring Security с MFA
- Блокировка после N попыток
- Сложные пароли

### A08: Software and Data Integrity Failures

| Описание | Пример |
|----------|--------|
| Нарушение целостности | Загрузка плагинов без проверки |

**Java-защита:**
- Подпись кода (JAR signing)
- Проверка checksum
- SRI (Subresource Integrity)

### A09: Security Logging and Monitoring Failures

| Описание | Пример |
|----------|--------|
| Нет логов, нет мониторинга | Атака не обнаружена |

**Java-защита:**
- SLF4J/Logback для логов
- ELK/Splunk для анализа
- Алерты на аномалии

### A10: Server-Side Request Forgery (SSRF)

| Описание | Пример |
|----------|--------|
| Сервер делает запросы от имени атакующего | Загрузка URL, предоставленного пользователем |

**Java-защита:**
- Валидация URL
- Whitelist разрешённых хостов
- Не доверять пользовательскому вводу

## Сравнение с предыдущей версией

| 2017 | 2021 | Что изменилось |
|------|------|----------------|
| Injection | Injection | Остался #1 по факту |
| Broken Authentication | Identification failures | Расширили |
| Sensitive Data Exposure | Cryptographic failures | Уточнили |
| XML External Entities | — | Убрали (редко) |
| Broken Access Control | Broken Access Control | Подняли на #1 |
| Security Misconfiguration | Security Misconfiguration | Остался |
| XSS | — | В Injection |
| Insecure Deserialization | Insecure Design | Расширили |
| Using Components | Vulnerable Components | То же |
| Insufficient Logging | Logging Failures | То же |
| — | SSRF | Новый |

## Инструменты для Java

| Инструмент | Что делает |
|------------|------------|
| OWASP ZAP | Сканирование веб-приложений |
| SonarQube | Статический анализ |
| Dependency Check | Проверка зависимостей |
| SpotBugs + FindSecBugs | Поиск уязвимостей |
| Burp Suite | Pentest |

## Чек-лист понимания

- [ ] Что такое OWASP Top 10?
- [ ] Какие 10 угроз 2021?
- [ ] Чем A01 отличается от A07?
- [ ] Как защититься от Injection?
- [ ] Что такое SSRF?
- [ ] Как проверять зависимости?
- [ ] Что изменилось с 2017?
- [ ] Какие инструменты для Java?
- [ ] Что такое SRI?
- [ ] Как Spring Security помогает?

### Ответы на чек-лист

1. **OWASP Top 10** — список 10 самых критичных угроз веб-безопасности. Обновляется каждые 3-4 года.

2. **10 угроз 2021**: Broken Access Control, Cryptographic Failures, Injection, Insecure Design, Security Misconfiguration, Vulnerable Components, Authentication Failures, Integrity Failures, Logging Failures, SSRF.

3. **A01** — нарушение контроля доступа (IDOR, path traversal). **A07** — проблемы с аутентификацией (MFA, пароли).

4. **Защита от Injection**: PreparedStatement, ORM, input validation.

5. **SSRF** — Server-Side Request Forgery. Сервер делает запросы по URL от пользователя.

6. **Проверка зависимостей**: OWASP Dependency Check, Snyk, Dependabot.

7. **Изменения с 2017**: SSRF новый, XSS в Injection, XML External Entities убрали.

8. **Инструменты**: ZAP, SonarQube, Dependency Check, SpotBugs, Burp Suite.

9. **SRI** — Subresource Integrity. Проверка целостности загружаемых ресурсов.

10. **Spring Security** помогает: аутентификация, авторизация, CSRF, XSS, session management.

---

_Статья создана на основе анализа материалов Habr по OWASP Top 10_
