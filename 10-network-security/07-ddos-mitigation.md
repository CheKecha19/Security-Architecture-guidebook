# Защита от DDoS: От перегрузки до распределённой обороны

## Часть 1. Знакомство с проблемой: Почему DDoS — это цифровая осада замка?

Представьте средневековый замок, который осаждают. Но вместо армии профессиональных солдат к воротам подходит толпа из миллиона крестьян. Каждый по отдельности безобиден, но вместе они блокируют все подходы, переполняют склады провиантом своими требованиями, и нормальные гонцы не могут пробиться через толпу. Замок не падает от меча, но останавливается от перегрузки.

**DDoS** (Distributed Denial of Service) — атака доступностью через перегрузку. Злоумышленник не взламывает систему, а делает её недоступной для легитимных пользователей. **Volumetric** атаки переполняют канал (биты в секунду). **Protocol** атаки перегружают состояние сетевых устройств (соединения в секунду). **Application** атаки исчерпвывают ресурсы сервера (HTTP-запросы, медленные соединения).

В 2026 году DDoS достиг апогея. **IoT-бутнеты** из миллионов устройств (камеры, роутеры, термостаты) генерируют трафик >3 Тбит/с. **Ransom DDoS** — злоумышленники требуют выкуп за прекращение атаки. **Reflection/amplification** — отправка запросов на сервисы с поддельным source, которые отвечают во много раз больше. Защита требует **multi-layered approach**: edge scrubbing, rate limiting, anycast, CDN, WAF.

| Тип DDoS | Уровень | Цель | Пример | Масштаб |
|----------|---------|------|--------|---------|
| Volumetric | L3-L4 | Переполнение канала | UDP flood, DNS amplification | 1-3 Тбит/с |
| Protocol | L3-L4 | Перегрузка state table | SYN flood, ACK flood, Ping of Death | 1-10M pps |
| Application | L7 | Исчерпание ресурсов | HTTP flood, Slowloris, RUDY | 10-100K req/s |

**Эволюция:** В 1996 **Panix** — первый крупный DDoS (SYN flood). В 2000 **Yahoo!, eBay, Amazon, CNN** атакованы за неделю — DDoS становится оружием. В 2010 **Anonymous** использует **LOIC** (Low Orbit Ion Cannon) — DDoS становится политическим инструментом. В 2016 **Mirai** (IoT ботнет) атакует **Dyn** — 1.2 Тбит/с через DNS amplification. В 2018 **Memcached amplification** — 51,000x amplification factor, атаки >1.7 Тбит/с. В 2024 **HTTP/2 Rapid Reset** (CVE-2023-44487) — новый вектор application-layer DDoS.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Финтех и ransom DDoS
Платёжный процессор получает письмо: "Заплатите 5 BTC, или ваши серверы лягут". Через час начинается **application-layer DDoS**: 100K HTTP POST-запросов в секунду на `/api/payments`. WAF пытается фильтровать, но запросы выглядят легитимными (правильные headers, валидные параметры). Backend-сервера перегружаются валидацией. Платежи не проходят 4 часа. Потери: $500K в час простоя + репутационный ущерб. Компания не платит выкуп, но закупает **CloudFlare Magic Transit**.

### Сценарий 2: Игровая платформа и UDP amplification
Онлайн-игра с 10M пользователей. Злоумышленник использует **NTP amplification**: отправляет `monlist` запросы на 10K открытых NTP-серверов с поддельным source IP (IP адрес игрового сервера). Каждый запрос 1 байт, ответ 446 байт (список 600 клиентов). Амплификация 446x. Игровой сервер получает 100 Гбит/с UDP трафика. Провайдер не справляется, null-routes IP. Игра недоступна 6 часов. **Урок:** Открытые NTP/DNS/memcached — пушки для amplification.

### Сценарий 3: SaaS и HTTP/2 Rapid Reset
Облачный SaaS использует **HTTP/2** для API. Злоумышленник эксплуатирует **CVE-2023-44487** (HTTP/2 Rapid Reset): отправляет большое количество HTTP/2 **RST_STREAM** кадров, отменяя запросы мгновенно. Каждый запрос создаёт state на сервере, но RST_STREAM освобождает его на клиенте. Сервер тратит ресурсы на обработку, клиент — нет. 10K таких запросов в секунду исчерпвают **thread pool** backend'а. API ложится. **Урок:** Новые протоколы = новые векторы.

## Часть 3. Техническая глубина: Как устроены DDoS-атаки и защита

### Amplification: Пушки для DDoS
**Аналогия:** Это как если бы вы послали письмо с просьбой "пришлите мне каталог" на адрес жертвы. Магазин отправляет тяжёлую книгу (каталог) жертве, хотя она не просила. Атакующий посылает тысячи таких писем от имени жертвы — её почтовый ящик переполняется.

**Механизм amplification:**
1. Злоумышленник отправляет **запрос** на публичный сервис (NTP, DNS, memcached, SSDP) с **поддельным source IP** (жертва)
2. Сервер отвечает **большим пакетом** на адрес жертвы
3. **Amplification factor** = размер ответа / размер запроса

| Протокол | Запрос | Ответ | Amplification | Mitigation |
|----------|--------|-------|---------------|------------|
| DNS | 60 bytes | 4000+ bytes (ANY) | 54x | Disable ANY, Response Rate Limiting |
| NTP (monlist) | 1 byte | 446 bytes | 446x | Disable monlist, upgrade to ntpd 4.2.7p26+ |
| Memcached | 15 bytes | 750K bytes | 51,000x | Firewall port 11211, disable UDP |
| SSDP | 60 bytes | 800+ bytes | 13x | Disable UPnP, block port 1900 |
| SNMP | 60 bytes | 1500+ bytes | 25x | Community string hardening, ACL |

**Защита от amplification:**
- **BCP38 / uRPF** (Unicast Reverse Path Forwarding) — проверяет, что source IP пакета маршрутизируется через интерфейс, с которого пришёл. Предотвращает IP-spoofing.
- **Disable unnecessary services**: `monlist` в NTP, ANY-запросы в DNS, UDP в memcached.
- **Rate limiting**: Ограничение на количество запросов с одного IP.
- **CDN / Anycast**: Распределение трафика по множеству точек присутствия.

### SYN flood: Перегрузка state table
**SYN flood** — классическая protocol-layer атака. Злоумышленник отправляет **SYN** без завершения handshake'а. Сервер выделяет ресурсы на **half-open connection** и ждёт ACK. Полуоткрытых соединений может быть ограниченное количество (backlog). Когда backlog заполнен — легитимные клиенты не могут подключиться.

**Защита от SYN flood:**
- **SYN cookies**: Сервер не сохраняет state до получения ACK. Отправляет **cookie** (MAC от параметров соединения) в SYN-ACK. Клиент должен вернуть cookie в ACK.
- **SYN proxy**: Промежуточное устройство (load balancer, firewall) завершает handshake с клиентом, затем устанавливает отдельное соединение с сервером.
- **Connection rate limiting**: Ограничение на количество SYN в секунду с одного IP.
- **TCP SYN Cookies**: `sysctl net.ipv4.tcp_syncookies = 1` на Linux.

### Application-layer DDoS: Slowloris, RUDY, HTTP flood
**Slowloris** (2009) — открывает HTTP-соединение и отправляет **частичные HTTP headers** очень медленно (по одному байту каждые 15 секунд). Сервер ждёт завершения headers, удерживая соединение открытым. 10K Slowloris-соединений исчерпвают **MaxClients** в Apache.

**RUDY** (R-U-Dead-Yet) — тоже самое, но с **POST-данными**: отправляет body очень медленно.

**HTTP/2 Rapid Reset** (2023) — отправляет большое количество **RST_STREAM** кадров. Сервер создаёт state для каждого stream'а, клиент мгновенно отменяет. Асимметрия ресурсов.

| Application DDoS | Механизм | Цель сервера | Защита |
|------------------|----------|--------------|--------|
| Slowloris | Медленные headers | Ждать завершения request headers | Timeout на headers, min data rate |
| RUDY | Медленный POST body | Ждать завершения body | Body timeout, max body size |
| HTTP flood | Быстрые валидные запросы | Обработать каждый запрос | Rate limiting, CAPTCHA, CDN caching |
| Cache-bypass | Unique URL params | Не использовать cache | Normalization, ignore certain params |
| Connection exhaustion | Держать соединения открытыми | Удерживать file descriptors | Connection timeout, max connections per IP |

## Часть 4. Архитектура защиты: Defense in Depth для DDoS

### Multi-layered DDoS defense

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| Edge (Provider) | Scrubbing center | — | Cloudflare Magic Transit, AWS Shield Advanced |
| Edge (Provider) | Anycast | RFC 1546 | Cloudflare, Fastly, AWS CloudFront |
| Edge (Provider) | BGP Flowspec | RFC 8955 | RTBH, traffic steering |
| Network | Rate limiting | — | iptables, nftables, Cisco CAR |
| Network | SYN cookies / proxy | RFC 4987 | Linux kernel, F5, A10 |
| Application | CDN caching | — | Cloudflare, Akamai, AWS CloudFront |
| Application | WAF rate limiting | — | AWS WAF, Cloudflare Rate Limiting |
| Application | CAPTCHA / challenge | — | reCAPTCHA, hCaptcha |
| Application | Auto-scaling | — | Kubernetes HPA, AWS Auto Scaling |

### Мониторинг и метрики
**Ключевые метрики DDoS:**
- **Bits per second (bps)**: Вolumetric трафик
- **Packets per second (pps)**: Protocol-layer load
- **Requests per second (rps)**: Application-layer load
- **SYN ratio**: SYN / (SYN+ACK+FIN) — внезапный рост = SYN flood
- **Error rate**: 5xx / total responses — сервер перегружен
- **Geographic distribution**: Необычный трафик из определённых стран
- **ASN distribution**: Трафик из необычных автономных систем

### Реальные инциденты

**Инцидент 1: Dyn DDoS via Mirai (2016)**
**Dyn** — крупнейший DNS-провайдер. Атака через **Mirai botnet** (IoT-устройства с дефолтными паролями). Использовались **DNS amplification** и **TCP flood**. Пик: 1.2 Тбит/с. Twitter, GitHub, Reddit, Netflix, Spotify — downtime 6-11 часов. **Урок:** IoT + amplification = масштабный DDoS. Защита: IoT hardening (менять дефолтные пароли), scrubbing centers (Cloudflare, AWS Shield), anycast DNS, response rate limiting на авторитативных NS.

**Инцидент 2: GitHub Memcached DDoS (2018)**
**GitHub** подвергся атаке в 1.35 Тбит/с через **Memcached amplification**. Злоумышленник отправлял `stats` запросы на открытые memcached-серверы с поддельным source IP GitHub. Amplification: 51,000x. GitHub использовал **Akamai Prolexic** — трафик был обнаружен и mitigated за 8 минут. **Урок:** Amplification-атаки масштабируются экспоненциально. Защита: memcached firewall (port 11211), BCP38/uRPF, scrubbing centers с auto-mitigation.

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"У нас CDN, значит защищены от DDoS"** — CDN защищает от volumetric и некоторых application атак, но: (1) не защищает от direct-to-origin атак (если IP origin известен); (2) не защищает от API-атак на не-кэшируемые endpoints; (3) не защищает от network-layer attacks на origin. CDN — один слой.
2. **"Синтетический банк и SYN flood"** — SYN cookies решают проблему на сервере, но volumetric SYN flood переполнит канал до сервера. Нужен scrubbing на уровне провайдера. Defense in depth.
3. **"DDoS — это только volumetric"** — Application-layer DDoS (Slowloris, HTTP flood) может положить сервер 10K req/s, что не является volumetric. WAF + rate limiting + challenge required.

### Три ситуационных вопроса для Senior+

**Q1:** Как защититься от application-layer DDoS, когда запросы выглядят легитимными?
**A:** (1) **Behavioral analysis**: ML-модель, обученная на нормальном трафике, обнаруживает аномалии (необычная последовательность endpoint'ов, необычное время между запросами); (2) **Proof-of-work**: Требовать клиента решить небольшую вычислительную задачу (hashcash) перед обработкой запроса; (3) **Device fingerprinting**: Отличать реальные браузеры от headless/chrome (Puppeteer, Playwright); (4) **Progressive challenge**: Лёгкие запросы без challenge, подозрительные — CAPTCHA; (5) **API-specific**: Rate limiting per API key, not IP — обход IP-based лимитов через прокси не работает. Ключевое: application DDoS требует application-layer defense.

**Q2:** Почему memcached amplification был так опасен (51,000x), и как предотвратить подобное?
**A:** **Memcached** по умолчанию слушал UDP-порт 11211 без аутентификации. Запрос `stats` (15 байт) возвращал статистику (до 750KB). Amplification = 51,000x. **Причина:** (1) UDP без аутентификации; (2) Ответ во много раз больше запроса; (3) Публично доступные серверы. **Защита:** (1) **Firewall port 11211** — memcached не должен быть публичным; (2) **Disable UDP** — memcached использует TCP; (3) **Authentication** — SASL для memcached; (4) **BCP38/uRPF** — предотвращает IP-spoofing на уровне провайдера. Универсальный принцип: любой UDP-сервис с большим ответом — потенциальная amplification-га.

**Q3:** Какой подход к DDoS-защите выбрать: on-premise appliance или cloud scrubbing?
**A:** **Trade-off analysis:** On-premise (F5, A10, Arbor): (1) Полный контроль; (2) Низкая latency; (3) Высокая capex; (4) Масштабируется до определённого предела (10-40 Gbит/с); (5) Требует skilled personnel. Cloud scrubbing (Cloudflare Magic Transit, AWS Shield Advanced, Akamai Prolexic): (1) Почти бесконечный масштаб (3+ Тбит/с); (2) Opex модель; (3) Добавляет latency (трафик идёт через scrubbing center); (4) Auto-mitigation; (5) Shared responsibility. **Рекомендация:** Гибрид — on-premise для "нормальных" атак (<10 Gbит/с), cloud scrubbing для крупных (>100 Gbит/с). Always Ready — cloud scrubbing pre-configured, activation via BGP announcement.

### Чек-лист защиты от DDoS (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | Cloud scrubbing pre-configured (BGP ready)? | Cloudflare Magic Transit или AWS Shield Advanced | Обязательно |
| 2 | Origin IP скрыт от публичного доступа? | CDN proxy, нет прямого DNS A-record на origin | Обязательно |
| 3 | BCP38 / uRPF включены на border routers? | `show ip interface` или конфигурация | Обязательно |
| 4 | SYN cookies включены на всех Linux-серверах? | `sysctl net.ipv4.tcp_syncookies` | Обязательно |
| 5 | Rate limiting на application layer (API)? | Nginx limit_req, AWS WAF rate rules | Высокая |
| 6 | NTP / DNS / Memcached не публично доступны? | `nmap -sU -p 123,53,11211` | Обязательно |
| 7 | CDN кэширует static content (уменьшает origin load)? | Cache hit ratio >80% | Высокая |
| 8 | Auto-scaling настроен для traffic spikes? | HPA, VPA, или cloud auto-scaling | Высокая |
| 9 | DDoS playbook и runbook протестированы? | Tabletop exercise quarterly | Высокая |
| 10 | Anycast DNS для распределения запросов? | Route 53, Cloudflare DNS | Средняя |

### Key Takeaways
1. **DDoS — атака доступностью, не конфиденциальности** — но downtime стоит дороже многих breaches.
2. **Amplification = IoT + UDP + misconfiguration** — закрывать ненужные сервисы, включать BCP38.
3. **Multi-layer defense обязателен** — edge scrubbing + network rate limiting + application challenge + auto-scaling.
4. **Application DDoS > volumetric** — 10K req/s могут положить API, не будучи volumetric.
5. **Always Ready, not On-Demand** — scrubbing должен быть pre-configured, activation через BGP.
