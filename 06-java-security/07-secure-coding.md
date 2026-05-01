# Безопасное программирование на Java

## Часть 1: Что такое безопасное программирование?

Представь строительство моста. Инженер не просто рисует красивые арки. Он считает: какой ветер, какой вес, какие материалы, какие течения, какие землетрясения возможны. Для каждого фактора — запас прочности.

**Безопасное программирование** — это именно такое «проектирование с запасом прочности» для кода. Не просто «чтобы работало», а «чтобы не сломалось при атаке».

Разница между обычным и безопасным кодом:
- Обычный код: «Если пользователь введёт правильные данные — всё работает»
- Безопасный код: «Если пользователь введёт что угодно — ничего плохого не произойдёт»

## Часть 2: Принципы безопасности

### Defense in Depth (Защита в глубину)

Никогда не полагайся на один слой защиты. Если его пробьют — система падёт.

| Слой | Защита | Пример |
|------|--------|--------|
| Сеть | Firewall, WAF | Блокировка SQL Injection на уровне сети |
| Приложение | Валидация, авторизация | Проверка входных данных |
| Данные | Шифрование, бэкап | AES-256 для sensitive данных |
| Мониторинг | Логи, SIEM | Обнаружение подозрительной активности |

**Аналогия:** Замок на двери — хорошо. Замок + сигнализация — лучше. Замок + сигнализация + камеры + охрана — надёжно.

### Fail Secure (Безопасный отказ)

При ошибке система должна перейти в безопасное состояние.

| Ситуация | Неправильно (Fail Open) | Правильно (Fail Secure) |
|----------|------------------------|------------------------|
| Ошибка аутентификации | Разрешить доступ | Запретить доступ |
| Сбой авторизации | Разрешить действие | Запретить действие |
| Невозможно проверить подпись JWT | Принять токен | Отклонить токен |
| Падение системы авторизации | Открыть все двери | Закрыть все двери |

Пример в коде:

    // ❌ НЕПРАВИЛЬНО — fail open
    try {
        if (checkPermission(user, resource)) {
            return allowAccess();
        }
    } catch (Exception e) {
        // Ошибка — разрешить?
        return allowAccess();
    }
    
    // ✅ ПРАВИЛЬНО — fail secure
    try {
        if (checkPermission(user, resource)) {
            return allowAccess();
        }
    } catch (Exception e) {
        // Ошибка — запретить
        return denyAccess();
    }
    return denyAccess();

### Least Privilege (Принцип наименьших привилегий)

Давай только необходимые права. Не больше.

| Что | Плохо | Хорошо |
|-----|-------|--------|
| Пользователь БД | `GRANT ALL` | `GRANT SELECT, INSERT` |
| Сервисный аккаунт | SUPERUSER | Ограниченные права |
| Процесс приложения | Запуск от root | Запуск от dedicated user |
| Файл логов | `chmod 777` | `chmod 640` |

### Zero Trust (Нулевое доверие)

Не доверяй никому. Проверяй каждый запрос.

| Принцип | Реализация |
|---------|------------|
| Не доверяй сети | Проверяй каждый запрос, даже изнутри |
| Минимум доступа | Только необходимое |
| Предполагай компрометацию | Что уже взломали, как минимизировать ущерб |
| Проверяй идентичность | Каждый запрос — кто ты? |
| Шифруй всё | Данные в пути и в покое |

## Часть 3: Практики безопасного кодирования

### 1. Валидация входных данных

    // ✅ Whitelist подход
    public void processOrder(String productId, int quantity) {
        // Проверка формата
        if (!productId.matches("^[A-Z0-9]{10}$")) {
            throw new IllegalArgumentException("Invalid product ID");
        }
        
        // Проверка диапазона
        if (quantity < 1 || quantity > 100) {
            throw new IllegalArgumentException("Quantity must be 1-100");
        }
        
        // Бизнес-логика
        if (!productService.exists(productId)) {
            throw new NotFoundException("Product not found");
        }
    }

### 2. Аутентификация

    // ✅ Проверяем аутентификацию на каждом уровне
    @PreAuthorize("isAuthenticated()")
    public void transferMoney(String from, String to, BigDecimal amount) {
        // Дополнительная проверка
        if (!securityContext.isAuthenticated()) {
            throw new AccessDeniedException("Not authenticated");
        }
        
        // MFA для крупных сумм
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            mfaService.verify(securityContext.getUser());
        }
    }

### 3. Авторизация

    // ✅ Проверяем права на каждом запросе
    @PreAuthorize("hasRole('ADMIN') or #account.owner == authentication.name")
    public Account getAccount(String accountId) {
        Account account = accountRepository.findById(accountId);
        
        // Дополнительная проверка (defense in depth)
        if (!account.getOwner().equals(getCurrentUser())) {
            audit.log("UNAUTHORIZED_ACCESS", accountId);
            throw new AccessDeniedException();
        }
        
        return account;
    }

### 4. Шифрование

    // ✅ Всегда шифруем sensitive данные
    public class PaymentService {
        private final Cipher cipher;
        
        public String encryptCard(String cardNumber) {
            try {
                return Base64.getEncoder().encodeToString(
                    cipher.doFinal(cardNumber.getBytes())
                );
            } catch (Exception e) {
                throw new SecurityException("Encryption failed", e);
            }
        }
    }

### 5. Логирование

    // ✅ Логируем безопасно
    @Slf4j
    public class AuthService {
        public void login(String username, String password) {
            try {
                authenticate(username, password);
                log.info("User logged in: {}", username);  // ✅ Безопасно
            } catch (AuthenticationException e) {
                log.warn("Failed login attempt: {}", username);  // ✅ Безопасно
                // ❌ НЕЛЬЗЯ: log.warn("Password attempted: {}", password);
            }
        }
    }

Что логировать | Что НЕ логировать
---|---
Входы/выходы | Пароли
Ошибки (без деталей) | Токены
Изменения прав | Ключи шифрования
Доступ к ресурсам | Номера карт
Неудачные попытки | Персональные данные

### 6. Обработка ошибок

    // ✅ Не раскрываем внутреннюю информацию
    @RestControllerAdvice
    public class GlobalExceptionHandler {
        
        @ExceptionHandler(Exception.class)
        public ResponseEntity<ErrorResponse> handleException(Exception e) {
            // Логируем полную информацию
            log.error("Internal error", e);
            
            // Пользователю — общее сообщение
            return ResponseEntity.status(500)
                .body(new ErrorResponse("Internal server error", "ERR-001"));
        }
        
        @ExceptionHandler(AccessDeniedException.class)
        public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException e) {
            return ResponseEntity.status(403)
                .body(new ErrorResponse("Access denied", "ERR-403"));
        }
    }

### 7. Зависимости

    // ✅ Проверяем зависимости на уязвимости
    // OWASP Dependency Check
    // Snyk
    // Dependabot
    
    // В pom.xml
    <plugin>
        <groupId>org.owasp</groupId>
        <artifactId>dependency-check-maven</artifactId>
        <version>8.4.0</version>
    </plugin>

### 8. Конфигурация

    # ✅ Не используем дефолтные значения
    # application.properties
    
    # Не показывать stack trace
    server.error.include-stacktrace=never
    server.error.include-message=never
    
    # Отключить debug
    debug=false
    
    # Не хранить секреты в коде
    # Использовать переменные окружения или vault
    spring.datasource.password=${DB_PASSWORD}

## Часть 4: Чек-лист безопасности кода

| Проверка | Да/Нет |
|----------|--------|
| Все входные данные валидируются? | ⬜ |
| Используются PreparedStatement / ORM? | ⬜ |
| Output encoding при выводе пользовательских данных? | ⬜ |
| Аутентификация на всех endpoints? | ⬜ |
| Авторизация на каждом запросе? | ⬜ |
| HTTPS для всех соединений? | ⬜ |
| CSRF-токены на формах? | ⬜ |
| HttpOnly, Secure, SameSite на cookie? | ⬜ |
| BCrypt / Argon2 для паролей? | ⬜ |
| Логирование без секретов? | ⬜ |
| Обработка ошибок без раскрытия информации? | ⬜ |
| Регулярное обновление зависимостей? | ⬜ |
| Секреты не в коде? | ⬜ |
| Security headers настроены? | ⬜ |
| Файлы проверяются на тип и размер? | ⬜ |

## Инструменты

| Инструмент | Что делает | Когда использовать |
|------------|------------|-------------------|
| SonarQube | Статический анализ | В CI/CD pipeline |
| SpotBugs + FindSecBugs | Поиск уязвимостей | При сборке |
| OWASP Dependency Check | Проверка зависимостей | Ежедневно |
| Snyk | Сканирование зависимостей | В CI/CD |
| Checkmarx | SAST (Static Application Security Testing) | Enterprise |
| Burp Suite | DAST (Dynamic Testing) | Перед релизом |
| OWASP ZAP | Автоматический сканер | В CI/CD |

### Чек-лист комплаенса

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Защита данных карт | Шифрование, аудит, минимальные привилегии |
| SOX | Контроль доступа к финансовым системам | RBAC, аудит |
| GDPR | Защита персональных данных | Шифрование, валидация, аудит |
| HIPAA | Защита PHI | Шифрование, контроль доступа |
| ISO 27001 | Управление безопасностью | Политики, процедуры, аудит |
| OWASP ASVS | Требования к безопасности | Чек-лист по уровням |

## Вывод

Безопасное программирование — это мышление:
1. **Defense in depth** — многослойная защита
2. **Fail secure** — безопасный отказ
3. **Least privilege** — минимум прав
4. **Zero trust** — не доверяй никому
5. **Валидация** — проверяй всё
6. **Кодирование** — защищай вывод
7. **Логирование** — но не секреты
8. **Обработка ошибок** — без информационных утечек
9. **Зависимости** — проверяй и обновляй
10. **Конфигурация** — безопасная по умолчанию

Правило: **пиши код так, будто его будет атаковать профессионал.**

---
_Статья создана на основе анализа материалов Habr по безопасному программированию_