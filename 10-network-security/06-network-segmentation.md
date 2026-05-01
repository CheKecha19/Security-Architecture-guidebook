# Сегментация сети: От VLAN до Zero Trust Microsegmentation

## Часть 1. Знакомство с проблемой: Почему плоская сеть — это город без районов?

Представьте город, где нет районов, улиц и адресов — просто бесконечное поле домов. Если в одном доме начнётся пожар, он мгновенно распространится на весь город. Если преступник войдёт в один дом, он может свободно ходить по всем. Это **плоская сеть** — архитектурный антипаттерн, где все устройства находятся в одном broadcast-домене с неограниченным доступом друг к другу.

**Сегментация сети** делит инфраструктуру на изолированные зоны, каждая со своим уровнем доверия и правилами доступа. **VLAN** (Virtual LAN) — логическое разделение на уровне L2. **DMZ** (Demilitarized Zone) — буферная зона для публичных сервисов. **Microsegmentation** — детальное разделение на уровне отдельных workload'ов, не сетей. **Zero Trust segmentation** — разрешение доступа на основе identity, не IP-адреса.

В 2026 году сегментация — не опция, а необходимость. **Ransomware** движется lateral через плоские сети за минуты. **Compliance** (PCI DSS, HIPAA, GDPR) требует изоляции sensitive data. **Cloud-native** приложения требуют сегментации, которая перемещается вместе с workload'ами.

| Тип сегментации | Уровень | Гранулярность | Технология | Use Case |
|----------------|---------|---------------|------------|----------|
| VLAN | L2 | Подсеть /24-/16 | 802.1Q | Отделы, функции |
| DMZ | L3 | Зона | Firewall + routing | Public services |
| PVLAN | L2 | Порт | Private VLAN | Multi-tenant hosting |
| VXLAN / Overlay | L2 over L3 | Хост / VM | VXLAN, GENEVE | Cloud, SDN |
| Microsegmentation | L4-L7 | Workload / процесс | VMware NSX, Cilium | Zero Trust |
| Identity-based | L7 | Identity / role | SPIFFE, Istio | Service mesh |

**Эволюция:** В 1995 **VLAN** (IEEE 802.1Q) позволил логически разделить свитчи. В 2000 **DMZ** стал стандартом для защиты периметра. В 2005 **PVLAN** (Private VLAN) защитил хостинг-провайдеров. В 2012 **VXLAN** (RFC 7348) принёс L2-сегментацию в облака. В 2014 **Google BeyondCorp** объявил смерть perimeter. В 2019 **microsegmentation** стала must-have для container security. В 2026 **identity-based segmentation** — новый стандарт: не "что подключается", а "кто подключается".

## Часть 2. Три реальных сценария, почему это важно

### Сценарий 1: Больница и распространение ransomware
Больница использует плоскую сеть: ПК регистратуры, медицинские приборы, серверы ЭМК (электронные медицинские карты), financial system — всё в VLAN 10 (10.0.0.0/16). Сотрудник открывает фишинговое письмо, **TrickBot** инфицирует ПК. Червь сканирует сеть через **SMB**, находит непропатченный Windows-сервер ЭМК (**EternalBlue**), шифрует 500K записей. Затем прыгает на financial system. Восстановление: 3 недели, $10M, остановка приёма пациентов. **Урок:** Плоская сеть = ransomware paradise.

### Сценарий 2: Финтех и PCI DSS segmentation
Платёжный процессор обрабатывает карты. **PCI DSS Requirement 1** требует: "Установить и поддерживать firewall configuration standard для защиты CHD". Компания разделяет **CDE** (Cardholder Data Environment) от корпоративной сети через firewall. Но разработчик для удобства деплоя настраивает **jump host** с двумя NIC: одна в CDE, другая в corporate. Злоумышленник компрометирует jump host через corporate сеть (фишинг), получает доступ к CDE. 2M записей карт утекают. **Урок:** Segmentation не работает, если есть "удобные" обходы.

### Сценарий 3: Облачный SaaS и Kubernetes microsegmentation
SaaS-платформа на Kubernetes. Изначально все поды в одном namespace с открытым egress. Злоумышленник компрометирует **frontend pod** через уязвимость в библиотеке. Из frontend он свободно обращается к **database pod** (порт 5432), **message queue** (5672), ** secrets store** (8200). Сливает всю базу и конфиденциальные ключи. После инцидента внедряют **Cilium Network Policies**: default-deny, identity-based доступ между сервисами, DNS-aware egress filtering. **Урок:** Cloud-native требует cloud-native segmentation.

## Часть 3. Техническая глубина: Как устроена сегментация

### VLAN и PVLAN: Классика L2
**Аналогия:** VLAN — это как если бы вы разделили здание на этажи с отдельными лестницами. Жители одного этажа не встречают жителей другого. PVLAN — это этажи, где квартиры даже на одном этаже не видят друг друга.

**802.1Q VLAN** добавляет 4-байтовый тег к Ethernet-кадру: **TPID** (0x8100) + **PCP** (приоритет) + **DEI** + **VID** (VLAN ID, 12 бит = 4094 VLAN). Свитчи маршрутизируют кадры на основе VID.

**PVLAN (Private VLAN)** — расширение VLAN для изоляции портов внутри одного VLAN:
- **Promiscuous port**: Общается со всеми (шлюз, firewall)
- **Community port**: Общается с портами той же community + promiscuous
- **Isolated port**: Общается только с promiscuous (не с другими isolated)

| Тип порта PVLAN | Может общаться с | Use Case |
|----------------|-----------------|----------|
| Promiscuous | Всеми | Gateway, firewall, load balancer |
| Community | Той же community + promiscuous | Группа серверов одного приложения |
| Isolated | Только promiscuous | Хостинг-клиенты, не доверяющие друг другу |

**Проблемы VLAN:**
1. **VLAN hopping**: Двойное тегирование 802.1Q обходит segmentation. Защита: native VLAN ≠ user VLAN, pruning, port security.
2. **Broadcast domain**: ARP, DHCP, STP — всё это broadcast, нагружающий сеть.
3. **L3 routing**: VLAN'ы разделяют L2, но маршрутизация между ними требует firewall — часто забывают.
4. **Scalability**: 4094 VLAN — мало для крупных облаков.

### VXLAN и Overlay Networks: Сегментация в облаке
**VXLAN** (Virtual Extensible LAN, RFC 7348) инкапсулирует L2-кадры в UDP-пакеты (порт 4789). **24-bit VNI** (VXLAN Network Identifier) = 16M сегментов (против 4094 VLAN). VXLAN позволяет создавать L2-сегменты поверх L3-сетей — критично для облаков, где L2 между хостами невозможен.

**GENEVE** — эволюция VXLAN, более гибкий заголовок (variable-length options). Используется в **Open vSwitch**, **Cilium**.

**Overlay security considerations:**
- **Encapsulation overhead**: VXLAN добавляет 50 байт — MTU должен быть 1550+, иначе fragmentation.
- **BUM traffic**: Broadcast, Unknown unicast, Multicast — реплицируются через underlay, создавая нагрузку.
- **Policy enforcement**: Firewall в overlay сложнее, чем в underlay. **Distributed firewall** (VMware NSX, Cilium) — решение.

### Microsegmentation: От сети к workload
**Microsegmentation** — разделение на уровне отдельных workload'ов, не сетей. Каждый сервер, контейнер, VM — свой security boundary.

**Технологии microsegmentation:**
- **Hypervisor-based**: VMware NSX, Cisco ACI — firewall на уровне vSwitch, policies привязаны к VM, не IP.
- **Host-based**: Illumio, Guardicore — агент на ОС управляет iptables/Windows Firewall.
- **Container-native**: Cilium (eBPF), Calico — policies привязаны к pod identity, не IP.
- **Service mesh**: Istio, Linkerd — mTLS + authorization между сервисами, L7-aware.

**Identity-based segmentation** — следующий уровень: доступ определяется **кем** (user, service, device), а не **откуда** (IP, subnet). **SPIFFE/SPIRE** выдаёт workload'ам short-lived identities (SVID), **Cilium** использует these identities для policies.

| Подход | Гранулярность | Mobility | Complexity | Зрелость |
|--------|--------------|----------|------------|----------|
| VLAN | Subnet | Низкая | Низкая | Зрелая |
| VXLAN | Subnet (облако) | Средняя | Средняя | Зрелая |
| Hypervisor microseg | VM | Высокая | Высокая | Зрелая |
| Host-based microseg | Process | Высокая | Очень высокая | Зрелая |
| Container network policies | Pod | Очень высокая | Средняя | Растущая |
| Service mesh | Service | Очень высокая | Высокая | Растущая |
| Identity-based | Workload identity | Очень высокая | Очень высокая | Новая |

## Часть 4. Архитектура защиты: Defense in Depth для Segmentation

### Многоуровневая сегментация

| Уровень | Технология | Стандарт | Реализация |
|---------|-----------|----------|------------|
| ЦОД / Облако | AZ / Region isolation | AWS/Azure/GCP best practices | Multi-AZ, disaster recovery |
| Сеть | VLAN / VXLAN | 802.1Q, RFC 7348 | Cisco, Arista, VMware NSX |
| DMZ | Perimeter firewall | NIST SP 800-41 | Palo Alto, Fortinet, pfSense |
| Microsegmentation | Host-based firewall | NIST SP 800-125B | Illumio, Guardicore |
| Container | Network Policies | Kubernetes docs | Cilium, Calico, Weave |
| Service | Service Mesh mTLS | Istio/Linkerd docs | Istio, Linkerd, Consul |
| Identity | SPIFFE/SPIRE | SPIFFE standard | SPIRE, cert-manager |

### Мониторинг и метрики
**Ключевые метрики сегментации:**
- **East-west traffic ratio**: % трафика внутри сети vs north-south (через периметр). Высокий east-west = риск lateral movement.
- **Segment boundary crossings**: Количество обращений между зонами. Рост = возможный policy drift.
- **Unsegmented assets**: % систем без assigned security zone.
- **Policy coverage**: % workload'ов с active microsegmentation policies.
- **Lateral movement detection**: Обращения к необычным портам после compromise одного хоста.

### Реальные инциденты

**Инцидент 1: NotPetya через Maersk (2017)**
**NotPetya** началась через украинское ПО **M.E.Doc** (compromised update). Но распространилась глобально через **EternalBlue** (SMB) и **Mimikatz** (credential extraction). **Maersk** — крупнейший контейнерный перевозчик — потерял 49K laptops, 1200 приложений, все 7 domain controllers. Причина: плоская сеть, где одна compromised workstation имела доступ к всему. Восстановление: 2 недели, $300M. **Урок:** Плоская сеть + ransomware = business continuity disaster. Защита: microsegmentation, credential guard (Windows Defender Credential Guard), application whitelisting, offline DC backups.

**Инцидент 2: Target через HVAC (2013)**
**Target** сегментировала сеть: POS, corporate, HVAC. Но firewall разрешал HVAC → POS для "управления температурой". Злоумышленник компрометировал HVAC-вендора, использовал VPN, прошёл через firewall в POS-сегмент. 40M кредитных карт. **Урок:** Segmentation слаба, если есть exceptions для "удобства". Защита: default-deny, regular firewall audit, vendor access through jump host с MFA и session recording, Zero Trust (не доверять даже "доверенным" системам).

## Часть 5. Практика: Подготовка к собеседованию и чек-лист

### Три распространённых заблуждения
1. **"VLAN = security"** — VLAN разделяет broadcast domain, но не защищает от: VLAN hopping, routing между VLAN без firewall, ARP-spoofing внутри VLAN. VLAN — сегментация, не защита.
2. **"DMZ защищает внутреннюю сеть"** — DMZ — буфер, но не барьер. Если web-сервер в DMZ компрометирован, и есть доступ из DMZ в internal (через API, базу данных), злоумышленник пройдёт дальше. DMZ нужен плюс firewall rules plus application-level security.
3. **"Microsegmentation слишком сложна для нашего размера"** — Microsegmentation масштабируется. Начните с crown jewels (базы данных, DC), постепенно расширяйте. Инструменты (Cilium, NSX) упрощают deployment через policy generation из observed traffic.

### Три ситуационных вопроса для Senior+

**Q1:** Как защититься от VLAN hopping в корпоративной сети?
**A:** (1) **Native VLAN отличная от user VLAN** — native VLAN не используется для user traffic; (2) **VLAN pruning** — trunk'и несут только нужные VLAN; (3) **Port security** — access порты в static mode, не dynamic trunking; (4) **802.1X** — аутентификация перед доступом к VLAN; (5) **Private VLAN** — isolated порты не общаются друг с другом даже в одном VLAN. Defense in depth на L2.

**Q2:** Как реализовать microsegmentation для legacy приложения, которое не поддерживает современные auth?
**A:** (1) **Network-level microsegmentation** — Illumio/Guardicore агент на ОС ограничивает исходящие соединения legacy app, не требуя изменений; (2) **Service mesh sidecar** — Envoy proxy добавляет mTLS + auth, не меняя legacy; (3) **Jump host / bastion** — весь доступ к legacy через контролируемый хост с MFA; (4) **Database firewall** — Imperva, DBShield — ограничивают SQL-запросы, не требуя app changes; (5) **Read-only replicas** — legacy читает из replica, не пишет в primary. Ключ: не менять legacy, менять сеть вокруг него.

**Q3:** Почему identity-based segmentation лучше IP-based, и какие риски?
**A:** **Преимущества identity-based:** (1) **Mobility** — workload меняет IP ( Kubernetes pod reschedule), identity сохраняется; (2) **Multi-cloud** — одинаковые policies в AWS, Azure, on-prem; (3) ** granularity** — доступ между сервисами, не подсетями; (4) **Audit** — кто обращался к чему, не какой IP к какому. **Риски:** (1) **Identity provider compromise** — если SPIRE/SPIFFE CA скомпрометирован, вся сегментация рушится; (2) **Complexity** — новый слой infrastructure требует operational maturity; (3) **Performance** — identity lookup на каждое соединение добавляет latency; (4) **Vendor lock-in** — зависимость от platform (Istio, NSX). Рекомендация: гибрид — IP-based для perimeter, identity-based для internal services.

### Чек-лист сегментации (10 пунктов)

| # | Проверка | Ответ | Критичность |
|---|----------|-------|-------------|
| 1 | Плоская сеть устранена (нет /16 без разделения)? | Network diagram с зонами | Обязательно |
| 2 | DMZ для public-facing сервисов настроена? | Firewall rules: DMZ → Internal deny by default | Обязательно |
| 3 | VLAN hopping защита включена (native VLAN ≠ user)? | `show vlan` на свитчах | Обязательно |
| 4 | PVLAN или isolated ports для multi-tenant? | `show port private-vlan` | Высокая |
| 5 | Firewall между всеми VLAN (не L3 routing)? | ACL inspect на L3 interfaces | Обязательно |
| 6 | CDE (PCI) полностью сегментирована и audited? | QSA assessment, quarterly scan | Обязательно |
| 7 | Kubernetes Network Policies default-deny? | `kubectl get netpol --all-namespaces` | Высокая |
| 8 | Microsegmentation на workload-level (не только VLAN)? | Illumio, Cilium, или NSX policies | Высокая |
| 9 | OT/ICS сегмент от IT с физическим/air-gapped разделением? | Purdue Model Level 3.5 firewall | Обязательно |
| 10 | Identity-based segmentation пилотируется для critical services? | SPIFFE/SPIRE или Istio policies | Средняя |

### Key Takeaways
1. **Плоская сеть — антипаттерн** — ransomware lateral movement, compliance failure, operational chaos.
2. **VLAN ≠ безопасность** — segmentation needs firewall between zones, not just L2 separation.
3. **DMZ — буфер, не барьер** — compromised DMZ host + weak internal rules = full breach.
4. **Microsegmentation = future** — от subnet-based к workload-based, from IP to identity.
5. **Zero Trust segmentation** — verify every access, even inside perimeter. "Inside" ≠ "trusted".
