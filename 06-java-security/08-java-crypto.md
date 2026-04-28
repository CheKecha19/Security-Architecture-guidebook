# Криптография в Java

## Что такое криптография?

Представь, что ты отправляешь письмо. Не электронное, а настоящее, бумажное. Ты кладешь его в конверт. Но конверт — прозрачный. Все видят, что внутри.

Теперь представь, что ты кладешь письмо в сейф. Сейф запираешь. Отправляешь. Получатель имеет ключ. Открывает. Читает. Никто по пути не видел содержимого.

**Криптография** — это именно такой "сейф" для данных. Превращает понятную информацию в непонятный набор символов. Только у кого есть ключ — может прочитать.

## Зачем нужна криптография?

### Ситуация 1: Пароли

Пользователь ввёл пароль. Если хранить как текст — утечка = все пароли раскрыты. Храним хеш — утечка = ничего страшного.

### Ситуация 2: Данные

База данных содержит персональные данные. Если диск украдут — данные зашифрованы.

### Ситуация 3: Связь

Два сервера общаются. Если трафик перехватят — ничего не поймут.

## Типы криптографии

### Хеширование

| Алгоритм | Описание | Безопасность |
|----------|----------|--------------|
| MD5 | 128 бит | Небезопасен |
| SHA-1 | 160 бит | Небезопасен |
| SHA-256 | 256 бит | Безопасен |
| SHA-3 | Переменная | Безопасен |

```java
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] hash = digest.digest(password.getBytes(StandardCharsets.UTF_8));
```

### Симметричное шифрование

| Алгоритм | Ключ | Описание |
|----------|------|----------|
| AES | 128/256 бит | Стандарт |
| ChaCha20 | 256 бит | Современный |
| 3DES | 168 бит | Устарел |

```java
SecretKey key = KeyGenerator.getInstance("AES").generateKey();
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, key);
byte[] encrypted = cipher.doFinal(data.getBytes());
```

### Асимметричное шифрование

| Алгоритм | Ключи | Описание |
|----------|-------|----------|
| RSA | 2048+ бит | Универсальный |
| ECC | 256+ бит | Современный |
| DH | — | Обмен ключами |

```java
KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
keyGen.initialize(2048);
KeyPair pair = keyGen.generateKeyPair();
```

### Цифровая подпись

| Алгоритм | Описание |
|----------|----------|
| RSA + SHA-256 | Стандарт |
| ECDSA | ECC-based |
| Ed25519 | Современный |

```java
Signature signature = Signature.getInstance("SHA256withRSA");
signature.initSign(privateKey);
signature.update(data.getBytes());
byte[] sig = signature.sign();
```

## JCA и JCE

### Java Cryptography Architecture (JCA)

| Компонент | Описание |
|-----------|----------|
| Provider | Реализация алгоритмов |
| Engine | Интерфейс (Cipher, Signature) |
| SPI | Service Provider Interface |

### Java Cryptography Extension (JCE)

| Ограничение | Решение |
|-------------|---------|
| 128-bit key | Unlimited Strength Jurisdiction Policy |
| Алгоритмы | BouncyCastle |

### BouncyCastle

```java
Security.addProvider(new BouncyCastleProvider());
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding", "BC");
```

## Хеширование паролей

### Почему не SHA?

SHA быстрый. Для паролей — плохо. Можно перебирать миллионы в секунду.

### BCrypt

| Параметр | Описание |
|----------|----------|
| Salt | Автоматический |
| Cost | Сложность (10-12) |
| Output | 60 символов |

```java
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(password);
boolean matches = encoder.matches(password, hash);
```

### Argon2

| Параметр | Описание |
|----------|----------|
| Memory | Использование памяти |
| Iterations | Итерации |
| Parallelism | Параллелизм |

```java
Argon2PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
```

### SCrypt

| Параметр | Описание |
|----------|----------|
| CPU | Использование CPU |
| Memory | Использование памяти |

## TLS/SSL

### Версии

| Версия | Безопасность |
|--------|--------------|
| SSL 2.0 | Небезопасен |
| SSL 3.0 | Небезопасен |
| TLS 1.0 | Устарел |
| TLS 1.1 | Устарел |
| TLS 1.2 | Минимум |
| TLS 1.3 | Рекомендуется |

### Конфигурация

```java
SSLContext context = SSLContext.getInstance("TLSv1.3");
context.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), null);
```

### Cipher Suites

| Suite | Описание |
|-------|----------|
| TLS_AES_256_GCM_SHA384 | TLS 1.3 |
| TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 | TLS 1.2 |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Не изобретай | Используй стандартные библиотеки |
| BCrypt/Argon2 | Для паролей |
| AES-256-GCM | Для данных |
| RSA-2048+ | Для асимметрии |
| TLS 1.3 | Для связи |
| Случайность | SecureRandom |
| Не храни ключи | KMS, HSM |
| Обновляй | Алгоритмы, библиотеки |

## Чек-лист понимания

- [ ] Что такое криптография?
- [ ] Какие типы?
- [ ] Чем хеширование отличается от шифрования?
- [ ] Что такое JCA?
- [ ] Что такое JCE?
- [ ] Почему BCrypt, а не SHA?
- [ ] Какие параметры BCrypt?
- [ ] Что такое TLS?
- [ ] Какие версии TLS?
- [ ] Какие лучшие практики?

### Ответы на чек-лист

1. **Криптография** — наука о защите информации. Хеширование, шифрование, подпись.

2. **Типы**: хеширование (SHA), симметричное (AES), асимметричное (RSA), подпись (ECDSA).

3. **Хеширование** — одностороннее. Нельзя восстановить. **Шифрование** — двустороннее. Можно расшифровать.

4. **JCA** — Java Cryptography Architecture. Архитектура криптографии в Java.

5. **JCE** — Java Cryptography Extension. Расширение с дополнительными алгоритмами.

6. **BCrypt**, потому что медленный. SHA быстрый — можно перебирать.

7. **Параметры BCrypt**: salt (автоматический), cost (10-12), output (60 символов).

8. **TLS** — Transport Layer Security. Протокол защиты связи.

9. **Версии TLS**: 1.0 (устарел), 1.1 (устарел), 1.2 (минимум), 1.3 (рекомендуется).

10. **Лучшие практики**: не изобретай, BCrypt/Argon2, AES-256-GCM, RSA-2048+, TLS 1.3, SecureRandom, KMS, обновлять.

---

_Статья создана на основе анализа материалов Habr по криптографии в Java_
