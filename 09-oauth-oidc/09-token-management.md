# Управление токенами

## Access Token

| Параметр | Рекомендация |
|----------|--------------|
| Срок жизни | 15-30 минут |
| Формат | JWT |
| Хранение | Память (не localStorage) |
| Передача | Authorization: Bearer |

## Refresh Token

| Параметр | Рекомендация |
|----------|--------------|
| Срок жизни | 7-30 дней |
| Хранение | HttpOnly cookie или secure storage |
| Rotation | Новый refresh при каждом использовании |
| Revocation | Возможность отзыва |

## Revocation

| Метод | Описание |
|-------|----------|
| Token Revocation | RFC 7009 |
| Logout | Инвалидировать токен |
| Session Management | Хранить активные токены |

## Introspection

`
POST /introspect
Authorization: Basic client_id:client_secret
Content-Type: application/x-www-form-urlencoded

token=access_token

{
  "active": true,
  "scope": "read write",
  "client_id": "my-app",
  "username": "alice",
  "exp": 1234567890
}
`

## Best Practices

| Практика | Описание |
|----------|----------|
| Не хранить в localStorage | XSS-риск |
| HttpOnly cookie | Защита от XSS |
| SameSite cookie | Защита CSRF |
| Короткий access | Минимум риска |
| Rotation refresh | Компрометация — минимальный ущерб |
| Мониторинг | Необычное использование |

---
_Статья создана на основе анализа материалов Habr_
