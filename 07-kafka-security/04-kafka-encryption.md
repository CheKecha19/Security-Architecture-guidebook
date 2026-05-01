# Шифрование в Kafka: защита данных в пути и в покое

## Что такое шифрование в Kafka?

Представь, что ты отправляешь важное письмо. Обычный конверт — почтальон может прочитать. Сейф — никто не прочитает по пути. Получатель имеет ключ — открывает.

**Шифрование в Kafka** — это «сейф» для данных в распределённой системе. Защищает три состояния данных:
- **In-transit** — при передаче между продюсером и брокером, между брокерами, между брокером и консьюмером
- **At-rest** — при хранении на диске брокеров
- **In-use** — при обработке (менее применимо к Kafka как системе хранения, но важно для приложений)

Без шифрования данные в Kafka — открытый текст. Любой, кто имеет доступ к сети или диску, может прочитать всё.

## Эволюция и мотивация

Ранние версии Kafka (0.8-0.9) не имели встроенного шифрования. Все данные передавались открытым текстом. В корпоративных сетях это считалось приемлемым: «у нас firewall, зачем шифровать внутри сети?»

Три события изменили эту парадигму. Во-первых, **взлом Target в 2013 году**: злоумышленники проникли в корпоративную сеть через HVAC-систему и перехватили трафик между серверами. Данные в пути оказались уязвимыми. Во-вторых, **Snowden revelations (2013)**: документы показали, что перехват внутреннего трафика — стандартная практика. В-третьих, **регуляторное давление**: GDPR требует «appropriate safeguards» для передачи данных, PCI DSS требует шифрования данных карт при передаче.

Kafka 0.9 (2015) ввела поддержку SSL/TLS для шифрования in-transit. Версия 0.10 добавила SASL_SSL — комбинацию аутентификации и шифрования. Версия 2.0 улучшила производительность TLS. Современная Kafka поддерживает TLS 1.2 и 1.3, конфигурируемые cipher suites, и возможность ограничения протоколов.

Для at-rest шифрования Kafka не имеет встроенного механизма (в отличие от PostgreSQL с pgcrypto). Шифрование диска — задача операционной системы или облачного провайдера. Это архитектурное решение: Kafka фокусируется на потоковой обработке, шифрование хранения — ответственность инфраструктуры.

## Зачем это нужно?

### Сценарий 1: Перехват трафика в дата-центре

Компания арендует серверы в colocation. Другой арендатор того же дата-центра имеет доступ к shared network infrastructure. Без шифрования — он может перехватывать трафик между продюсером и брокером. Все сообщения видны: номера карт, персональные данные, финансовые транзакции.

С TLS: даже при перехвате — только зашифрованные данные. Невозможно прочитать без приватного ключа сервера.

### Сценарий 2: Кража дисков

Дата-центр обновляет серверы. Старые диски с данными Kafka отправляются на утилизацию. Без at-rest encryption — данные recoverable с диска даже после форматирования. Злоумышленник покупает диск на eBay, восстанавливает данные.

С LUKS encryption: данные на диске зашифрованы. Без ключа — бесполезны. Утилизация диска безопасна.

### Сценарий 3: Compliance

PCI DSS Requirement 4.1: «Transmit cardholder data via open, public networks MUST encrypt using strong cryptography». Если Kafka передаёт данные карт — они должны быть зашифрованы. Requirement 3.4: «Render PAN unreadable anywhere it is stored» — шифрование at-rest.

Без шифрования — аудит не пройден, штрафы, расторжение контракта с платёжной системой.

## Основные концепции

### TLS для Kafka

**TLS** (Transport Layer Security) — криптографический протокол для защиты соединений.

**Как работает:**
1. **Handshake** — клиент и сервер договариваются о версии TLS, cipher suite, обмениваются сертификатами
2. **Key exchange** — генерация сеансового симметричного ключа через асимметричную криптографию
3. **Encrypted transmission** — все данные шифруются симметричным ключом
4. **Integrity check** — MAC (Message Authentication Code) гарантирует, что данные не изменены

**Что защищает TLS:**
- **Конфиденциальность** — перехваченные данные нечитаемы
- **Целостность** — модификация данных обнаруживается
- **Аутентификация сервера** — клиент уверен, что подключается к настоящему брокеру

**Что НЕ защищает TLS (без mTLS):**
- **Аутентификация клиента** — кто подключается, TLS не проверяет
- **Авторизация** — что можно делать

### Версии TLS

| Версия | Алгоритмы | Статус |
|--------|-----------|--------|
| SSL 2.0 | Export cipher suites | ❌ Устарел, небезопасен |
| SSL 3.0 | CBC, RC4 | ❌ Устарел, POODLE attack |
| TLS 1.0 | CBC, RC4 | ❌ Устарел |
| TLS 1.1 | CBC | ❌ Устарел |
| TLS 1.2 | CBC, GCM, ECDHE | ✅ Приемлем |
| TLS 1.3 | AEAD, ECDHE, 0-RTT | ✅ Рекомендуется |

**Рекомендация:** TLS 1.2 minimum, TLS 1.3 preferred.

### Cipher Suites

Cipher suite — комбинация алгоритмов для handshake и шифрования.

| Компонент | TLS 1.2 | TLS 1.3 |
|-----------|---------|---------|
| Key exchange | RSA, DHE, ECDHE | ECDHE (always) |
| Authentication | RSA, ECDSA, DSS | RSA, ECDSA, EdDSA |
| Symmetric encryption | AES-CBC, AES-GCM, ChaCha20 | AES-GCM, ChaCha20 |
| Hash | SHA-256, SHA-384 | SHA-256, SHA-384 |

**Рекомендуемые suites для Kafka:**
- TLS_AES_256_GCM_SHA384 (TLS 1.3)
- TLS_CHACHA20_POLY1305_SHA256 (TLS 1.3)
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (TLS 1.2)
- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 (TLS 1.2)

### Настройка TLS на брокере

```
# server.properties
# Listener с SSL
listeners=SSL://:9093

# Keystore (сертификат брокера + приватный ключ)
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password

# Truststore (доверенные CA)
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststore-password

# Протоколы и cipher suites
ssl.protocol=TLSv1.3
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256

# mTLS (взаимная аутентификация)
ssl.client.auth=required  # Или 'requested' для опциональной
```

### Keystore и Truststore

**Keystore** — хранит сертификат брокера (public key) и приватный ключ. Защищён паролем.

**Truststore** — хранит доверенные CA (Certificate Authorities). Используется для проверки сертификатов клиентов (при mTLS) или сервера (при проверке).

**Создание (simplified):**
1. Создать keystore с ключевой парой
2. Создать CSR (Certificate Signing Request)
3. Подписать CSR внутренним CA (или публичным)
4. Импортировать подписанный сертификат в keystore
5. Импортировать CA в truststore

### Настройка клиента

```
# producer.properties / consumer.properties
security.protocol=SSL

# Truststore (доверяем серверу)
ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
ssl.truststore.password=truststore-password

# Для mTLS:
ssl.keystore.location=/var/private/ssl/kafka.client.keystore.jks
ssl.keystore.password=keystore-password
```

### SASL_SSL — комбинация аутентификации и шифрования

SASL_SSL = SASL (аутентификация) + SSL (шифрование). Это рекомендуемая конфигурация для production.

```
# server.properties
listeners=SASL_SSL://:9093
security.inter.broker.protocol=SASL_SSL

# SASL
sasl.enabled.mechanisms=SCRAM-SHA-256
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# SSL (keystore/truststore)
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=keystore-password
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststore-password
```

Клиентская конфигурация:
```
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="alice" password="alice-secret";
ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
ssl.truststore.password=truststore-password
```

### At-rest шифрование

Kafka не имеет встроенного at-rest шифрования. Варианты:

| Уровень | Решение | Когда использовать |
|---------|---------|-------------------|
| **OS-level** | LUKS, dm-crypt, BitLocker | Bare metal, on-premise |
| **Cloud** | AWS EBS encryption, Azure Disk Encryption | Облако |
| **Filesystem** | eCryptfs, EncFS | Специфические случаи |
| **Application** | Шифрование сообщений перед записью | Максимальная security |

**LUKS для Kafka:**
```
# Создать зашифрованный раздел
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 kafka-data
mkfs.ext4 /dev/mapper/kafka-data
mount /dev/mapper/kafka-data /var/lib/kafka

# В postgresql.conf (аналог для Kafka — log.dirs)
log.dirs=/var/lib/kafka
```

**AWS EBS encryption:**
```
# При создании инстанса
EBS volumes:
  - Encrypted: Yes
  - KMS key: aws/ebs или customer-managed key
```

## Сравнение подходов

| Подход | In-transit | At-rest | Сложность | Производительность |
|--------|-----------|---------|-----------|-------------------|
| **PLAINTEXT** | ❌ Нет | ❌ Нет | Низкая | Максимальная |
| **SSL (только сервер)** | ✅ Да | ❌ Нет | Средняя | Снижение ~10-20% |
| **SASL_SSL** | ✅ Да | ❌ Нет | Средняя | Снижение ~15-25% |
| **mTLS** | ✅ Да | ❌ Нет | Высокая | Снижение ~15-25% |
| **SASL_SSL + LUKS** | ✅ Да | ✅ Да | Высокая | Снижение ~20-30% |
| **SASL_SSL + Cloud KMS** | ✅ Да | ✅ Да | Средняя | Снижение ~20-30% |

**Производительность TLS:**
- TLS handshake — однократные накладные расходы при подключении
- Symmetric encryption — AES-NI (hardware acceleration) минимизирует overhead
- TLS 1.3 — быстрее 1.2 (1-RTT handshake)
- Ключевое узкое место — не шифрование, а сетевой round-trip

## Уроки из реальных инцидентов

### Инцидент 1: Heartbleed и Kafka (2014)

Heartbleed (CVE-2014-0160) — уязвимость в OpenSSL, позволяющая читать память процесса. Kafka-брокеры использовали OpenSSL для TLS.

**Хронология:** Злоумышленник отправлял malformed heartbeat request. Брокер отвечал 64KB памяти, включая приватные ключи, пароли, сообщения. Повторение — чтение всей памяти процесса.

**Ущерб:** Компрометация приватных ключей. Возможность расшифровки прошлых и будущих соединений. Утечка sensitive данных из памяти.

**Как защита изменила бы исход:** Регулярное обновление OpenSSL. Мониторинг security advisories. Использование Perfect Forward Secrecy (ECDHE) — даже при компрометации приватного ключа, прошлые сессии остаются защищёнными. Минимизация данных в памяти процесса.

### Инцидент 2: Отсутствие at-rest шифрования (2020)

Компания мигрировала в облако. Старые EBS volumes с данными Kafka были «удалены» — но облачный провайдер физически уничтожает данные не сразу. Через задержку в несколько дней другой клиент получил тот же физический диск. Данные восстановлены.

**Ущерб:** Утечка данных за 2 года. Регуляторный штраф. Расследование.

**Как защита изменила бы исход:** EBS encryption с KMS. Даже при получении физического диска — данные зашифрованы, ключ в KMS. Без доступа к KMS — бесполезно. AWS гарантирует: encrypted volumes никогда не попадут к другому клиенту в незашифрованном виде.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **OpenSSL / BoringSSL** | TLS implementation | Runtime | NIST SC-13 |
| **Keytool / OpenSSL CLI** | Управление сертификатами | Настройка | NIST SC-17 |
| **Let's Encrypt / Internal CA** | Выдача сертификатов | Настройка | NIST SC-17 |
| **HashiCorp Vault / AWS KMS** | Управление ключами | CI/CD, runtime | NIST SC-28 |
| **Cert-manager (Kubernetes)** | Автоматическая ротация сертификатов | Kubernetes | NIST SC-17 |
| **LUKS / dm-crypt** | OS-level disk encryption | Настройка сервера | NIST SC-28 |
| **Cloud encryption (EBS, Azure)** | Managed encryption | Облако | SOC 2, ISO 27001 |
| **SSL Labs / testssl.sh** | Проверка конфигурации TLS | Регулярно | CIS Benchmarks |

## Архитектурные решения

### Defense-in-Depth для шифрования

| Слой | Защита | Ключевой вопрос |
|------|--------|----------------|
| Network | VLAN, firewall | Кто имеет сетевой доступ? |
| Transport | TLS 1.3 | Перехваченный трафик читаем? |
| Authentication | mTLS или SASL | Кто подключается? |
| Storage | LUKS / Cloud KMS | Украденный диск читаем? |
| Application | Шифрование сообщений | При компрометации Kafka — данные защищены? |
| Backup | Шифрование backup | Backup в безопасном месте? |

### Метрики шифрования

| Метрика | Целевое значение | Проверка |
|---------|-----------------|----------|
| TLS coverage | 100% | `kafka-acls` + мониторинг |
| TLS version | TLS 1.2+ | SSL Labs scan |
| Weak cipher usage | 0 | testssl.sh |
| Certificate expiration | > 30 дней | Monitoring |
| Certificate revocation | Актуальный CRL | Ежедневно |
| Disk encryption | 100% | OS-level check |
| Backup encryption | 100% | Backup verification |

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «SSL достаточно для безопасности Kafka».

SSL обеспечивает конфиденциальность в пути. Но не аутентифицирует клиентов (без mTLS), не авторизует, не шифрует at-rest. Нужна комбинация: SSL + SASL + ACL + disk encryption.

**Ошибка 2:** «Шифрование сильно снижает производительность, его лучше отключить».

Modern CPUs имеют AES-NI instructions — hardware acceleration для AES. Overhead TLS 1.3 с AES-GCM — 5-15%. Для большинства систем это приемлемо. «Не шифровать из-за производительности» — false economy. Утечка данных обойдётся дороже.

**Ошибка 3:** «Самоподписанный сертификат — ок для внутреннего использования».

Самоподписанные сертификаты требуют ручного распространения trust. Риск: человеческий фактор (забыли обновить, неверный truststore). Рекомендация: internal CA с автоматическим распространением. Let's Encrypt для публичных endpoint. Vault для dynamic certificates.

### Три ситуационных вопроса

**Вопрос 1:** «Как обеспечить end-to-end шифрование в Kafka, где продюсер и консьюмер не доверяют брокерам?»

Ответ: **Application-level encryption**. Продюсер шифрует сообщение перед записью в Kafka (например, с использованием public key консьюмера). Брокер видит только зашифрованные данные. Консьюмер расшифровывает своим приватным ключом. Брокер не имеет доступа к содержимому. Это **end-to-end encryption** (E2EE). Минус: теряется возможность обработки данных в Kafka (Streams, ksqlDB). Плюс: максимальная безопасность.

**Вопрос 2:** «Как ротировать TLS сертификаты без downtime?»

Ответ: **Dual certificate support**. Брокер настроен принимать и старый, и новый сертификат одновременно. Обновляем клиентов по очереди. Затем обновляем брокеров по очереди. **Rolling restart** через orchestrator (Kubernetes, Ansible). **Cert-manager** в Kubernetes автоматизирует этот процесс. **Graceful reload** — некоторые версии Kafka поддерживают reload конфигурации без restart.

**Вопрос 3:** «PCI DSS требует шифрования данных карт при передаче. Как это реализовать в Kafka?»

Ответ: **TLS 1.2+** для всех соединений (prodюсер-брокер, брокер-брокер, брокер-консьюмер). **mTLS** для взаимной аутентификации. **Strong cipher suites** (AES-256-GCM, ECDHE). **No SSL termination** — шифрование end-to-end, не прерывается на load balancer. **Encryption at-rest** — LUKS или EBS encryption для log.dirs. **Key management** — KMS, регулярная ротация. **Audit** — логирование всех криптографических операций.

## Чек-лист понимания

- [ ] Что защищает **TLS**? Что не защищает?
- [ ] Какие **версии TLS** безопасны?
- [ ] Чем **TLS 1.3** лучше 1.2?
- [ ] Что такое **cipher suite**?
- [ ] Что такое **keystore** и **truststore**?
- [ ] Что такое **mTLS**?
- [ ] Как **SASL_SSL** отличается от SSL?
- [ ] Какие варианты **at-rest шифрования** для Kafka?
- [ ] Какова **производительность** TLS?
- [ ] Как **ротировать сертификаты**?

### Ответы на чек-лист

1. **TLS** защищает: конфиденциальность (шифрование), целостность (MAC), аутентификацию сервера. **Не защищает:** аутентификацию клиента (без mTLS), авторизацию, данные в памяти/на диске.

2. **Безопасны:** TLS 1.2 и 1.3. **Не безопасны:** SSL 2.0, 3.0, TLS 1.0, 1.1 (уязвимости, устаревшие алгоритмы).

3. **TLS 1.3** быстрее (1-RTT handshake), безопаснее (удалены устаревшие алгоритмы), проще (меньше настроек). Обязательно использует Perfect Forward Secrecy.

4. **Cipher suite** — набор алгоритмов: key exchange, authentication, symmetric encryption, hash. Пример: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384.

5. **Keystore** — хранит сертификат и приватный ключ сервера. **Truststore** — хранит доверенные CA для проверки сертификатов.

6. **mTLS** (mutual TLS) — взаимная аутентификация. Клиент и сервер проверяют сертификаты друг друга. Используется в Kubernetes, high-security средах.

7. **SASL_SSL** = SASL (аутентификация, например SCRAM) + SSL (шифрование). **SSL** = только шифрование (без аутентификации клиента, если не mTLS).

8. **At-rest:** OS-level (LUKS, dm-crypt), cloud (EBS encryption, Azure Disk Encryption), application-level (шифрование сообщений перед записью).

9. **Производительность TLS:** Modern CPU с AES-NI — overhead 5-15%. TLS 1.3 быстрее 1.2. Узкое место — сетевой round-trip, не шифрование.

10. **Ротация:** dual certificate (старый + новый), rolling restart, cert-manager для автоматизации, graceful reload если поддерживается.

## Ключевые выводы для собеседования

- **TLS 1.3 — preferred, 1.2 — minimum** — никаких старых версий
- **SASL_SSL — стандарт production** — аутентификация + шифрование
- **At-rest — ответственность инфраструктуры** — LUKS, cloud KMS
- **Performance overhead — приемлемый** — AES-NI, TLS 1.3
- **Certificate management — автоматизировать** — cert-manager, Vault

---

_Статья создана на основе анализа материалов по безопасности Kafka_
