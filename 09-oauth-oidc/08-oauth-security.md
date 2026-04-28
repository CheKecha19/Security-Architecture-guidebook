# Безопасность OAuth 2.0

## Основные угрозы

### CSRF на Authorization Endpoint

| Атака | Защита |
|-------|--------|
| Поддельный запрос | Параметр state |
| Повтор запроса | state — случайная строка |

### Redirect URI Manipulation

| Атака | Защита |
|-------|--------|
| Отправка токена на другой сайт | Регистрация redirect_uri |
| Открытый redirect | Точное сравнение URI |

### Token Leakage

| Атака | Защита |
|-------|--------|
| Утечка в логах | Не логировать токены |
| Утечка в URL | POST вместо GET |
| Утечка в истории | Не хранить в URL |

### Scope Escalation

| Атака | Защита |
|-------|--------|
| Запрос большего scope | Проверка на сервере |
| Игнорирование отказа | Обработка ошибок |

## Защита Authorization Code

| Мера | Описание |
|------|----------|
| Короткий срок | 10 минут |
| Одноразовый | Использован — сгорел |
| PKCE | Обязательно для public clients |
| client_secret | Для confidential clients |

## Защита Tokens

| Мера | Описание |
|------|----------|
| HTTPS | Всегда |
| Короткий срок | Access: 15-60 минут |
| Refresh rotation | Новый refresh при каждом использовании |
| Revocation | Возможность отзыва |
| Binding | Привязка к устройству |

## Security Headers

| Header | Описание |
|--------|----------|
| Strict-Transport-Security | HTTPS only |
| X-Frame-Options | Защита clickjacking |
| Content-Security-Policy | Защита XSS |

---
_Статья создана на основе анализа материалов Habr_
