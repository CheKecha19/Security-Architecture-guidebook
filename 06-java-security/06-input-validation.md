# Валидация входных данных в Java

## Что такое валидация?

Представь вход в театр. На входе стоит билетёр. Он проверяет: есть ли у тебя билет? На правильный ли спектакль? На правильное ли место? Не просрочен ли билет? Если всё ок — пропускает. Если нет — не пускает.

**Валидация входных данных** — это именно такой "билетёр" для приложения. Каждый запрос, каждый параметр, каждый файл — проверяется. Корректные данные? Проходят. Некорректные — отбрасываются.

Без валидации любой может отправить что угодно. SQL-инъекцию. XSS. Переполнение буфера. Валидация — первая линия обороны.

## Зачем нужна валидация?

### Ситуация 1: SQL Injection

Пользователь ввёл в поле: "'; DROP TABLE users; --"
Без валидации — база данных удалена.

### Ситуация 2: XSS

Пользователь ввёл: "<script>alert('xss')</script>"
Без валидации — скрипт выполнится в браузере других пользователей.

### Ситуация 3: Path Traversal

Пользователь запросил: "../../../etc/passwd"
Без валидации — чтение системных файлов.

## Типы валидации

### Синтаксическая

| Что проверяем | Пример |
|---------------|--------|
| Тип | Это число? |
| Длина | Не длиннее 100 символов |
| Формат | Email, телефон |
| Диапазон | Число 1-100 |
| RegEx | Только буквы и цифры |

### Семантическая

| Что проверяем | Пример |
|---------------|--------|
| Бизнес-логика | Дата рождения в прошлом |
| Уникальность | Email не занят |
| Существование | ID существует |
| Права | Пользователь может это? |

### Whitelist vs Blacklist

| Подход | Описание | Пример |
|--------|----------|--------|
| Whitelist | Разрешить только известное good | Только буквы |
| Blacklist | Запретить известное bad | Запретить <script> |

**Whitelist предпочтительнее.**

## Валидация в Java

### Bean Validation (JSR-380)

```java
public class User {
    @NotNull
    @Size(min=2, max=50)
    private String name;
    
    @Email
    private String email;
    
    @Min(18)
    @Max(120)
    private int age;
}
```

### Spring Validation

```java
@PostMapping("/users")
public ResponseEntity<User> create(
    @Valid @RequestBody User user,
    BindingResult result) {
    if (result.hasErrors()) {
        return ResponseEntity.badRequest().build();
    }
    return ResponseEntity.ok(service.create(user));
}
```

### Custom Validator

```java
public class SafeHtmlValidator implements ConstraintValidator<SafeHtml, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || !value.matches(".*<script.*>.*");
    }
}
```

## Sanitization

### HTML

| Библиотека | Описание |
|------------|----------|
| OWASP Java HTML Sanitizer | Удаляет опасные теги |
| jsoup | Парсинг и очистка HTML |

### SQL

| Метод | Описание |
|-------|----------|
| PreparedStatement | Параметризованные запросы |
| JPA/Hibernate | ORM с параметрами |
| Stored procedures | Хранимые процедуры |

### URL

| Метод | Описание |
|-------|----------|
| URL encoding | Кодирование символов |
| Whitelist хостов | Только разрешённые |
| File validation | Проверка пути |

## Encoding

### Output Encoding

| Контекст | Метод |
|----------|-------|
| HTML | HtmlUtils.htmlEscape() |
| JavaScript | StringEscapeUtils.escapeEcmaScript() |
| CSS | OwaspEncoder.encodeForCSS() |
| URL | URLEncoder.encode() |
| XML | StringEscapeUtils.escapeXml() |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Всегда валидировать | На сервере, не только на клиенте |
| Whitelist | Разрешить известное хорошее |
| Fail fast | Отклонить сразу |
| Не доверять входу | Любой вход — потенциально вредоносный |
| Layered defense | Валидация + параметризация + encoding |
| Log | Но не логировать пароли |

## Чек-лист понимания

- [ ] Что такое валидация?
- [ ] Какие типы валидации?
- [ ] Чем whitelist отличается от blacklist?
- [ ] Что такое Bean Validation?
- [ ] Как защититься от SQL injection?
- [ ] Что такое sanitization?
- [ ] Какие библиотеки для HTML?
- [ ] Что такое output encoding?
- [ ] Какие лучшие практики?
- [ ] Почему валидация на сервере обязательна?

### Ответы на чек-лист

1. **Валидация** — проверка входных данных на корректность.

2. **Типы**: синтаксическая (тип, длина, формат), семантическая (бизнес-логика).

3. **Whitelist** — разрешить известное хорошее. **Blacklist** — запретить известное плохое. Whitelist лучше.

4. **Bean Validation** — JSR-380. Аннотации: @NotNull, @Size, @Email, @Min, @Max.

5. **SQL Injection**: PreparedStatement, ORM, stored procedures.

6. **Sanitization** — очистка данных от опасного содержимого.

7. **HTML**: OWASP Java HTML Sanitizer, jsoup.

8. **Output encoding** — кодирование перед выводом в разные контексты.

9. **Лучшие практики**: всегда валидировать, whitelist, fail fast, не доверять входу, layered defense.

10. **Валидация на сервере обязательна**, потому что клиентскую валидацию можно обойти.

---

_Статья создана на основе анализа материалов Habr по валидации_
