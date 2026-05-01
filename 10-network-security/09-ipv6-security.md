# Безопасность IPv6: Расширенный attack surface под новыми адресами

## Часть 1. Знакомство с проблемой: Почему IPv6 — это новый город с невидимыми жителями?

Представьте, что вы переехали из маленького городка (всего 4 миллиарда адресов — хватало едва-едва) в мегаполис размером с галактику. Каждому устройству, каждому датчику, каждой лампочке можно дать публичный адрес — адресов хватит на каждый атом на Земле. Но город построили быстро, и охрана ещё не расставлена. Камеры не везде работают, заборы не достроены.

**IPv6** был создан, чтобы решить проблему исчерпания IPv4-адресов. Вместо 32 бит (4.3 миллиарда адресов) — 128 бит (340 ундециллионов адресов). Но с огромным адресным пространством пришёл и расширенный attack surface. **Neighbor Discovery Protocol (NDP)** заменил ARP, и это не улучшение безопасности — это смена вектора атак. **IPv6 fragmentation** работает иначе. **Transition mechanisms** (6to4, Teredo, ISATAP) создают мосты между IPv4 и IPv6, и каждый мост — это обход firewall.

В 2026 году IPv6 adoption >40% глобально, но IPv6 security awareness — <20%. **IPv4-only firewall** не видит IPv6-трафик. **Dual-stack** означает, что одно устройство имеет два адреса, два стека, и два attack surface.

| Изменение в IPv6 | Что ушло из IPv4 | Новая угроза | Почему это проблема |
|-----------------|-----------------|--------------|---------------------|
| 128-bit addresses | 32-bit addresses | Scanning сложнее, но возможен | Слишком большой range для brute-force, но enumeration techniques существуют |
| NDP вместо ARP | ARP | NDP spoofing (аналог ARP-spoofing) | NDP не имеет встроенной аутентификации |
| SLAAC (автоконфигурация) | DHCP | Rogue RA (Router Advertisement) | Любой хост может объявить себя маршрутизатором |
| Fragmentation в IPv6 | Fragmentation в IPv4 | Atomic fragments, overlapping fragments | Другой механизм, новые уязвимости |
| Extension headers | Options в IP header | DoS через бесконечную цепочку заголовков | Не все firewall/IDS могут парсить extension headers |

**Эволюция:** В 1998 **IPv6** стандартизирован (RFC 2460). В 2011 **IANA** исчерпал IPv4-адреса. В 2012 **World IPv6 Launch** — Google, Facebook, Yahoo включили IPv6. В 2017 **SLAAC** privacy extensions (RFC 7217) — стабильные, но не предсказуемые адреса. В 2019 **RFC 8504** — требования безопасности IPv6 для узлов. В 2024 **IPv6-only** дата-центры (Facebook, Google) — экономия на NAT и IPv4-менеджменте, но полностью новый threat model.

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Корпоративная сеть и rogue RA
Организация развернула IPv6 dual-stack на всех endpoint'ах. Сотрудник подключает личный роутер с IPv6 к корпоративной розетке для расширения Wi-Fi. Роутер отправляет **Router Advertisement (RA)** с параметрами: "Я — шлюз, мой prefix — 2001:db8:1::/64, DNS — 2001:db8:1::53". Windows/Linux клиенты принимают RA и перенаправляют трафик через личный роутер вместо корпоративного firewall. Злоумышленник слушает весь трафик отдела. **Урок:** RA guard на свитчах обязателен.

### Сценарий 2: Дата-центр и сканирование IPv6
Клауд-провайдер назначил `/120` (256 адресов) для каждого арендатора в VPC. Один арендатор начинает сканировать свою сеть IPv6. Но из-за ошибки в routing, трафик к соседнему `/120` тоже проходит. Сканирование находит открытые порты (SSH, базы данных) в соседних подсетях. Злоумышленник атакует их и получает доступ к гипервизору. **Урок:** IPv6 микросегментация требует ACL/firewall, не только маршрутизации.

### Сценарий 3: IPv6-only инфраструктура и transition mechanism bypass
Организация использует IPv6-only внутреннюю сеть и **NAT64** для доступа к IPv4-интернету. Аудитор проверяет firewall: ACL на IPv4 есть, на IPv6 — нет ("ну мы же IPv6-only внутри, зачем firewall?"). Злоумышленник обходит NAT64 (запросы извне IPv6 → IPv4) и получает доступ к внутренним серверам. **Урок:** Firewall нужен на обоих стеках.

## Часть 3. Техническая глубина: Как устроены угрозы IPv6

### Neighbor Discovery Protocol (NDP): Ахиллесова пята IPv6
**Аналогия:** Это как если бы в новом городе не было телефонной книги. Каждый житель должен сам выкрикивать "Я здесь, мой адрес — такой-то!", и любой может выкрикнуть "Я — мэр, присылайте налоги мне!".

**NDP** выполняет функции ARP + часть ICMP в IPv4. Использует **ICMPv6** сообщения:
- **Neighbor Solicitation (NS)**: "Кто имеет адрес X? Сообщи мне свой MAC" (аналог ARP request)
- **Neighbor Advertisement (NA)**: "Адрес X имеет MAC Y" (ARP reply)
- **Router Solicitation (RS)**: "Есть ли тут маршрутизатор?"
- **Router Advertisement (RA)**: "Я — маршрутизатор, используйте prefix P, DNS D"

**Уязвимости NDP:**
- **NDP spoofing**: Аналог ARP-spoofing — отправка поддельного NA, перехват трафика
- **Rogue RA**: Отправка поддельного RA, перенаправление трафика или DoS (неправильный префикс)
- **Address Resolution DoS**: Лавина NS = CPU saturation на цели

| Атака NDP | IPv4-эквивалент | Механизм | Защита |
|-----------|----------------|----------|--------|
| NDP spoofing | ARP spoofing | Поддельный NA с правильным IP и чужим MAC | SeND (не внедрён), DAI-style inspection |
| Rogue RA | DHCP spoofing | Поддельный RA с ложным роутером/prefix | RA Guard (RFC 6105) |
| NS/NA flood | ARP flood | Лавина NS = исчерпание NDP cache | Rate limiting, ND table limits |
| Duplicate Address Detection (DAD) DoS | — | Блокировка всех адресов как "уже используемых" | DAD с rate limiting |

### SLAAC: Автоконфигурация и privacy implications
**SLAAC** (Stateless Address Autoconfiguration) — хост автоматически получает IPv6-адрес без DHCP-сервера. Получает **prefix** из RA и генерирует **interface identifier** (IID) на основе MAC-адреса (EUI-64) или случайно (privacy extensions).

**Проблемы SLAAC:**
1. **EUI-64 раскрывает MAC-адрес**: Идентификатор интерфейса = `ff:fe` + MAC. Злоумышленник знает MAC вашего устройства, может отслеживать его в других сетях.
2. **Privacy extensions не панацея**: Случайный IID меняется каждые 24 часа, но стабильный IID (RFC 7217) предсказуем для данной сети.
3. **IPv6 address scanning сложнее, но не невозможен**: /64 имеет 18.4 квинтиллионов адресов — не просканируешь brute-force. Но серверы имеют известные адреса (::1, ::53, ::443), а DHCPv6/DNS выдают записи.

### IPv6 Fragmentation: От atomic до overlapping
В IPv6 **фрагментация происходит только на источнике** (не на маршрутизаторах, как в IPv4). Используется **Fragment Header** (extension header #44). **Atomic fragments** — пакеты с Fragment Header, но без реальной фрагментации (M=0, Offset=0). **Overlapping fragments** — второй фрагмент накладывается на первый, заменяя данные.

**Проблемы:**
- **Atomic fragments trick firewall**: Firewall видит Fragment Header, может пропустить без глубокой инспекции
- **Overlapping fragments**: NIDS видит одни данные, хост видит другие — bypass сигнатур
- **Fragmented NDP**: Атака NDP через фрагментированные пакеты — сложнее обнаружение

**Защита:**
- **Запретить atomic fragments**: RFC 6946 (IPv6 atomic fragments considered harmful)
- **Reassembly before inspection**: Firewall должен собрать пакет перед проверкой (дорого по ресурсам)
- **Minimize fragment acceptance**: Хост должен принимать минимум фрагментов

### Transition Mechanisms: Почему dual-stack — это двойные проблемы
**Dual-stack**, **6to4**, **Teredo**, **ISATAP** — всё это мосты между IPv4 и IPv6. Каждый мост создаёт путь, который может обходить IPv4-only firewall.

**Teredo** инкапсулирует IPv6 внутри UDP-IPv4 (туннелирование через NAT). **6to4** — автоматическое туннелирование IPv6 через IPv4 (2002::/16). **ISATAP** — IPv6 внутри IPv4 внутри организации.

| Transition mechanism | Как работает | Угроза | Защита |
|---------------------|-------------|--------|--------|
| Dual-stack | Два стека на одном интерфейсе | Двойной attack surface, firewall на одном стеке не видит другой | ACL/firewall на обоих стеках |
| 6to4 | 2002::/16 туннелирование | Трафик обходит NAT/firewall | Блокировать 6to4 на границе |
| Teredo | UDP-туннель через NAT | Трафик внутри UDP, выглядит как обычный | DPI, блокировать Teredo на endpoint'ах |
| ISATAP | Внутренний туннель | Обход внутреннего firewall | Отключить ISATAP, использовать native IPv6 |

## Часть 4. Архитектура защиты: Defense in Depth для IPv6

### Многоуровневая защита IPv6

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| Router | RA Guard | RFC 6105 | Cisco "ipv6 nd raguard" |
| Router | ND Inspection | — | Cisco "ipv6 nd inspection" |
| Router | IPv6 ACL (аналог IPv4) | — | `ipv6 access-list` на интерфейсах |
| Router | IPv6 source guard | RFC 7039 | Проверка source IPv6 и MAC (binding table) |
| Firewall | IPv6 stateful inspection | — | NGFW с полной поддержкой IPv6 |
| Firewall | Extension header filtering | RFC 9098 | Блокировать вредоносные extension headers |
| Endpoint | IPv6 host firewall | — | Windows Firewall, iptables6 |
| Endpoint | SLAAC privacy extensions | RFC 8981 | Temporary IID rotation |
| Endpoint | SeND (Secure Neighbor Discovery) | RFC 3971 | Криптографическая защита NDP (редко внедрён) |

### Мониторинг и метрики
**Ключевые метрики безопасности IPv6:**
- **IPv6 traffic ratio**: % IPv6 vs IPv4 трафика — должен соответствовать ожидаемому
- **RA frequency**: Слишком частые RA = возможная rogue RA атака
- **NDP cache churn**: Частые изменения = возможный NDP spoofing
- **Extension header distribution**: Необычные extension headers = возможная атака
- **Fragmented traffic ratio**: Необычно высокая фрагментация = evasion attempt
- **Transition mechanism traffic**: Трафик через 6to4/Teredo = потенциальный обход

### Реальные инциденты

**Инцидент 1: IPv6 DDoS на CloudFlare (2014)**
**CloudFlare** обнаружил, что IPv6-трафик может быть использован для **amplification DDoS** через **ICMPv6** ошибки. Атакующий отправляет IPv6 пакет с поддельным source, маршрутизатор возвращает ICMPv6 Destination Unreachable на жертву. Amplification factor: 1:1 (не огромный, но это новый вектор, который не ожидался). **Урок:** IPv6 добавляет новые amplification-векторы. Защита: ICMPv6 rate limiting, uRPF для IPv6.

**Инцидент 2: VPN bypass через IPv6 (2023)**
Корпоративный VPN (Cisco AnyConnect) настроен для защиты IPv4-трафика. IPv6 на endpoint'ах был включён, но VPN-клиент не поддерживал IPv6 инкапсуляцию. Результат: **IPv6-трафик шёл напрямую**, обходя VPN-туннель. DNS-запросы IPv6, веб-доступ к IPv6-сайтам — всё утекало. VPN не защищал от IPv6-утечки. **Урок:** Проверять поддержку IPv6 в VPN-клиенте. Принудительно включать IPv6 через VPN или отключать IPv6 на endpoint'е при отсутствии поддержки (kill switch).

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"IPv6 более безопасен, чем IPv4"** — IPv6 имеет встроенный **IPsec** как "обязательный" (по спецификации), но это опционально и не внедряется. В реальности, IPv6 добавляет новые протоколы (NDP, SLAAC, extension headers) с новыми уязвимостями. Security is a process, не встроенное свойство.
2. **"IPv6 адреса невозможно просканировать"** — `/64` имеет 2^64 адресов — bruteforce невозможен. Но: серверы используют известные адреса (::1, ::80, ::443), DNS выдаёт все записи, DHCPv6 знает все leased адреса, multicast groups enumerable. Scanning изменился — он больше не bruteforce, а enumeration.
3. **"Dual-stack — это простой переходный этап"** — Dual-stack удваивает attack surface. Каждое устройство имеет два IP-адреса, два стека с правилами. Firewall на IPv4 не видит IPv6-трафик. Конфигурация должна быть симметричной.

### Три ситуационных вопроса для Senior+

**Q1:** Как предотвратить rogue RA в корпоративной сети?
**A:** (1) **RA Guard** (RFC 6105) на свитчах — разрешает RA только от доверенных портов/источников; (2) **ND Inspection** — проверяет NS/NA сообщения через binding table (аналог DAI для IPv6); (3) **IPv6 Snooping** — изучение IPv6-адресов из DAD (Duplicate Address Detection) для создания binding table; (4) **Access Lists** — блокировать RA на портах, где нет маршрутизаторов; (5) **SeND** — криптографическая защита NDP, но практически не внедрён. RA Guard — минимальное обязательное.

**Q2:** Как защитить IPv6-сеть, если часть инфраструктуры не поддерживает IPv6-фильтрацию (legacy firewall)?
**A:** (1) **Tunnel all IPv6 через современный firewall** — перенаправить IPv6-трафик через GRE/IPSec туннель к современному NGFW; (2) **Host-based firewall** (iptables6, Windows Firewall) — блокировать нежелательный трафик на endpoint; (3) **Isolated IPv6 segment** — выделить IPv6 в отдельный VLAN с собственным firewall; (4) **Отключить IPv6 на legacy сегменте** — если IPv6 не нужен, отключить совсем; (5) **NAT66** (хотя и противоречит философии IPv6) — скрыть внутреннюю топологию, как NAT в IPv4. Лучшее — обновить инфраструктуру, но временные меры возможны.

**Q3:** Как обнаружить IPv6-трафик, обходящий корпоративный VPN?
**A:** (1) **Monitor IPv6 traffic patterns at endpoint** — если endpoint делает IPv6-запросы напрямую, это видно в netflow/sflow; (2) **DNS analysis** — если endpoint делает AAAA-запросы (IPv6 DNS) во время VPN-сессии, но A-запросы (IPv4) через VPN; (3) **VPN client configuration audit** — проверить, что VPN-клиент конфигурируется для IPv6 (AnyConnect имеет "IPv6 drop all" вне туннеля); (4) **Browser-based detection** — WebRTC leak check (публичный тест). Решение: VPN kill switch для IPv6 (force all IPv6 через туннель или блокировать при отсутствии туннеля).

### Чек-лист защиты IPv6 (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | RA Guard включён на всех access-свитчах? | `ipv6 nd raguard` на интерфейсах | Обязательно |
| 2 | IPv6 ACL симметричны с IPv4 ACL (firewall)? | Stateful firewall rules для IPv6 | Обязательно |
| 3 | Transition mechanisms (6to4, Teredo, ISATAP) заблокированы? | ACL на границе, отключены на endpoint | Обязательно |
| 4 | Privacy extensions для SLAAC включены? | `net.ipv6.conf.all.use_tempaddr = 2` | Высокая |
| 5 | Extension headers фильтруются (атомные фрагменты запрещены)? | Firewall rule для Fragment Header без данных | Высокая |
| 6 | IPv6 NDP spoofing detection (ND Inspection) настроено? | `ipv6 nd inspection` | Высокая |
| 7 | IPv6 трафик логируется и мониторится (аналогично IPv4)? | Netflow, sFlow для IPv6 | Высокая |
| 8 | VPN клиент проверен на IPv6 leak (kill switch)? | Внешний IPv6 leak test | Высокая |
| 9 | IPv6 на endpoint отключён, если не используется? | GPO, MDM для отключения IPv6 | Средняя |
| 10 | IPv6-specific scanning/penetration test проведён? | Сканирование IPv6 (не только IPv4) | Высокая |

### Key Takeaways
1. **IPv6 ≠ безопаснее IPv4** — новый протокол = новые уязвимости. NDP, SLAAC, extension headers — все векторы атак.
2. **Dual-stack = двойной attack surface** — firewall на IPv4 не видит IPv6. Конфигурация должна быть симметричной.
3. **RA Guard обязателен** — rogue RA = компрометация маршрутизации всей подсети.
4. **Transition mechanisms = обходы** — 6to4, Teredo, ISATAP создают пути мимо firewall. Блокировать.
5. **Scanning изменился, не исчез** — enumeration techniques заменяют bruteforce. DNS, DHCPv6, multicast = источники адресов.
