# Безопасность SD-WAN и SASE: От distributed edge к облачному периметру

## Часть 1. Знакомство с проблемой: Почему традиционный периметр больше не существует?

Представьте, что ваша компания владела одним большим офисным зданием с центральным КПП. Все входящие и выходящие проходили через этот КПП — охрана проверяла каждого, логировала, досматривала сумки. Теперь компания выросла до сотен филиалов, где каждый филиал — это маленький офис, домашний кабинет сотрудника, или облачное приложение в AWS. Единого КПП больше нет. Каждому филиалу нужен свой КПП, но нанимать охрану в каждый кабинет дорого и неэффективно.

**SD-WAN** и **SASE** — два эволюционных этапа решения этой проблемы. **SD-WAN** (Software-Defined Wide Area Network) виртуализирует WAN — программно управляет маршрутизацией через MPLS, broadband, 4G/5G. **SASE** (Secure Access Service Edge) добавляет безопасность в эту виртуализацию — firewall, SWG, CASB, ZTNA как облачные сервисы, доставляемые из ближайшей точки присутствия (PoP).

В 2026 году SD-WAN уже mainstream, SASE — rapidly growing. **Gartner** прогнозирует, что к 2027 году 50% организаций будут использовать SASE (против 5% в 2021). Но каждый новый слой абстракции создаёт новые security challenges: **централизация точек отказа**, **конфигурационный drift**, **расширение trust boundary** до облачного провайдера.

| Концепция | Что виртуализирует | Главный security challenge | Ключевая технология |
|-----------|-------------------|--------------------------|---------------------|
| SD-WAN | WAN-маршрутизацию | Overlay сеть обходит традиционные сетевые политики | Viptela, Silver Peak, VeloCloud |
| SASE | Доставку security сервисов | Trust boundary расширен до облачного провайдера | Zscaler, Netskope, Cloudflare |
| SSE (Secure Service Edge) | Security часть SASE | Тоже что SASE, но без SD-WAN | Palo Alto, Check Point |
| ZTNA (часть SASE) | Доступ к приложениям | Идентичность вместо IP = dependency на IdP | Zscaler Private Access, Cloudflare Access |

**Эволюция:** В 2014 **SD-WAN** появился как решение: "а не дорого ли поддерживать MPLS?". В 2017 **VeloCloud** приобретён VMware — SD-WAN становится enterprise. В 2019 **Gartner** ввёл термин **SASE** — конвергенция сети и безопасности. В 2021 **COVID-19** ускорил adoption: удалённые работники не идут через центральный офис. В 2023 **ZTNA** внутри SASE заменил VPN для облачных приложений. В 2026 **SSE** (Secure Service Edge) — SASE без SD-WAN, фокус на security.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Ритейл и SD-WAN misconfiguration
Сеть из 500 магазинов использует SD-WAN для связи с центральным офисом. Конфигурация: **overlay VPN** (IPsec) между каждым магазином и двумя хабами (active-active). Администратор настраивает VPN-туннель, используя **pre-shared key** вместо сертификатов. Ключ — "BranchToHub2024!" — хранится в Ansible-репозитории (GitLab). Репозиторий — публичный (ошибка permission). Злоумышленник находит ключ, подключается к VPN-хабу извне, получает доступ к **PCI CDE** через внутреннюю маршрутизацию. Потери: 1M карт скомпрометированы. **Урок:** SD-WAN автоматизация не исключает базовые ошибки.

### Сценарий 2: SaaS-компания и SASE adoption
Облачный SaaS мигрирует с on-premise firewall на **SASE** (Zscaler). Конфигурация: все endpoint'ы отправляют трафик через Zscaler cloud (ZIA — Zscaler Internet Access). Но забыли настроить **ZTNA** для внутренних приложений. Разработчики используют **SSH через public IP** на порте 22 для доступа к staging. Злоумышленник сканирует IP range компании, находит SSH, брутфорсит (слабый пароль). Получает shell, извлекает AWS IAM credentials, сливает S3. **Урок:** SASE защищает интернет-трафик, но не внутренний доступ — нужен ZTNA layer.

### Сценарий 3: Корпорация и SSE как часть cloud migration
Корпорация мигрирует Office 365 и Salesforce в облако. Использует **SSE** (Secure Service Edge) через **Netskope** для: CASB (защита SaaS), DLP, threat protection. Но администратор забыл настроить **DLP rules** для SharePoint Online. Сотрудник публикует Excel-файл с данными клиентов (PII) через SharePoint, думая что это внутреннее. Netskope не блокирует — нет правила. Данные 50K клиентов доступны через share link. **Урок:** SASE — фреймворк, не silver bullet. Политики должны покрывать все векторы exfiltration.

## Часть 3. Техническая глубина: Как устроены SD-WAN и SASE

### SD-WAN: Программное управление WAN-маршрутизацией
**Аналогия:** Это как GPS-навигатор для трафика между офисами, который автоматически переключается между платными дорогами (MPLS), федеральными трассами (broadband) и просёлочными (4G) в зависимости от пробок и типа груза.

**Архитектура SD-WAN:**
1. **Control plane**: Центральный контроллер (Viptela vManage, VeloCloud Orchestrator) управляет политиками.
2. **Data plane**: Edge-устройства (vEdge, SD-WAN Edge) выполняют маршрутизацию.
3. **Management plane**: GUI/API для конфигурации и мониторинга.

**Security в SD-WAN:**
- **Overlay IPSec**: Все туннели между edge-устройствами зашифрованы (обычно IPsec IKEv2 или WireGuard)
- **Segmentation**: VPN-like segment (VLAN-like в SD-WAN overlay)
- **Firewall на edge**: Stateful/NGFW правила на edge-устройствах
- **PKI для аутентификации**: Сертификаты, не pre-shared keys

| SD-WAN vendor | Протокол | Шифрование | Особенности |
|--------------|----------|------------|-------------|
| Cisco (Viptela) | TLOC | IPsec IKEv2 + AES-256-GCM | Segment routing, firewall, UTD (Snort IPS) |
| VMware (VeloCloud) | DMPO | IPsec IKEv2 | Dynamic MultiPath Optimization, облачный оркестратор |
| Fortinet Secure SD-WAN | — | IPsec + ASIC acceleration | Встроенный NGFW + SD-WAN |
| Silver Peak (Aruba) | — | IPsec | Фокус на WAN optimization |

**Проблемы SD-WAN:**
1. **Centralized controller = single point of compromise** — если vManage скомпрометирован, все edge-устройства переконфигурированы.
2. **Overlay bypasses underlay policies** — Overlay сеть может игнорировать физическую топологию и её ограничения.
3. **Key management** — SD-WAN разворачивает сотни IPsec-туннелей, управление ключами в масштабе — нетривиально.

### SASE: Безопасность как облачный сервис
**SASE** объединяет несколько security функций в облачную доставку:
- **SWG** (Secure Web Gateway): Фильтрация веб-трафика, блокировка вредоносных URL
- **CASB** (Cloud Access Security Broker): Защита SaaS (Office 365, Salesforce, Google Workspace)
- **ZTNA** (Zero Trust Network Access): Доступ к приложениям без VPN
- **FWaaS** (Firewall as a Service): Облачный NGFW
- **DLP** (Data Loss Prevention): Предотвращение утечки данных

**Архитектура SASE:**
- **PoP** (Point of Presence): Сеть из десятков/сотен дата-центров провайдера
- **Client connector**: Агент на endpoint (Zscaler Client Connector, Netskope Client) или туннель (GRE, IPsec)
- **Policy enforcement**: Все политики применяются в PoP перед отправкой в интернет/приложение

| SASE provider | Компоненты | PoP coverage | Особенности |
|--------------|------------|--------------|-------------|
| Zscaler | ZIA + ZPA + ZDX | 150+ глобально | Лидер, mature platform |
| Netskope | SWG + CASB + ZTNA | 60+ | Сильный CASB (API-based + inline) |
| Cloudflare (One) | SWG + ZTNA + FWaaS + DNS | 300+ | Собственная сеть, интеграция с Workers |
| Palo Alto (Prisma Access) | SWG + CASB + FWaaS + ZTNA | 100+ | Интеграция с NGFW и Cortex |

**Проблемы SASE:**
1. **Trust boundary shift**: Вы доверяете SASE-провайдеру весь свой трафик — включая TLS-расшифровку, инспекцию, логирование.
2. **Latency**: Каждый пакет идёт через PoP, который может быть за 1000 км. Для latency-sensitive приложений — критично.
3. **Vendor lock-in**: Переход с Zscaler на Netskope — не просто смена URL. Агенты, политики, логи — всё другое.
4. **SSL Inspection at scale**: SASE должен терминейт TLS, инспектировать, re-encrypt для каждого соединения. 10K пользователей × 100 TCP connections = 1M SSL терминаций.

## Часть 4. Архитектура защиты: Defense in Depth для SD-WAN/SASE

### Многоуровневая защита SD-WAN/SASE

| Уровень | SD-WAN | SASE | Стандарт |
|---------|--------|------|----------|
| Edge | IPsec IKEv2 + PKI | — | RFC 7296 |
| Edge | Segmentation (overlay VPN) | — | Policy-based routing |
| Cloud | — | SWG (web filtering) | NIST SP 800-41 |
| Cloud | — | CASB (SaaS protection) | NIST SP 800-210 |
| Cloud | — | ZTNA (application access) | NIST SP 800-207 |
| Cloud | — | FWaaS (cloud firewall) | PCI DSS, SOC 2 |
| Endpoint | Agent-based steering | Client connector agent | — |
| Endpoint | Host-based firewall | Host-based firewall | CIS Controls |
| Identity | Certificate-based auth | IdP integration (SAML/OIDC) | NIST SP 800-63 |

### Мониторинг и метрики
**Ключевые метрики безопасности SD-WAN/SASE:**
- **Tunnel uptime**: Все IPsec-туннели активны
- **Certificate expiry**: Сертификаты edge-устройств и PoP не истекают
- **SSL inspection rate**: % TLS-трафика, который инспектируется
- **Policy coverage**: % трафика, защищённого DLP/CASB/SWG правилами
- **ZTNA adoption vs VPN**: Миграция с VPN на ZTNA
- **SaaS shadow IT**: Приложения, обнаруженные CASB, которых нет в approved list
- **DLP incidents**: Количество блокированных попыток эксфильтрации

### Реальные инциденты

**Инцидент 1: SD-WAN controller compromise (2020)**
Исследователи обнаружили, что **Cisco SD-WAN** (Viptela) имел уязвимость **CVE-2021-1295** — удалённое выполнение кода в vManage через неаутентифицированный REST API. Злоумышленник с доступом к интернету мог получить shell на контроллере, затем сконфигурировать edge-устройства для перенаправления трафика. **Урок:** Центральный контроллер SD-WAN = high-value target. Защита: изолировать контроллер за firewall, патчить немедленно, ограничить API доступ, мониторить аномальные конфигурации.

**Инцидент 2: SASE SSL Inspection gap (2023)**
Организация использовала **Zscaler ZIA** для инспекции TLS-трафика. Но обнаружилось, что **DoH** (DNS over HTTPS) и **ECH** (Encrypted Client Hello) трафик проходил без инспекции — Zscaler не мог расшифровать, потому что SNI был скрыт, и DNS-запросы были внутри HTTPS. Вредоносное ПО в сети использовало DoH для C2 коммуникации, обходя инспекцию. **Урок:** SASE — не full visibility. ECH, DoH, QUIC, HTTP/3 — протоколы, которые создают gaps. Защита: мониторинг bypass attempts, блокировка неизвестных протоколов, layered defense (endpoint detection дополнительно к сетевой).

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"SD-WAN автоматически безопасен из-за IPsec"** — IPsec в SD-WAN шифрует туннели, но: (1) не проверяет содержимое; (2) не защищает от ошибок конфигурации (PSK вместо PKI); (3) не предотвращает lateral movement внутри overlay. SD-WAN = сеть, не безопасность.
2. **"SASE заменяет все on-premise security"** — SASE отлично для удалённых работников и облачных приложений, но: (1) не заменяет OT/ICS firewall на фабрике; (2) не заменяет endpoint security (AV/EDR); (3) не заменяет database security. SASE — один слой, defense in depth всё ещё нужен.
3. **"SASE провайдер уже всё правильно настроит"** — Shared responsibility: провайдер даёт платформу, вы настраиваете политики. Misconfiguration (слабые DLP правила, неполный SSL inspection, незащищённые ZTNA сегменты) — ваша ответственность.

### Три ситуационных вопроса для Senior+

**Q1:** Как защитить SD-WAN deployment, если оркестратор находится в облаке провайдера?
**A:** (1) **MFA на управление**: Доступ к контроллеру только через SAML/OIDC с MFA; (2) **API security**: API ключи с ограниченным сроком, scope-based, rate limiting; (3) **Audit logging**: Все изменения конфигурации логируются, алертуют на неожиданные изменения; (4) **PKI для edge**: Сертификаты (S/MIME, X.509) для аутентификации edge-устройств, не PSK; (5) **Segmented management**: Контроллер в отдельном VLAN, доступ через jump host с session recording. Принцип: контроллер = crown jewel, многоуровневая защита доступа к нему.

**Q2:** Почему SASE сложно мониторить, и как решить эту проблему?
**A:** **Трудности:** SASE создаёт огромный объём логов (10K пользователей × 100M events/day). Логи распределены по разным PoP и вендору. Нет единого формата. **Решение:** (1) **SIEM integration** (Splunk, Azure Sentinel) — централизованный сбор логов из SASE provider; (2) **SASE-native analytics** — Zscaler ZIA Insights, Netskope Advanced Analytics; (3) **Custom dashboards** — Grafana + API polling SASE provider; (4) **Correlation** — объединение SASE логов с endpoint (CrowdStrike) и identity (Azure AD) для full picture. Ключевое: SASE даёт видимость, но инструмент нужен для её осмысления.

**Q3:** Как мигрировать с традиционного VPN на ZTNA в рамках SASE?
**A:** (1) **Inventory**: Список всех приложений, к которым VPN даёт доступ; (2) **Pilot**: Выбрать 1-2 cloud-приложения (Office 365, Jira) — перенести на ZTNA, отключить VPN-доступ; (3) **ZTNA connector**: Развернуть connector в сети (Zscaler ZPA connector) для internal apps; (4) **IdP integration**: ZTNA использует identity из Azure AD/Okta — настроить SSO; (5) **Phased sunset**: VPN туннель остаётся для legacy (мейнфреймы, OT), но всё новое — через ZTNA; (6) **User communication**: "VPN replaced with better experience, same security". Потери: user friction (нужен новый агент), но ZTNA быстрее VPN и не требует "включить VPN перед работой".

### Чек-лист защиты SD-WAN/SASE (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | SD-WAN использует PKI (сертификаты), не PSK для туннелей? | Сертификаты от enterprise CA | Обязательно |
| 2 | SD-WAN контроллер защищён MFA и сегментирован? | Access control audit controller | Обязательно |
| 3 | SASE SSL inspection покрывает 100% TLS-трафика? | Zscaler SSL inspection logs | Высокая |
| 4 | DLP правила для SaaS (Office 365, Salesforce) активированы? | Netskope CASB DLP policies | Высокая |
| 5 | ZTNA заменяет VPN для >=80% приложений? | User access method tracking | Высокая |
| 6 | Shadow IT (неавторизованные SaaS) детектируется CASB? | Netskope CASB Cloud Registry | Высокая |
| 7 | Логи SASE направлены в SIEM (корреляция с другими источниками)? | Splunk/Azure Sentinel integration | Высокая |
| 8 | ECH / DoH / QUIC blocked или inspected? | Protocol control в SASE | Средняя |
| 9 | SASE provider disaster recovery протестирован? | Multi-PoP failover test | Средняя |
| 10 | VPN kill switch при отсутствии ZTNA для legacy? | Policy для VPN-пользователей | Средняя |

### Key Takeaways
1. **SD-WAN ≠ security** — виртуализация WAN-маршрутизации не делает её безопасной. PKI, segmentation, monitoring всё ещё нужны.
2. **SASE — архитектурный сдвиг** — безопасность доставляется из облака, trust boundary расширен. Выбирать provider wisely.
3. **Shared responsibility** — провайдер даёт платформу, вы настраиваете политики. Misconfiguration — ваш риск.
4. **SSL Inspection — cornerstone** — без него SASE не видит шифрованный C2 и эксфильтрацию. Новые протоколы (ECH, DoH) создают gaps.
5. **ZTNA > VPN** — для облачных приложений, identity-based access надёжнее IP-based VPN. Legacy всё ещё нуждается в туннелях.
