# OpenID Connect

## Что такое OpenID Connect?

Представь, что у тебя есть пропуск в клуб. Пропуск даёт доступ. Но на прописи нет имени, фото, возраста. Только "доступ разрешён".

А теперь представь паспорт. В паспорте есть имя, фото, дата рождения, адрес. Паспорт не только даёт доступ, но и доказывает, кто ты.

**OpenID Connect (OIDC)** — это именно такой "паспорт" поверх OAuth. OAuth даёт доступ. OIDC добавляет информацию о пользователе.

## Разница с OAuth

| Критерий | OAuth 2.0 | OpenID Connect |
|----------|-----------|----------------|
| Цель | Авторизация (доступ) | Аутентификация (кто ты) |
| Токен | Access Token | ID Token + Access Token |
| Информация | Нет данных о пользователе | Claims: имя, email, фото |
| Discovery | Нет | .well-known/openid-configuration |
| Logout | Не определён | RP-Initiated Logout |

## ID Token

| Часть | Описание | Пример |
|-------|----------|--------|
| Header | Алгоритм подписи | RS256 |
| Payload | Claims | sub, name, email |
| Signature | Подпись | Проверка целостности |

### Claims

| Claim | Описание |
|-------|----------|
| sub | Subject — уникальный ID |
| name | Имя |
| email | Email |
| picture | Фото |
| iss | Issuer — кто выдал |
| aud | Audience — для кого |
| exp | Expiration — срок действия |
| iat | Issued at — когда выдан |
| nonce | Защита от replay |

## Discovery

`
GET /.well-known/openid-configuration

{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json"
}
`

## UserInfo Endpoint

`
GET /userinfo
Authorization: Bearer access_token

{
  "sub": "12345",
  "name": "Alice",
  "email": "alice@example.com"
}
`

---
_Статья создана на основе анализа материалов Habr_
