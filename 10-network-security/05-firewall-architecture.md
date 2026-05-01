# Архитектура Firewall: От простого шлюза до интеллектуального периметра

## Часть 1. Знакомство с проблемой: Почему firewall — это охранник, который знает не всех гостей?

Представьте себе охранника у входа в здание. Его инструкция проста: "Пропускай только тех, у кого есть бейдж". Это **stateless firewall** — проверяет каждый пакет отдельно, без контекста. Теперь представьте охранника, который ведёт журнал: "Этот человек вошёл в 9:00, значит, он может выйти в 18:00". Это **stateful firewall** — отслеживает состояние соединений. Наконец, представьте охранника с камерами, базой данных лиц, анализом поведения и связью с полицией. Это **NGFW** (Next-Generation Firewall) — проверяет не только кто, но и что делает.

**Firewall** — первый и последний рубеж сетевой безопасности. Он фильтрует трафик на основе правил, но правила — лишь набор допущений о том, что "хорошо" и что "плохо". **Stateful inspection** отслеживает TCP-сессии, но не понимает содержимое HTTP-запроса. **NGFW** инспектирует приложения, но требует расшифровки TLS — что создаёт новые риски. **WAF** (Web Application Firewall) защищает веб-приложения, но может быть обойдён через полиморфные атаки.

В 2026 году firewall-ландшафт гибридный: **on-premise NGFW** (Palo Alto, Fortinet), **cloud-native firewalls** (AWS Security Groups, Azure NSG, GCP VPC Firewall), **distributed firewalls** (Cilium, Calico в Kubernetes), и **SASE**-решения, где firewall — облачный сервис. Каждый тип имеет свой threat model, свои ограничения, и свои точки отказа.

| Поколение firewall | Принцип | Что проверяет | Что не видит |
|-------------------|---------|---------------|--------------|
| 1-е: Packet filter | Stateless | IP, port, protocol | Состояние соединения, payload |
| 2-е: Stateful | Connection tracking | IP, port + state table | Содержимое приложения |
| 3-е: Application (Proxy) | Application-layer | HTTP, FTP, SMTP commands | Encryption (TLS), zero-days |
| 4-е: NGFW | Deep packet inspection | Applications, users, threats | Encrypted payload без MITM |
| 5-е: AI/ML-driven | Behavioral analysis | Аномалии, patterns | Adversarial ML, mimicry |

**Эволюция:** В 1988 **DEC SEAL** — первый commercial firewall. В 1993 **Stateful inspection** в Check Point FireWall-1. В 2004 **Palo Alto Networks** запустила **NGFW** с App-ID (идентификация приложений независимо от порта). В 2014 **WAF** стал обязательным для PCI DSS. В 2019 **Gartner** ввёл термин **SASE** (Secure Access Service Edge) — конвергенция SD-WAN и cloud-native security. В 2026 **AI-driven firewalls** используют behavioral analysis для zero-day detection.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Электронная коммерция и WAF bypass
Интернет-магазин использует **mod_security WAF** для защиты от SQL injection. Правило: блокировать `union select`. Злоумышленник отправляет: `UNI%6Fn SEL%45CT` — URL-encoded. WAF декодирует один раз, видит `UNI%6Fn`, не срабатывает. Backend декодирует второй раз, получает `union select`. SQL injection проходит. Потери: база клиентов (2M записей) сливается за 3 дня. **Урок:** WAF — не замена secure coding.

### Сценарий 2: Облачная инфраструктура и security group misconfiguration
DevOps-инженер открывает **AWS Security Group** для тестирования: порт 0-65535 открыт для "0.0.0.0/0". Забывает закрыть. Злоумышленник сканирует EC2, находит **ElastiCache Redis** на порту 6379 без пароля, извлекает session tokens, получает доступ к **RDS PostgreSQL** через те же security group rules. База клиентов — 10M записей. Урок: Cloud firewalls — просты в настройке, просты в misconfiguration.

### Сценарий 3: Производственная сеть и OT firewall bypass
Фабрика использует **Purdue Model**: Level 0 (sensors) → Level 1 (PLCs) → Level 2 (SCADA) → Level 3 (MES) → Level 4 (ERP). **OT firewall** разделяет Level 2 и Level 3. Но инженер для удобства настраивает **NAT rule**: любой запрос из Level 3 перенаправляется на SCADA-сервер в Level 2. Злоумышленник из корпоративной сети (Level 4) использует этот NAT для доступа к PLC. Меняет параметры производственного процесса — брак на $1M за сутки. **Урок:** Firewall rules — не просто конфигурация, это security architecture.

## Часть 3. Техническая глубина: Как устроены современные firewalls

### Stateful Inspection: Журнал соединений
**Аналогия:** Это как если бы охранник вёл журнал: "Кому я открыл дверь в 9:00 — тому я доверяю выйти в 18:00 без повторной проверки документов".

**Stateful firewall** отслеживает **connection state table** — список всех активных соединений. Когда приходит TCP SYN, firewall создаёт запись: `src IP:port, dst IP:port, state=SYN_SENT`. После SYN-ACK: `state=ESTABLISHED`. FIN/ACK: `state=CLOSING`. Таймаут: запись удаляется.

**Проблемы stateful inspection:**
1. **State table exhaustion** — DDoS отправляет миллионы SYN, таблица переполняется. Решение: **SYN-proxy**, **SYN-cookies**, rate limiting.
2. **Asymmetric routing** — если ответы приходят через другой путь, firewall не видит SYN-ACK и дропает пакеты. Решение: **state synchronization** между firewall'ами.
3. **Protocol complexity** — FTP, SIP, H.323 требуют **ALG** (Application Layer Gateway) для отслеживания динамических портов. ALG — источник уязвимостей.
4. **Encrypted traffic** — firewall видит IP/port, но не payload (если TLS). Решение: **SSL inspection** (MITM с собственным CA) — но это создаёт privacy и legal риски.

| Тип inspection | Что проверяет | Производительность | Безопасность |
|---------------|---------------|-------------------|--------------|
| Stateless ACL | IP, port, protocol | Высокая | Низкая |
| Stateful | Connection state | Высокая | Средняя |
| Deep Packet Inspection (DPI) | Payload, signatures | Средняя | Высокая |
| Application-aware | App protocol, commands | Низкая | Высокая |
| Behavioral / ML | Аномалии, baseline | Средняя | Очень высокая |

### NGFW: Application-ID, User-ID, Content-ID
**NGFW** (Next-Generation Firewall) идёт за пределы порта/протокола:
- **App-ID**: Идентифицирует приложение независимо от порта (Skype через 443, Tor через 80).
- **User-ID**: Связывает IP-адрес с Active Directory username — правила вида "разрешить Маркетингу, запретить HR".
- **Content-ID**: Инспектирует payload — Data Loss Prevention (DLP), фильтрация файлов, threat prevention.
- **SSL Inspection**: Terminates TLS, инспектирует, re-encrypts — требует CA-сертификата на клиентах.

**Проблемы NGFW:**
1. **SSL Inspection breaks things**: Certificate pinning, HSTS, certificate transparency — всё ломается.
2. **Performance impact**: DPI на 10 Gbps требует dedicated hardware (FPGA, ASIC).
3. **False positives**: App-ID может ошибочно идентифицировать легитимный трафик как вредоносный.
4. **Centralization**: NGFW — single point of failure и bottleneck.

### WAF: Защита веб-приложений и его ограничения
**WAF** (Web Application Firewall) работает на уровне HTTP и защищает от OWASP Top 10: SQLi, XSS, CSRF, LFI/RFI, injection. Работает через **negative security model** (блокировать известные плохие паттерны) или **positive security model** (разрешать только известные хорошие паттерны).

**Обход WAF:**
- **Encoding**: URL encoding, Unicode, HTML entities, base64 — WAF декодирует не всё.
- **Comment injection**: `UN/**/ION SEL/**/ECT` — комментарии разрывают сигнатуры.
- **HTTP Parameter Pollution**: Повторение параметров с разными значениями — backend берёт первый, WAF — последний.
- **Chunked transfer encoding**: Разбиение payload на chunks — WAF может не собрать полное тело.
- **Polymorphic attacks**: Меняющиеся сигнатуры, генерируемые автоматически.

| Тип WAF | Модель | Точность | Сложность |
|---------|--------|----------|-----------|
| Negative (blacklist) | Блокировать известные атаки | Низкая (false negatives) | Низкая |
| Positive (whitelist) | Разрешать только валидные запросы | Высокая (false positives) | Высокая |
| Hybrid | Комбинация | Средняя | Средняя |
| ML-based | Аномалии на основе обучения | Высокая (с обучением) | Очень высокая |

### Cloud-Native Firewalls: Security Groups, NACLs, Network Policies
**AWS Security Groups** — stateful firewall на уровне инстанса. **NACL** (Network ACL) — stateless на уровне подсети. **VPC Firewall Rules** (GCP) — гибрид. **Azure NSG** — stateful.

**Kubernetes Network Policies** — L3/L4 firewall между подами. Реализуется через **Cilium** (eBPF), **Calico** (iptables/IPSet), или **Weave Net**. **Cilium** идёт дальше — **L7 policies** (HTTP-aware), **DNS-based policies**, **identity-based security** (не IP-based).

## Часть 4. Архитектура защиты: Defense in Depth для Firewall

### Многоуровневая защита Firewall

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| Приложение | WAF | OWASP CRS | ModSecurity, AWS WAF, Cloudflare |
| Приложение | API Gateway rate limiting | RFC 6585 | Kong, AWS API Gateway |
| Транспорт | NGFW SSL Inspection | — | Palo Alto, Fortinet, Check Point |
| Сеть | Stateful firewall | RFC 791 + extensions | Cisco ASA, pfSense, iptables |
| Сеть | Cloud security groups | AWS/Azure/GCP docs | AWS SG, Azure NSG, GCP VPC |
| Сеть | Microsegmentation | NIST SP 800-125B | VMware NSX, Cisco ACI |
| Endpoint | Host-based firewall | — | Windows Firewall, iptables, ufw |
| Container | Network Policies | Kubernetes docs | Cilium, Calico, Weave |

### Мониторинг и метрики
**Ключевые метрики безопасности firewall:**
- **Blocked vs allowed ratio**: Внезапное изменение = возможная misconfiguration
- **Top blocked signatures**: Часто срабатывающие = возможные false positives или persistent атака
- **SSL inspection rate**: % TLS-соединений, которые инспектируются
- **WAF false positive rate**: Легитимные запросы, заблокированные WAF
- **Firewall rule count**: >1000 правил = complexity debt, рост ошибок
- **Shadowed rules**: Правила, которые никогда не срабатывают (мертвый код в ACL)

### Реальные инциденты

**Инцидент 1: Target breach через HVAC (2013)**
Злоумышленники вошли в сеть **Target** через подрядчика HVAC, у которого был VPN-доступ. **Firewall разрешал** трафик из HVAC-сегмента в **POS-сегмент** (правило для удобства управления). Злоумышленники двигались lateral, украли данные 40M кредитных карт. **Урок:** Firewall правила — это security architecture, не operational convenience. Защита: микросегментация, default-deny, regular firewall audit, Zero Trust (не доверять даже "доверенным" вендорам).

**Инцидент 2: Capital One через WAF bypass (2019)**
**Capital One** использовала **AWS WAF** и **ModSecurity**. Злоумышленник использовал **Server-Side Request Forgery (SSRF)** через уязвимость в **EC2 metadata service** (IMDSv1). WAF не сработал, потому что SSRF — legitimate HTTP-запрос к внутреннему IP. Злоумышленник получил IAM credentials из metadata, обошёл firewall rules (они разрешали access from EC2 instances), слил 100M записей. **Урок:** WAF защищает от атак на приложение, не от misconfiguration инфраструктуры. Защита: IMDSv2 (session-based), security group audit, least privilege IAM.

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"Firewall защищает от всего"** — Firewall фильтрует трафик, но: (1) не защищает от XSS/SQLi (это WAF/secure coding); (2) не защищает от insider threats; (3) не защищает от encrypted C2. Firewall — один слой defense in depth.
2. **"NGFW с SSL inspection = полная видимость"** — SSL inspection ломает certificate pinning, создаёт legal/privacy риски, снижает производительность на 40-60%, и не работает для pinned certs. Кроме того, malware может использовать **ESNI/ECH** для скрытия SNI.
3. **"Cloud security groups — простые и безопасные"** — Security Groups легко misconfigure: "0.0.0.0/0" на SSH (22), открытые порты баз данных, слишком широкие CIDR. Cloud — shared responsibility: провайдер защищает облако, вы защищаете в облаке.

### Три ситуационных вопроса для Senior+

**Q1:** Как защититься от WAF bypass через encoding и obfuscation?
**A:** Defense in depth: (1) **Secure coding** — WAF не замена parameterized queries, output encoding; (2) **Positive security model** — разрешать только валидные паттерны (whitelist), не блокировать известные плохие (blacklist); (3) **Multiple decoding layers** — WAF должен декодировать URL, HTML, base64 перед проверкой; (4) **Request normalization** — canonicalize input before inspection; (5) **RASP** (Runtime Application Self-Protection) — защита на уровне приложения, не сети. WAF — perimeter defense, не единственная.

**Q2:** Почему stateful firewall может дропать legitimate трафик в asymmetric routing?
**A:** **Asymmetric routing**: запрос идёт через Firewall A, ответ — через Firewall B. Firewall B не видел SYN, поэтому у него нет записи в state table. Он дропает SYN-ACK как out-of-state. Решения: (1) **Symmetric routing** — настроить маршрутизацию; (2) **State synchronization** — firewall'ы обмениваются state table (Cisco ASA cluster, Palo Alto HA); (3) **Stateless ACL** для known asymmetric paths; (4) **Active-active** с session affinity. Ключевое правило: stateful inspection предполагает, что оба направления проходят через один firewall.

**Q3:** Как реализовать microsegmentation в Kubernetes с zero trust между подами?
**A:** (1) **Network Policies** — default-deny all ingress/egress, затем whitelist по namespace/podSelector; (2) **Cilium** — identity-based policies (не IP-based), L7-aware (HTTP-aware rules), DNS-based egress control; (3) **Service Mesh** — Istio/Linkerd добавляют mTLS + authorization policies на уровне сервиса; (4) **Admission controllers** — OPA/Gatekeeper запрещают поды без network policies; (5) **Egress gateway** — все outbound трафик через контролируемый прокси. Микросегментация в K8s = identity + intent, не IP + port.

### Чек-лист защиты Firewall (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | Default-deny policy на всех firewall'ах? | Нет правила "permit any any" | Обязательно |
| 2 | Stateful inspection на perimeter? | Connection tracking включён | Обязательно |
| 3 | WAF защищает все public-facing приложения? | AWS WAF, Cloudflare, или on-prem | Обязательно |
| 4 | SSL inspection настроен с минимальными breaks? | Certificate pinning whitelist, HSTS preserve | Высокая |
| 5 | Cloud security groups audited monthly? | AWS Config, SCP для запрета 0.0.0.0/22 | Обязательно |
| 6 | Microsegmentation внутри сети (не flat)? | VMware NSX, Cisco ACI, или host-based | Высокая |
| 7 | Kubernetes Network Policies default-deny? | `kubectl get networkpolicies --all-namespaces` | Высокая |
| 8 | Firewall rules reviewed quarterly? | Удаление shadowed, unused rules | Высокая |
| 9 | OT/ICS network segmented от IT? | Purdue Model, ISA-99, OT firewall | Обязательно |
| 10 | DDoS protection перед firewall (scrubbing)? | CloudFlare Magic Transit, AWS Shield | Высокая |

### Key Takeaways
1. **Firewall — не silver bullet** — perimeter defense, не application security. XSS/SQLi проходят через open ports.
2. **Stateful > stateless** — connection tracking даёт контекст, но требует symmetric routing.
3. **WAF не заменяет secure coding** — обход через encoding, HPP, chunked transfer — реальны.
4. **Cloud firewall'ы легко misconfigure** — "0.0.0.0/0" — чащее, чем хотелось бы. Regular audit обязателен.
5. **Microsegmentation = future** — от perimeter к identity-based, от IP-based к intent-based.
