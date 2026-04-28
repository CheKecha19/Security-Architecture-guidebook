# Spring Security: Фреймворк безопасности

## Что такое Spring Security?

Представь клуб. В клубе есть вход с охраной. Охрана проверяет: есть ли у тебя приглашение? Какой уровень доступа? Ты — VIP или обычный гость? В какой зал можно? В бар? В танцпол? В закрытую комнату?

**Spring Security** — это именно такая "охрана" для Java-приложений. Проверяет: кто ты (аутентификация), куда можешь (авторизация), что тебе доступно (контроль доступа).

Spring Security — это фреймворк безопасности для Java-приложений. Работает с Spring Framework, Spring Boot. Защищает от угроз: SQL injection, XSS, CSRF, session fixation.

## Зачем нужен Spring Security?

### Ситуация 1: Защита API

HR-платформа предоставляет API для партнёров. Нужно проверять: кто вызывает, имеет ли право.

Spring Security: JWT-токен → проверка подписи → определение роли → разрешение или запрет.

### Ситуация 2: Форма входа

Пользователь вводит логин и пароль. Нужно проверить, создать сессию, защитить от CSRF.

Spring Security: форма → проверка → сессия → CSRF-токен.

### Ситуация 3: Роли

HR-менеджер может видеть все резюме. Кандидат — только своё. Админ — всё.

Spring Security: роли, права на методы, аннотации.

## Архитектура Spring Security

### Основные компоненты

| Компонент | Описание | Аналогия |
|-----------|----------|----------|
| SecurityContext | Контекст безопасности | Бейдж на входе |
| Authentication | Аутентификация | Проверка приглашения |
| UserDetails | Информация о пользователе | Профиль гостя |
| UserDetailsService | Загрузка пользователя | База гостей |
| GrantedAuthority | Права | Уровень доступа |
| FilterChain | Цепочка фильтров | Посты охраны |
| SecurityFilterChain | Настройка фильтров | Маршрут охраны |

### Аутентификация

| Способ | Описание | Когда |
|--------|----------|-------|
| Form Login | Форма логин/пароль | Веб-приложения |
| HTTP Basic | Заголовок Authorization | API |
| Digest | Хешированный пароль | Устарел |
| OAuth2 | Токен от провайдера | SSO |
| SAML | XML-токен | Enterprise |
| JWT | JSON Web Token | Stateless API |
| LDAP | Active Directory | Корпорации |
| Kerberos | Сетевой протокол | Windows |

### Авторизация

| Тип | Описание | Пример |
|-----|----------|--------|
| URL-based | По URL | /admin/** → ADMIN |
| Method-based | По методу | @PreAuthorize |
| Domain-based | По данным | @PostFilter |

### Аннотации

| Аннотация | Описание |
|-----------|----------|
| @PreAuthorize | Перед вызовом |
| @PostAuthorize | После вызова |
| @Secured | Роли |
| @RolesAllowed | Роли (JSR-250) |
| @PreFilter | Фильтр входных данных |
| @PostFilter | Фильтр выходных данных |

## Защита от угроз

### CSRF

| Что делает | Как |
|------------|-----|
| Генерирует токен | Для каждой сессии |
| Проверяет | В POST/PUT/DELETE |
| Исключения | API с JWT |

### XSS

| Что делает | Как |
|------------|-----|
| Content-Type | Правильный |
| CSP | Content Security Policy |
| Headers | X-Content-Type-Options |

### Session Fixation

| Что делает | Как |
|------------|-----|
| Меняет ID | После аутентификации |
| Ограничивает | Один пользователь — одна сессия |

### Clickjacking

| Что делает | Как |
|------------|-----|
| X-Frame-Options | DENY или SAMEORIGIN |

## Конфигурация

### Spring Boot 3 (Java 17+)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http
            .csrf(csrf -> csrf.disable()) // для API
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### OAuth2 Resource Server

```java
http
    .oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.jwtAuthenticationConverter(converter))
    );
```

## Лучшие практики

| Практика | Описание |
|----------|----------|
| HTTPS | Всегда |
| CSRF | Включить для форм |
| CSP | Content Security Policy |
| HSTS | Strict-Transport-Security |
| Password encoding | BCrypt, Argon2 |
| Session management | Минимум, timeout |
| JWT | Короткий срок, refresh |
| Audit | Логирование |

## Чек-лист понимания

- [ ] Что такое Spring Security?
- [ ] Какие компоненты?
- [ ] Какие способы аутентификации?
- [ ] Какие аннотации для авторизации?
- [ ] Как защита от CSRF?
- [ ] Как защита от XSS?
- [ ] Как настроить OAuth2?
- [ ] Какие лучшие практики?
- [ ] Что такое SecurityFilterChain?
- [ ] Как Spring Boot 3 отличается?

### Ответы на чек-лист

1. **Spring Security** — фреймворк безопасности для Java. Аутентификация, авторизация, защита от угроз.

2. **Компоненты**: SecurityContext, Authentication, UserDetails, UserDetailsService, GrantedAuthority, FilterChain.

3. **Аутентификация**: Form Login, HTTP Basic, OAuth2, SAML, JWT, LDAP, Kerberos.

4. **Аннотации**: @PreAuthorize, @PostAuthorize, @Secured, @RolesAllowed, @PreFilter, @PostFilter.

5. **CSRF**: генерирует токен, проверяет в POST/PUT/DELETE.

6. **XSS**: Content-Type, CSP, X-Content-Type-Options.

7. **OAuth2**: .oauth2ResourceServer(oauth2 -> oauth2.jwt(...)).

8. **Лучшие практики**: HTTPS, CSRF, CSP, HSTS, BCrypt, минимум сессии, короткий JWT, audit.

9. **SecurityFilterChain** — цепочка фильтров безопасности. Заменяет WebSecurityConfigurerAdapter.

10. **Spring Boot 3**: Java 17+, SecurityFilterChain вместо WebSecurityConfigurerAdapter, lambda DSL.

---

_Статья создана на основе анализа материалов Habr по Spring Security_
