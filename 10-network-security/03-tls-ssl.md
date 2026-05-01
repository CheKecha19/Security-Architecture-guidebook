# Безопасность TLS/SSL: От рукопожатия до нулевого доверия

## Часть 1. Знакомство с проблемой: Почему TLS — это дипломат с проверяемым удостоверением?

Представьте, что вы прибываете в заграничную страницу для переговоров. На границе вас встречает чиновник, который предъявляет удостоверение: "Я — представитель министерства торговли". Вы проверяете печать, сверяете фото, звоните в министерство для подтверждения. Только после этого начинаете переговоры. **TLS** — это именно такой протокол доверия в цифровом мире. Он позволяет двум сторонам, никогда ранее не встречавшимся, установить защищённый канал связи.

**TLS** (Transport Layer Security) работает между прикладным и транспортным уровнем TCP/IP. Он обеспечивает три критических свойства: **конфиденциальность** (шифрование), **целостность** (MAC — Message Authentication Code), и **аутентичность** (сертификаты X.509). Но эта защита не волшебная — она зависит от цепочки доверия, конфигурации, и выбора криптографических примитивов.

В 2026 году **TLS 1.3** (RFC 8446) — текущий стандарт. Он устранил множество устаревших и опасных опций TLS 1.2: MD5, SHA-1, RSA key exchange, CBC-режимы, произвольный порядок обмена сообщениями. **0-RTT** (Zero Round Trip Time) позволяет возобновлять сессии без дополнительного рукопожатия — но за счёт **forward secrecy**. **mTLS** (mutual TLS) требует сертификата от клиента, превращая аутентификацию в двустороннюю.

| Версия TLS | Год | Ключевые изменения | Статус |
|-----------|-----|-------------------|--------|
| SSL 2.0 | 1995 | Первый протокол | Устарел, уязвим |
| SSL 3.0 | 1996 | Улучшенное шифрование | POODLE-атака, запрещён |
| TLS 1.0 | 1999 | Переименование, BEAST | Устарел |
| TLS 1.1 | 2006 | Защита от CBC-атак | Устарел |
| TLS 1.2 | 2008 | SHA-256, AEAD, гибкость | Допустим, но не рекомендуется |
| TLS 1.3 | 2018 | Упрощённый handshake, FS по умолчанию | Рекомендуется |

**Эволюция:** В 1995 **Netscape** создал SSL для защиты кредитных карт. В 2000-х **BEAST** (2011), **CRIME** (2012), **BREACH** (2013), **POODLE** (2014), **FREAK** (2015), **Logjam** (2015), **DROWN** (2016), **SWEET32** (2016) — каждая атака эксплуатировала слабость в cipher suite или режиме. В 2018 **TLS 1.3** вырезал всё опасное. В 2024 **post-quantum cryptography** начала интеграцию в TLS через **hybrid key exchange**.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Электронная коммерция и downgrade-атака
Интернет-магазин поддерживает TLS 1.0 для совместимости с legacy-клиентами. Злоумышленник (MITM) перехватывает ClientHello и заявляет: "Я поддерживаю только TLS 1.0". Сервер соглашается. Затем злоумышленник использует **BEAST**-атаку против CBC-шифра в TLS 1.0, расшифровывая cookies сессии за 1000 запросов. Получив session cookie администратора, он меняет цены на товары. Потери: $2M за неделю.

### Сценарий 2: Микросервисы и отсутствие mTLS
Облачный SaaS-платформа использует Kubernetes с **Service Mesh** (Istio). Разработчик забыл включить **mTLS** между сервисами. Злоумышленник, получив доступ к одному поду через уязвимость в библиотеке, свободно общается с другими сервисами по HTTP. Он вызывает API платёжного сервиса напрямую, обходя API gateway и аутентификацию. Сливает данные 10M пользователей. Урок: внутри периметра тоже нужен TLS.

### Сценарий 3: IoT и certificate pinning
Производитель умных замков внедрил **certificate pinning** в мобильное приложение. Но при обновлении сертификата на сервере забыли обновить pinning hash в приложении. Приложение отказалось подключаться — замки не открывались удалённо. Тысячи клиентов остались без доступа. Компания выпустила срочное обновление, но репутационный ущерб был колоссален. Урок: pinning требует operational discipline.

## Часть 3. Техническая глубина: Как устроен TLS

### TLS Handshake: От ClientHello до Application Data
**Аналогия:** Это как если бы два дипломата, встретившись впервые, обменялись визитками (сертификатами), договорились о языке (cipher suite), и создали секретный код, по которому будут шифровать все последующие переговоры.

**TLS 1.3 handshake** (упрощённый):
1. **ClientHello** — client отправляет supported versions, cipher suites, key shares (public keys для ECDHE)
2. **ServerHello** — server выбирает версию, cipher, отправляет свой key share, encrypted extensions
3. **{Certificate}** — сертификат сервера, подписанный CA
4. **{CertificateVerify}** — подпись transcript handshake'а для доказательства владения приватным ключом
5. **{Finished}** — MAC на весь handshake для защиты от tampering
6. **{Finished}** (client) — аналогично
7. **Application Data** — зашифрованный трафик

Всё после ServerHello зашифровано. **1-RTT** для новых соединений, **0-RTT** для возобновления (PSK — Pre-Shared Key).

| Этап handshake | Что передаётся | Защита | Уязвимость |
|---------------|---------------|--------|-----------|
| ClientHello | Versions, ciphers, key shares | Открытым текстом | Version/cipher downgrade |
| ServerHello | Selected params, key share | Открытым текстом | Key share manipulation |
| Certificate | X.509 chain | Открытым (в TLS 1.2) / Encrypted (TLS 1.3) | Поддельный сертификат |
| CertificateVerify | Signature of transcript | Зашифровано | Если приватный ключ украден |
| Finished | MAC of handshake | Зашифровано | Tampering detection |

### Cipher Suites: От MD5 до AEAD
**Cipher suite** определяет три алгоритма: key exchange, authentication, symmetric encryption + MAC.

**TLS 1.3 cipher suites** (всего 5, всё AEAD):
- **TLS_AES_256_GCM_SHA384** — AES-256 в GCM-режиме, SHA-384 для PRF
- **TLS_CHACHA20_POLY1305_SHA256** — ChaCha20 + Poly1305 MAC, быстрее на ARM/мобильных без AES-NI
- **TLS_AES_128_GCM_SHA256** — AES-128, быстрее, но меньший ключ

**Запрещено в TLS 1.3:** RSA key exchange (нет forward secrecy), CBC-режимы (BEAST, Lucky13), RC4 (статистические атаки), MD5/SHA-1 (коллизии), произвольное сжатие (CRIME, BREACH).

| Алгоритм | Назначение | Статус | Почему |
|----------|-----------|--------|--------|
| RSA key exchange | Передача premaster secret | Запрещён | Нет forward secrecy |
| ECDHE (P-256, P-384, X25519) | Ephemeral key exchange | Рекомендуется | Forward secrecy |
| AES-CBC | Симметричное шифрование | Запрещён | BEAST, Lucky13, padding oracle |
| AES-GCM | AEAD шифрование | Рекомендуется | Аутентифицированное шифрование |
| ChaCha20-Poly1305 | AEAD для мобильных | Рекомендуется | Быстрее без AES-NI, timing-safe |
| SHA-1 | Hash | Запрещён | Коллизии (SHAttered) |
| SHA-256/384 | Hash | Рекомендуется | Устойчив к коллизиям |

### Certificate Pinning и его компромиссы
**Certificate pinning** — жёсткая привязка приложения к конкретному сертификату или его hash. Защищает от компрометации CA (когда злоумышленник получает валидный сертификат для вашего домена от скомпрометированного или злонамеренного CA).

**Проблемы pinning'а:**
1. **Deployment complexity** — нужно встраивать hash в приложение при каждом релизе
2. **Certificate rotation** — при обновлении сертификата приложение ломается
3. **Backup pins** — нужны fallback hash'ы, но они усложняют логику
4. **Revocation** — если приватный ключ скомпрометирован, pinning не помогает

**Modern alternative:** **Expect-CT** (Certificate Transparency enforcement) требует, чтобы сертификаты были залогированы в публичных CT-логах. **CAA records** (RFC 6844) ограничивают, какие CA могут выдавать сертификаты для домена.

### mTLS: Когда клиент тоже доказывает личность
**mTLS** (mutual TLS) — сервер требует client certificate. Используется в:
- **B2B API** — партнёры аутентифицируются по сертификатам
- **Service Mesh** — Istio, Linkerd используют mTLS между подами
- **IoT** — устройства идентифицируются по embedded certificates

**Проблемы mTLS:**
1. **Certificate management at scale** — тысячи клиентских сертификатов требуют PKI
2. **Revocation** — CRL и OCSP не масштабируются; OCSP stapling помогает только для серверов
3. **Client certificate privacy** — в TLS 1.2 client certificate передаётся открытым текстом (until encrypted handshake in TLS 1.3)
4. **UX** — пользователи не понимают, как выбрать сертификат в браузере

## Часть 4. Архитектура защиты: Defense in Depth для TLS

### Многоуровневая защита TLS

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| Приложение | Certificate pinning | RFC 7469 (deprecated) | OkHttp, AFNetworking |
| Приложение | Expect-CT | RFC 6962 | HTTP header |
| Транспорт | TLS 1.3 only | RFC 8446 | OpenSSL 1.1.1+, BoringSSL |
| Транспорт | mTLS | RFC 8446 | Istio, Linkerd, nginx |
| Сеть | CAA records | RFC 6844 | DNS-записи |
| Сеть | Certificate Transparency | RFC 6962 | CT-logs (Google, Cloudflare) |
| PKI | OCSP Must-Staple | RFC 7633 | X.509 extension |
| PKI | Short-lived certificates | CAB Forum | 90 days (Let's Encrypt) |

### Мониторинг и метрики
**Ключевые метрики безопасности TLS:**
- **TLS version distribution**: % TLS 1.3 vs 1.2 vs старше
- **Cipher suite usage**: Нет ли запрещённых (RSA, CBC, RC4)
- **Certificate expiry**: Дней до expiration для каждого сертификата
- **CT log coverage**: Все ли сертификаты залогированы
- **mTLS enforcement rate**: % сервисов с обязательным client cert
- **Handshake failures**: Причины — expired cert, untrusted CA, wrong hostname

### Реальные инциденты

**Инцидент 1: DigiNotar (2011)**
Голландский CA **DigiNotar** был скомпрометирован. Злоумышленник выдал себе валидные сертификаты для `*.google.com`, `*.microsoft.com`, `*.facebook.com`, `*.cia.gov`. Сертификаты использовались для MITM-атак в Иране против 300K пользователей. **Урок:** Компрометация CA = глобальная катастрофа. Защита: Certificate Transparency (все сертификаты видны в логах), HPKP (deprecated, но CAA + Expect-CT заменяют), short-lived certificates (меньше окно атаки).

**Инцидент 2: Heartbleed (2014)**
Уязвимость **CVE-2014-0160** в OpenSSL позволяла читать память сервера через malformed heartbeat request. В памяти могли быть: приватные ключи TLS, cookies, пароли, данные пользователей. 17% серверов интернета были уязвимы. **Урок:** Реализация протокола может быть уязвима независимо от протокола. Защита: memory-safe languages (Rust для TLS-реализаций — rustls), sandboxing, regular security audits, rapid patching.

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"TLS шифрует всё, значит я в безопасности"** — TLS защищает в пути (in transit), но не защищает: от XSS (на стороне клиента), от SQL injection (на сервере), от фишинга (пользователь сам введёт данные на фейковый сайт с валидным TLS). TLS ≠ безопасность приложения.
2. **"TLS 1.2 достаточно, не нужно спешить с 1.3"** — TLS 1.2 имеет 37 cipher suites, многие уязвимы. BEAST, CRIME, BREACH, POODLE, Lucky13 — всё атаки на TLS 1.2. TLS 1.3 вырезал всё опасное. Поддержка TLS 1.2 — technical debt.
3. **"mTLS — это silver bullet для API security"** — mTLS аутентифицирует, но не авторизует. Client cert говорит "кто ты", но не "что тебе разрешено". Нужен дополнительный authorization layer. Кроме того, управление client certs в масштабе — PKI-ад.

### Три ситуационных вопроса для Senior+

**Q1:** Как защититься от компрометации CA в мире, где Certificate Transparency уже работает?
**A:** Defense in depth: (1) **CAA records** — ограничиваем, какие CA могут выдавать сертификаты для нашего домена; (2) **Expect-CT** — браузер требует CT-logging; (3) **Short-lived certificates** (90 дней Let's Encrypt) — уменьшаем окно атаки; (4) **Application-level pinning** (для mobile) — хотя deprecated, для high-risk приложений может быть оправдан; (5) **DANE** (RFC 6698) — DNSSEC-подписанные TLSA-записи с hash сертификата. Если CA скомпрометирован, но сертификат не соответствует TLSA-записи — клиент откажется.

**Q2:** Почему 0-RTT в TLS 1.3 считается менее безопасным, чем 1-RTT?
**A:** **0-RTT** использует **PSK** (Pre-Shared Key) из предыдущей сессии. Это позволяет отправлять данные сразу, без round trip. Но: (1) **No forward secrecy for 0-RTT data** — если PSK скомпрометирован, все 0-RTT сообщения расшифровываются; (2) **Replay attacks** — 0-RTT данные могут быть повторно отправлены злоумышленником. Защита: использовать 0-RTT только для idempotent операций (GET без side effects), включить **early data** ограничения.

**Q3:** Как обнаружить, что сервер всё ещё поддерживает уязвимые cipher suites?
**A:** **Active scanning:** `nmap --script ssl-enum-ciphers` или SSL Labs scan. **Passive monitoring:** TLS handshake inspection через IDS/IPS (Zeek, Suricata). **Configuration audit:** Chef/Puppet/Ansible проверяют конфигурацию web-серверов. **CI/CD gates:** Запрещают деплой, если сканирование обнаруживает weak ciphers. **Certificate/Cipher monitoring:** Censys, Shodan для external visibility.

### Чек-лист защиты TLS (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | TLS 1.3 enforced, старые версии отключены? | `openssl s_client -tls1_3` | Обязательно |
| 2 | Нет RSA key exchange, только ECDHE? | `nmap --script ssl-enum-ciphers` | Обязательно |
| 3 | AES-GCM или ChaCha20-Poly1305, нет CBC/RC4? | SSL Labs scan = A+ | Обязательно |
| 4 | Certificate Transparency logging enabled? | Censys.io поиск по домену | Обязательно |
| 5 | CAA records настроены для всех доменов? | `dig CAA example.com` | Высокая |
| 6 | mTLS между микросервисами (service mesh)? | Istio/Linkerd конфигурация | Высокая |
| 7 | OCSP Must-Staple на сертификатах? | `openssl x509 -in cert.pem -text` | Высокая |
| 8 | Certificate expiry monitoring (<30 days alert)? | Prometheus + Alertmanager | Обязательно |
| 9 | HSTS preload + includeSubDomains? | `curl -I https://example.com` | Высокая |
| 10 | DANE/TLSA records для критических сервисов? | `dig TLSA _443._tcp.example.com` | Средняя |

### Key Takeaways
1. **TLS защищает в пути, не от всего** — XSS, SQLi, фишинг работают поверх TLS.
2. **TLS 1.3 — минимальный стандарт** — TLS 1.2 имеет 20+ лет уязвимостей.
3. **CA — single point of failure** — DigiNotar показал, что компрометация CA = глобальный MITM. CT + CAA + short-lived certs = defense in depth.
4. **mTLS ≠ authorization** — аутентификация ≠ разрешение. Нужен дополнительный authorization layer.
5. **0-RTT trade-off** — скорость vs forward secrecy. Использовать осторожно, только для idempotent операций.
