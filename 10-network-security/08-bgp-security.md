# Безопасность BGP: От доверия к маршрутам до криптографической валидации

## Часть 1. Знакомство с проблемой: Почему BGP — это система, где каждый может объявить "эта улица моя"?

Представьте город без кадастровой палаты. Любой житель может поставить знак: "Эта улица теперь называется по-другому, и все дома здесь — мои". Почтовые службы, такси, экстренные службы — все начинают использовать новое название, потому что никто не проверяет, действительно ли этот человек владелец. Это **BGP** (Border Gateway Protocol) — протокол, который управляет маршрутизацией между автономными системами (AS) в интернете.

**BGP** — это **path-vector** протокол. Каждая AS анонсирует **IP-префиксы** (блоки адресов), которые ей "принадлежат". Соседние AS (peers) обмениваются этими анонсами через **BGP sessions** (TCP порт 179). Но BGP не проверяет подлинность анонса. Если AS 64500 заявляет, что владеет префиксом `8.8.8.0/24` (Google), и её соседи принимают этот анонс — трафик к Google пойдёт через AS 64500. Это **BGP hijacking**.

В 2026 году BGP остаётся **одной из главных systemic vulnerabilities** интернета. **Route leaks** (ошибочные анонсы) случаются ежемесячно. **BGP hijacking** используется для крипто-краж, перехвата трафика, цензуры. **RPKI** (Resource Public Key Infrastructure) добавляет криптографическую валидацию, но adoption <50%.

| Атака BGP | Механизм | Результат | Частота |
|-----------|----------|-----------|---------|
| BGP Hijacking | Ложная анонсация чужого префикса | Перехват трафика | Ежемесячно |
| Route Leak | Ошибочная передача префиксов дальше, чем нужно | Непреднамеренный blackout | Еженедельно |
| BGP DoS | Лавина анонсов / withdrawals | Перегрузка роутеров | Редко |
| Prefix squatting | Анонсация неиспользуемых префиксов | DoS для legitimate owner | По ситуации |

**Эволюция:** В 1989 **BGP-4** стал стандартом (RFC 1105, затем RFC 4271). В 1998 первые публичные **BGP hijacking** инциденты. В 2008 **Pakistan Telecom** случайно анонсировал YouTube (/24) — YouTube был недоступен 2 часа. В 2014 **RPKI** начал deployment (RFC 6480). В 2018 **ROA** (Route Origin Authorization) — криптографическое подтверждение права анонсировать префикс. В 2018 **BGPsec** (RFC 8205) — криптографическая защита всего BGP path, но adoption близок к нулю из-за complexity. В 2024 **AS0 ROA** — способ "занулить" префикс, объявив его неиспользуемым.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Криптобиржа и BGP hijacking для кражи
Криптобиржа использует `exchange-crypto.com` для API и веб. Злоумышленник анонсирует `/24`, содержащий IP сервера биржи, через скомпрометированного или злонамеренного партнёра (маленький ISP). Часть интернета (в зависимости от path length) начинает отправлять трафик через злоумышленника. Он запускает **TLS MITM** (с валидным сертификатом от скомпрометированного CA или через certificate transparency gap). API-клиенты отправляют запросы на фейковый сервер, теряют API keys. Потери: $10M в криптовалюте за 2 часа.

### Сценарий 2: Правительственная организация и route leak
Госучреждение использует **multi-homing**: два upstream-провайдера для redundancy. Один провайдер по ошибке анонсирует full table (все префиксы) второму провайдеру, который принимает их и анонсирует дальше. Результат: **route leak** — трафик из всего интернета начинает идти через одного провайдера, перегружая его. Госучреждение теряет связь. Оказалось, что **max-prefix limit** и **filtering** не были настроены на пиринговой сессии. Урок: Configuration error = global impact.

### Сценарий 3: CDN-провайдер и prefix squatting
CDN-провайдер использует префикс, который ранее принадлежал другой компании (bankrupt). **RIR** (RIPE, ARIN) освободил префикс, но CDN начал его использовать без обновления **whois** и **ROA**. Другой злоумышленник анонсирует тот же префикс через другого провайдера. Часть интернета выбирает путь злоумышленника (more specific prefix = higher priority). CDN теряет 30% трафика. **Урок:** Prefix management требует operational discipline.

## Часть 3. Техническая глубина: Как устроен BGP и его уязвимости

### BGP Path Selection: Как маршрутизаторы выбирают путь
**Аналогия:** Это как если бы водители выбирали дорогу на основе: (1) самый короткий маршрут; (2) самый быстрый; (3) рекомендации друзей; (4) качество дороги. Но никто не проверяет, действительно ли друг знает этот район.

**BGP path selection** (упрощённо):
1. **Highest Weight** (Cisco proprietary, local preference)
2. **Highest Local Preference** (AS-wide preference)
3. **Locally originated** (originated by this router)
4. **Shortest AS-Path** (меньше AS в пути = "ближе")
5. **Lowest Origin type** (IGP < EGP < Incomplete)
6. **Lowest MED** (Multi-Exit Discriminator, между соседними AS)
7. **eBGP over iBGP** (external preferred over internal)
8. **Lowest IGP metric to next-hop** (внутренняя близость)
9. **Oldest path** (stability)
10. **Lowest Router ID**

**Уязвимости:**
- **Shortest AS-Path wins**: Злоумышленник может анонсировать короткий путь, перехватывая трафик.
- **More specific prefix wins**: `/24` побеждает `/23`, даже если `/23` "правильнее". Злоумышленник анонсирует более специфичный префикс.
- **No authentication**: BGP-сессия может быть установлена кем угодно, если IP достижим.

| Атрибут BGP | Назначение | Как эксплуатируется |
|------------|-----------|---------------------|
| AS-Path | Список AS в пути | Prepending (искусственное удлинение) или short path hijack |
| Next-Hop | IP адрес следующего роутера | Перенаправление через подконтрольный роутер |
| MED | Preference между multi-exit | Игнорируется многими AS, не эффективен |
| Community | Теги для policy | Подмена community = изменение policy соседей |
| Origin | Как префикс был создан | Type Incomplete = less trusted, but not rejected |

### RPKI: Криптографическая валидация
**RPKI** (Resource Public Key Infrastructure) — система, где **RIR** (RIPE, ARIN, APNIC) криптографически подписывают, какие AS имеют право анонсировать какие префиксы.

**Компоненты RPKI:**
- **ROA** (Route Origin Authorization): "AS 12345 может анонсировать 192.0.2.0/24"
- **Certificate**: RIR подписывает ROA своим ключом
- **Repository**: Публичное хранилище ROA (rsync, RRDP)
- **Validator**: Роутер или отдельный сервер, который загружает и проверяет ROA
- **BGP Decision**: Роутер проверяет: если ROA существует и не совпадает — reject (или depreference)

**Статусы RPKI:**
- **Valid**: Анонс соответствует ROA
- **Invalid**: Анонс противоречит ROA (AS или prefix не совпадают)
- **Not Found**: Нет ROA для этого префикса

**Проблемы RPKI:**
1. **Adoption**: <50% префиксов имеют ROA (хотя растёт)
2. **Invalid filtering**: Многие AS не отбрасывают Invalid, а только понижают preference
3. **ROA maxLength**: Если ROA разрешает /24, а анонс /25 — Invalid. Но /25 может быть легитимным (traffic engineering). Тонкая настройка.
4. **Relying party software**: RIPE Validator, Routinator, OctoRPKI — качество варирует
5. **Takover risk**: Если RIR ключ скомпрометирован = фальшивые ROA

### BGPsec: Защита всего пути
**BGPsec** (RFC 8205) криптографически подписывает **весь AS-Path**, не только origin. Каждая AS добавляет **BGPsec_Path_Segment** с подписью предыдущего.

**Проблемы BGPsec:**
1. **Deployment**: Требует обновления **всех** роутеров на пути. Если одна AS не поддерживает — путь не валидируется.
2. **Performance**: Криптографические операции на каждый BGP update (миллионы префиксов) — нагрузка на роутер.
3. **Key management**: Ключи должны ротироваться, но BGPsec не предлагает механизм.
4. **Incremental deployment**: Невозможно включить постепенно — нужен "flag day".

**Практический итог:** RPKI + ROA работает и внедряется. BGPsec — теоретически идеален, практически мёртв.

## Часть 4. Архитектура защиты: Defense in Depth для BGP

### Многоуровневая защита BGP

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| Registry | RPKI + ROA | RFC 6482 | RIPE NCC, ARIN, APNIC |
| Registry | AS0 ROA | RFC 6483 | "Tombstone" для освобождённых префиксов |
| Router | RPKI validation | RFC 6811 | Cisco, Juniper, Arista (нужен validator) |
| Router | BGPsec | RFC 8205 | Cisco IOS XR, Juniper JunOS (limited) |
| Router | MD5 session auth | RFC 2385 | TCP MD5 signature (deprecated, weak) |
| Router | GTSM / TTL security | RFC 5082 | Проверка TTL (BGP peers = TTL 255) |
| Router | Prefix limits | — | `maximum-prefix` на neighbor |
| Router | Route filters | — | Prefix-list, AS-path filter, community filter |
| Monitoring | BGP monitoring | — | BGPmon, EXA, RIPE RIS, RouteViews |
| Monitoring | RPKI validity tracking | — | RIPE RPKI Dashboard, Cloudflare Radar |

### Мониторинг и метрики
**Ключевые метрики BGP security:**
- **ROA coverage**: % префиксов вашей организации с ROA
- **RPKI validity rate**: % анонсов в интернете, которые Valid
- **BGP session stability**: Количество flaps (up/down) в час
- **Prefix origin changes**: Внезапные изменения origin AS для ваших префиксов
- **Route leak detection**: Появление ваших префиксов в неожиданных AS-Path
- **Hijack detection**: Анонсы ваших префиксов с другим origin AS

### Реальные инциденты

**Инцидент 1: Pakistan Telecom / YouTube (2008)**
**Pakistan Telecom** получила правительственный запрет на блокировку YouTube. Они анонсировали `/24` для YouTube (more specific, чем `/23` от Google). Анонс распространился через **PCCW** (их upstream) в глобальный интернет. YouTube был недоступен 2 часа для большей части мира. **Урок:** Route filtering не работает: PCCW приняла более специфичный префикс. Защита: **RPKI + ROA** — анонс от Pakistan Telecom был бы Invalid (AS не совпадает с ROA). **Max-prefix limits** и **filtering** на пиринговых сессиях обязательны.

**Инцидент 2: Amazon Route53 / BGP Hijacking (2018)**
Злоумышленники анонсировали префиксы **Route53** через украинского провайдера. DNS-запросы к **myetherwallet.com** перенаправлялись на фишинговый сайт. Пользователи отправляли ETH на кошельки злоумышленников. Потеря: ~$150K за 2 часа. **Урок:** BGP hijacking + DNS = идеальная комбинация. Защита: **RPKI + ROA** для валидации origin, **DNSSEC** для защиты от DNS-ответов (хотя BGP hijacking обходит DNSSEC, если resolver в зоне hijack), **DoH/DoT** (resolver не уязвим к локальному hijacking). Многослойная защита.

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"BGP — это протокол провайдеров, нам не важно"** — Любая организация с multi-homing, собственным AS, или даже с cloud-only presence (BYOIP в AWS) сталкивается с BGP. Hijacking вашего префикса = downtime, репутационный ущерб, потеря данных.
2. **"RPKI решает все проблемы BGP"** — RPKI валидирует **origin** (какая AS анонсирует), но не **path** (через какие AS идёт трафик). BGPsec решает path, но не внедрён. Route leaks, path manipulation — всё ещё возможны.
3. **"BGP hijacking — это всегда вредоносное"** — Чаще это ошибка конфигурации (route leak). Но эффект тот же. Защита должна предотвращать и злонамеренное, и случайное.

### Три ситуационных вопроса для Senior+

**Q1:** Как защититься от BGP hijacking, если мы не контролируем upstream-провайдеров?
**A:** (1) **RPKI + ROA**: Криптографически заявить, что только ваш AS может анонсировать ваши префиксы. Провайдеры с RPKI validation отклонят ложные анонсы. (2) **Multiple upstream с разными путями**: Hijacking через одного провайдера не затронет других. (3) **Monitoring**: BGPmon, EXA — алерты при появлении необычных анонсов ваших префиксов. (4) **IP anycast**: Распределение префиксов по множеству точек — hijacking одной точки не затронет другие. (5) **Law enforcement**: Для крупных инцидентов — контакт с RIPE/ARIN и провайдерами для "route withdrawal". Ключевое: RPKI — лучшая защита, но требует adoption провайдерами.

**Q2:** Почему BGPsec не внедрён, несмотря на 7 лет с RFC 8205?
**A:** (1) **Incremental deployment impossible**: BGPsec требует, чтобы все AS на пути поддерживали его. Невозможен gradual rollout. (2) **Performance**: Криптографические операции (ECDSA) на каждый update для 900K+ префиксов — нагрузка на роутер. (3) **Operational complexity**: Key management, rollover, revocation — не решены. (4) **RPKI "good enough"**: Для большинства атак (origin hijacking) RPKI + ROA достаточно. BGPsec защищает от path manipulation, но такие атаки реже. (5) **Vendor support**: Ограниченная поддержка в Cisco, Juniper. Практический итог: RPKI работает, BGPsec — мёртв.

**Q3:** Как обнаружить route leak в своей сети?
**A:** (1) **Max-prefix limits**: Если сосед внезапно анонсирует >X префиксов — автоматическое отключение сессии. (2) **AS-path filters**: Не принимать префиксы с "длинным" AS-path от customer-пирингов (customer должен анонсировать только свои префиксы). (3) **IRR filters**: Сравнение анонсов с **IRR** (Internet Routing Registry) — RADB, RIPE. (4) **RPKI validity**: Если customer анонсирует префикс с Invalid ROA — reject. (5) **Monitoring**: RIPE RIS, RouteViews — публичные базы BGP-состояний. Аномалии: префикс появляется в неожиданном AS-Path, более специфичный префикс от неизвестного origin. Defense in depth через filters + limits + monitoring.

### Чек-лист защиты BGP (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | RPKI ROA созданы для всех префиксов организации? | RIPE/ARIN RPKI dashboard | Обязательно |
| 2 | BGP-сессии с upstream имеют max-prefix limits? | `show ip bgp neighbors` | Обязательно |
| 3 | BGP MD5 или GTSM (TTL security) включены? | `password` или `ttl-security` в конфиге | Высокая |
| 4 | Route filters на customer-facing сессиях? | Prefix-list + AS-path filter | Обязательно |
| 5 | IRR (RADb/RIPE) объекты актуальны? | `whois -h whois.radb.net` | Высокая |
| 6 | RPKI validation включено на роутерах? | Validator (Routinator, RIPE) + router config | Высокая |
| 7 | Monitoring BGP anomalies (BGPmon/EXA)? | Подписка на alerts | Высокая |
| 8 | AS0 ROA для освобождённых префиксов? | "Tombstone" в RPKI | Средняя |
| 9 | Backup upstream с разными AS-Path? | Multi-homing с diverse paths | Средняя |
| 10 | BGPsec пилотируется (если применимо)? | Limited deployment, mainly research | Низкая |

### Key Takeaways
1. **BGP — доверяй, но проверяй** — архитектурное доверие без криптографии = systemic vulnerability.
2. **RPKI + ROA — работающая защита** — криптографическая валидация origin, adoption растёт.
3. **BGPsec — мёртв** — теоретически идеален, практически невозможен из-за deployment model.
4. **Route leaks чаще hijacking** — конфигурационные ошибки = тот же эффект. Filters + limits обязательны.
5. **Multi-layer defense**: RPKI + filters + monitoring + anycast + diverse upstream.
