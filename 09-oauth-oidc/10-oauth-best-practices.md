# Лучшие практики OAuth 2.0

## Чек-лист реализации

### Authorization Server

- [ ] HTTPS everywhere
- [ ] PKCE обязателен
- [ ] Точное сравнение redirect_uri
- [ ] Короткий срок кода (10 мин)
- [ ] State обязателен
- [ ] Scope валидация
- [ ] Token revocation
- [ ] Audit logs

### Client

- [ ] HTTPS everywhere
- [ ] PKCE для всех
- [ ] State проверка
- [ ] Токены в памяти
- [ ] Refresh rotation
- [ ] Logout инвалидация
- [ ] Error handling

### Resource Server

- [ ] Token validation
- [ ] Scope проверка
- [ ] HTTPS
- [ ] Rate limiting
- [ ] Audit

## Архитектура

`
Client → HTTPS → Authorization Server
Client → HTTPS → Resource Server
Authorization Server → HTTPS → Resource Server (introspection)
`

## Сравнение потоков

| Поток | Когда | PKCE | Client Secret |
|-------|-------|------|---------------|
| Authorization Code | Веб, мобиль | Обязательно | Confidential |
| Client Credentials | Server-to-server | Нет | Обязательно |
| Device Code | ТВ, IoT | Нет | Нет |
| Implicit (устарел) | — | — | — |
| Password (устарел) | — | — | — |

## Ограничения и нюансы

| Проблема | Решение |
|----------|---------|
| Сложность | Использовать библиотеки |
| Legacy | Не использовать Implicit, Password |
| Token size | Не хранить много данных |
| Clock skew | Проверка exp с запасом |
| Key rotation | JWKS endpoint |

---
_Статья создана на основе анализа материалов Habr_
