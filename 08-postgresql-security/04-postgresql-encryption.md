# Шифрование в PostgreSQL

## SSL

`
ssl = on
ssl_min_protocol_version = 'TLSv1.2'
ssl_max_protocol_version = 'TLSv1.3'
`

## TDE

| Способ | Описание |
|--------|----------|
| pgcrypto | Расширение |
| OS-level | LUKS, dm-crypt |
| Cloud | EBS encryption |

### pgcrypto

`sql
CREATE EXTENSION pgcrypto;

-- Шифрование
SELECT pgp_sym_encrypt('secret', 'key');

-- Дешифрование
SELECT pgp_sym_decrypt(encrypted, 'key');
`

### Column encryption

`sql
CREATE TABLE users (
    id serial PRIMARY KEY,
    email text,
    ssn bytea -- encrypted
);
`

## Лучшие практики

| Практика | Описание |
|----------|----------|
| TLS 1.3 | Всегда |
| pgcrypto | Для sensitive columns |
| OS-level | Для всего диска |
| Ключи | KMS, Vault |

---
_Статья создана на основе анализа материалов Habr по PostgreSQL_
