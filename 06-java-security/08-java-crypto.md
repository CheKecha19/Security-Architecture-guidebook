# Криптография в Java

## Часть 1: Что такое криптография?

Представь, что ты отправляешь письмо. Не электронное, а настоящее, бумажное. Ты кладешь его в конверт. Но конверт — прозрачный. Все видят, что внутри.

Теперь представь, что ты кладешь письмо в сейф. Сейф запираешь. Отправляешь. Получатель имеет ключ. Открывает. Читает. Никто по пути не видел содержимого.

**Криптография** — это именно такой «сейф» для данных. Превращает понятную информацию в непонятный набор символов. Только у кого есть ключ — может прочитать.

## Зачем нужна криптография?

### Ситуация 1: Пароли

Пользователь ввёл пароль. Если хранить как текст (`password123`) — утечка базы = все пароли раскрыты. Храним хеш (`$2a$12$...`) — утечка = ничего страшного. Хеш нельзя «расшифровать».

### Ситуация 2: Данные

База данных содержит персональные данные: паспорта, адреса. Если диск украдут — данные зашифрованы. Без ключа — бесполезны.

### Ситуация 3: Связь

Два сервера общаются по интернету. Если трафик перехватят — ничего не поймут. TLS шифрует всё.

## Часть 2: Типы криптографии

### Хеширование

**Хеширование** — односторонняя функция. Из данных получается фиксированная строка. Нельзя восстановить данные из хеша.

| Алгоритм | Длина | Статус |
|----------|-------|--------|
| MD5 | 128 бит | ❌ Устарел, коллизии |
| SHA-1 | 160 бит | ❌ Устарел, коллизии |
| SHA-256 | 256 бит | ✅ Безопасен |
| SHA-3 | 256-512 бит | ✅ Безопасен |
| BLAKE2 | 256-512 бит | ✅ Быстрый, безопасен |

    // SHA-256
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    byte[] hash = digest.digest("password".getBytes(StandardCharsets.UTF_8));
    String hex = Base64.getEncoder().encodeToString(hash);

**Пароли НЕ хешируются SHA-256!** Для паролей — специальные алгоритмы (см. ниже).

### Симметричное шифрование

Один ключ для шифрования и дешифрования.

| Алгоритм | Ключ | Режим | Статус |
|----------|------|-------|--------|
| AES | 128/256 бит | GCM, CBC | ✅ Стандарт |
| ChaCha20 | 256 бит | Poly1305 | ✅ Современный |
| 3DES | 168 бит | CBC | ❌ Устарел |

    // AES-256-GCM
    SecretKey key = KeyGenerator.getInstance("AES").generateKey();
    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    cipher.init(Cipher.ENCRYPT_MODE, key);
    byte[] encrypted = cipher.doFinal("secret".getBytes());
    
    byte[] iv = cipher.getIV();  // Сохранить для дешифрования

### Асимметричное шифрование

Два ключа: публичный (шифрует) и приватный (дешифрует).

| Алгоритм | Ключ | Использование |
|----------|------|---------------|
| RSA | 2048-4096 бит | Универсальный |
| ECC (ECDSA) | 256-521 бит | Современный, быстрый |
| DH | — | Обмен ключами |

    // RSA
    KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
    keyGen.initialize(2048);
    KeyPair pair = keyGen.generateKeyPair();
    
    // Шифрование публичным ключом
    Cipher cipher = Cipher.getInstance("RSA");
    cipher.init(Cipher.ENCRYPT_MODE, pair.getPublic());
    byte[] encrypted = cipher.doFinal("secret".getBytes());

### Цифровая подпись

Доказывает: данные не изменены, отправитель — именно он.

| Алгоритм | Описание |
|----------|----------|
| RSA + SHA-256 | Стандарт |
| ECDSA | ECC-based, быстрый |
| Ed25519 | Современный, безопасный |

    // Подпись
    Signature signature = Signature.getInstance("SHA256withRSA");
    signature.initSign(privateKey);
    signature.update(data.getBytes());
    byte[] sig = signature.sign();
    
    // Проверка
    Signature verifier = Signature.getInstance("SHA256withRSA");
    verifier.initVerify(publicKey);
    verifier.update(data.getBytes());
    boolean valid = verifier.verify(sig);

## Часть 3: JCA и JCE

### Java Cryptography Architecture (JCA)

Архитектура криптографии в Java:

| Компонент | Описание |
|-----------|----------|
| Provider | Реализация алгоритмов (Sun, BouncyCastle) |
| Engine | Интерфейс: Cipher, Signature, MessageDigest |
| SPI | Service Provider Interface — для создания провайдеров |

### Java Cryptography Extension (JCE)

Расширение с дополнительными алгоритмами.

**История проблемы:** До Java 9 был «Export Policy» — максимум 128-битные ключи. Нужно было скачивать «Unlimited Strength Jurisdiction Policy Files». С Java 9 — unlimited по умолчанию.

### BouncyCastle

Популярный сторонний провайдер:

    // Добавить провайдер
    Security.addProvider(new BouncyCastleProvider());
    
    // Использовать
    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding", "BC");

## Часть 4: Хеширование паролей

### Почему не SHA-256 для паролей?

SHA-256 **быстрый**. Для паролей это плохо: можно перебирать миллиарды паролей в секунду на GPU.

Для паролей нужны **медленные** алгоритмы: BCrypt, Argon2, SCrypt.

### BCrypt

    // Spring Security
    BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
    String hash = encoder.encode("password");
    boolean matches = encoder.matches("password", hash);
    
    // Пример хеша: $2a$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
    // $2a$ — версия
    // 12$ — cost factor (2^12 итераций)

| Параметр | Описание | Рекомендация |
|----------|----------|--------------|
| Cost | Сложность (логарифм итераций) | 12-14 |
| Salt | Автоматический | Генерируется BCrypt |
| Output | 60 символов | Хранить целиком |

### Argon2

Победитель Password Hashing Competition (2015):

    Argon2PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
    String hash = encoder.encode("password");

| Параметр | Описание |
|----------|----------|
| Memory | Использование памяти (KB) |
| Iterations | Итерации |
| Parallelism | Параллелизм |

### Сравнение алгоритмов паролей

| Алгоритм | Медленный | Устойчив к GPU | Устойчив к ASIC | Рекомендация |
|----------|-----------|----------------|-----------------|--------------|
| SHA-256 | ❌ | ❌ | ❌ | ❌ Нет |
| BCrypt | ✅ | ⚠️ | ⚠️ | ✅ Да |
| SCrypt | ✅ | ✅ | ⚠️ | ✅ Да |
| Argon2 | ✅ | ✅ | ✅ | ✅ Лучший |

## Часть 5: TLS/SSL

### Версии TLS

| Версия | Статус | Рекомендация |
|--------|--------|--------------|
| SSL 2.0 | ❌ Устарел | Запретить |
| SSL 3.0 | ❌ Устарел | Запретить |
| TLS 1.0 | ❌ Устарел | Запретить |
| TLS 1.1 | ❌ Устарел | Запретить |
| TLS 1.2 | ✅ Поддерживается | Минимум |
| TLS 1.3 | ✅ Актуальная | Рекомендуется |

    // Настройка SSLContext
    SSLContext context = SSLContext.getInstance("TLSv1.3");
    context.init(keyManagerFactory.getKeyManagers(), 
                 trustManagerFactory.getTrustManagers(), 
                 new SecureRandom());

### Cipher Suites

| Suite | TLS | Описание |
|-------|-----|----------|
| TLS_AES_256_GCM_SHA384 | 1.3 | AES-256-GCM |
| TLS_CHACHA20_POLY1305_SHA256 | 1.3 | ChaCha20 |
| TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 | 1.2 | Forward secrecy |

**Запрещённые:**
- NULL шифры
- EXPORT шифры
- DES, 3DES
- RC4
- MD5
- SHA1

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Не изобретай своё | Используй стандартные библиотеки |
| BCrypt/Argon2 | Для паролей |
| AES-256-GCM | Для шифрования данных |
| RSA-2048+ или ECC-256+ | Для асимметрии |
| TLS 1.3 | Для связи |
| SecureRandom | Для генерации случайных чисел |
| Не храни ключи в коде | KMS, Vault, HSM |
| Ротация ключей | Регулярно |
| Обновляй библиотеки | Новые уязвимости |

### Чек-лист

| Проверка | Статус |
|----------|--------|
| Пароли хешируются BCrypt/Argon2 | ⬜ |
| Шифрование — AES-256-GCM | ⬜ |
| TLS 1.2+ для всех соединений | ⬜ |
| Ключи не в коде | ⬜ |
| SecureRandom вместо Random | ⬜ |
| Регулярная ротация ключей | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| PCI DSS | Шифрование данных карт | AES-256, TLS 1.2+ |
| GDPR | Защита персональных данных | Шифрование, хеширование |
| HIPAA | Шифрование PHI | AES-256, управление ключами |
| NIST | Рекомендации алгоритмов | AES-256, SHA-256, RSA-2048+ |

## Вывод

Криптография в Java:
1. **Хеширование** — SHA-256 для данных, BCrypt/Argon2 для паролей
2. **Симметрия** — AES-256-GCM для шифрования
3. **Асимметрия** — RSA-2048+ или ECC для ключей и подписей
4. **TLS** — для защиты связи
5. **Управление ключами** — KMS, Vault, HSM

Правило: **не изобретай свою криптографию. Используй проверенные алгоритмы и библиотеки.**

---
_Статья создана на основе анализа материалов Habr по криптографии в Java_