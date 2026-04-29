# Diamond Model: Четырёхсторонний анализ угроз как основа Threat Intelligence

## Что такое Diamond Model?

Представьте детектива, расследующего серию убийств. Он не просто записывает: «найден труп на третьей улице». Он создаёт систематический портрет: кто убийца — его прошлое, мотивация, навыки; кто жертва — почему именно он, что у него было; какие инструменты — орудие, взломщик замков, отравляющее вещество; какая инфраструктура — убежище, транспорт, сеть информаторов. Четыре точки, соединённые линиями, образуют ромб — и только в анализе связей между ними детектив раскрывает картину. Убийца без мотива — пустой профиль. Жертва без слабостей — непонятный выбор. Инструменты без инфраструктуры — неуловимые улики.

**Diamond Model** — это фреймворк для систематического анализа кибератак, который постулирует: понять атаку можно, только рассмотрев четыре взаимосвязанных аспекта: злоумышленник (Adversary), жертва (Victim), инфраструктура злоумышленника (Infrastructure) и его возможности (Capability). Модель разработана аналитиками Агентства национальной безопасности США (Sergio Caltagirone, Andrew Pendergast, Christopher Betz, 2013) и стала стандартом де-факто в профессиональном сообществе Threat Intelligence. Главная её сила — не классификация, а **связи**: Diamond Model утверждает, что ценность анализа не в изолированных данных, а в анализе отношений между сущностями.

Ключевой принцип: угроза не существует в вакууме. Фишинг — это не просто «вредоносное письмо», а точка на линии «Злоумышленник → Инструмент → Инфраструктура → Жертва». Если убрать злоумышленника — письмо не было бы написано; если убрать инфраструктуру — письмо не дошло бы до жертвы. Diamond Model требует описать каждую из этих четырёх точек и, что критично, каждую связь между ними. Это делает его не «таблицей фактов», а **графовым анализом угроз**, где узлы — сущности, а рёбра — наблюдаемые отношения.

Фундаментальная проблема, которую решает Diamond Model: Threat Intelligence без связей — это шум. Без модели аналитики генерируют тысячи IoC (Indicators of Compromise): IP-адреса, домены, хеши файлов. Но эти данные не подсказывают, что делать: блокировать один IP бесполезно, если злоумышленник арендует тысячи в день. Diamond Model переводит IoC в context, показывая, какие элементы инфраструктуры принадлежат конкретной кампании, какие кампании принадлежат конкретной группе, какая группа нацелена на вашу отрасль.

## Эволюция и мотивация

До Diamond Model Threat Intelligence был IoC-центричным: аналитики собирали списки IP, доменов, хешей вредоносного ПО и отдавали SOC-операторам для блокировки. Этот подход работал, пока атаки были массовыми и нецелевыми (фишинг со зловредом, exploit-kit landing pages). К 2010-м годам три сдвига изменили картину. Первый: **targeted attacks** — APT-группы выбирают конкретную жертву и долгое время остаются незамеченными. Простая блокировка IP не остановит атаку: злоумышленник пересядет на другой сервер. Второй: **shared infrastructure** — APT-группы делят C2-серверы, bulletproof hosting, наборы инструментов. Один IP может принадлежать пяти кампаниям, а один вредонос — использоваться разными группами. Без связей между IoC разведка не отличает группу А от группы Б. Третий: **attribution complexity** — государственные атаки, mercenary APT, криминальные синдикаты — границы стираются. Фейковые флаги (false flags) играют роль: группа имитирует тактику другой группы для дезинформации.

Diamond Model — попытка структурировать этот хаос. Вместо бесконечных списков он предложил mental model: рассматривай атаку как систему из четырёх подсистем, изучи каждую и, что важнее, изучи каждую связь между ними. Это принципиальный сдвиг: от «что заблокировать» к «кто нас атакует, зачем и что будет дальше».

Аналогия: пожарные в 1950-х годах получали звонок о пожаре, но не знали, где он, как распространяется, сколько людей в здании. Они реагировали по факту — IoC-подход. К 2000-м появились IoT-датчики, камеры, building management systems. Fire command теперь видит: где огонь (Capability), какие зоны affected (Victim), какие пути эвакуации доступны (Infrastructure), какие структуры угрожают крах (Adversary — огонь как агент). Diamond Model — это mental framework для пожарного командира: без понимания всех четырёх аспектов он теряет людей.

**NIST SP 800-53**, контроль **IR-4** (Incident Handling) и **IR-10** (Integrated Information Security Analysis Team), требует от организаций не просто реагировать на инциденты, а анализировать их в контексте более широкой threat landscape. Diamond Model предоставляет структуру для этого анализа. **CIS Controls v8**, контроль **17.1** (Develop and Maintain an Incident Response Plan), предписывает интеграцию Threat Intelligence в incident response — именно Diamond Model даёт формат этой интеграции. **MITRE ATT&CK**, де-факто стандарт TTP-описания, интегрируется с Diamond Model: Capability-вершина diamond содержит набор ATT&CK techniques, позволяя аналитику сопоставить конкретную кампанию с общей библиотекой.

## Зачем это нужно?

### Сценарий 1: Анализ APT-кампании для понимания вектора

Фармацевтическая компания обнаруживает следы взлома в сети. SOC видит beacon-трафик на известный IP, использует Cobalt Strike, и находит exfiltrated документы о clinical trials. Вопрос: что делать? Простой подход — блокировать IP, чистить вредоносы. Diamond Model — поглубже: «Кто это? APT-группа, известная атаками на фармацевтику — мотивация кража IP. Жертва — clinical trials (имеют высокую стоимость на чёрном рынке). Инфраструктура — арендованные VPS, обновляемые раз в месяц. Capability — Cobalt Strike + custom downloader + credential dumping. Вывод: blocking IP не остановит — инфраструктура обновится. Нужно: усилить network segmentation между clinical data и corporate network (связь Victim → Infrastructure), review access controls на clinical trial repos (связь Victim → Capability — если бы credential dumping не сработал, атака остановилась бы), deploy deception в DMZ (связь Adversary → Infrastructure — if they come back, we see them first).»

**Бизнес-риск**: stolen intellectual property оценивается в $500M-$1B (cost of clinical trial replication). **Стандарт**: **ISO 27001 Annex A.8.2** требует анализа рисков — Diamond Model обеспечивает структуру для риск-ассессмента при инциденте.

### Сценарий 2: Threat Intelligence Sharing в ISAC

Член Information Sharing and Analysis Center (ISAC) получает ежедневно десятки IoC-фидов от других членов. Большинство — неактуальны: IP банкрота, а у них в отрасли другая инфраструктура. Diamond Model позволяет фильтровать: если известно, что Adversary-группа X атакует только энергетику (Victim sector), а мы финансы — этот feed неактуален. Если группа Y использует Capability-набор, для которого у нас есть контрмеры — low priority. Если группа Z меняет Infrastructure каждые три дня, но задействует один и тот же C2-протокол — нужно создать network-based detection signature по протоколу, а не по IP.

**Комплаенс**: **CIS Controls v8 17.4** (Conduct Security Awareness Training) — Diamond Model профилирует adversary TTPs, чтобы training был targeted. **NIST Cybersecurity Framework**, функция **Detect** (DE.CM-1), требует мониторинга сетевого трафика — Diamond Model направляет, какие именно IOC и TTP мониторить.

### Сценарий 3: Планирование proactive defence

Команда CISO хочет выделить бюджет на security improvements на следующий год. Анализ Diamond Model за год: Adversary-профили показывают, что 80% атак на отрасль — фишинг (Capability). Victim-профили: 60% жертв — сотрудники с доступом к email. Infrastructure: 90% фишинговых сайтов — созданы через compromised WordPress (shared hosting). Выводы proactive defence: 1) усилить email security (DMARC, sandboxing) — counter Capability; 2) user awareness training с фокусом на фишинг (Adversary TTPs) — counter Victim weakness; 3) monitoring CMS (WordPress) на defacement — counter Infrastructure. Diamond Model превращает reactive список IoC в proactive security roadmap.

**Бизнес-риск**: без proactive defence организация расходует бюджет на реагирование ($500K-$2M per breach), а не на предотвращение. **Стандарт**: **ISO 27001 Annex A.5.7** (Threat Intelligence) требует «систематический подход к мониторингу угроз» — Diamond Model даёт framework этого подхода.

### Сценарий 4: Incident Response и Prioritisation

Ransomware атакует организацию в пятницу 4 PM. SOC обнаружил encryptor на двух рабочих станциях. Diamond Model: Adversary — криминальная группировка Ransomware-as-a-Service (платформа). Victim — mid-size manufacturing (платёжеспособен, но дедлайны critical). Capability — commodity ransomware (off-the-shelf), развёрнутый через valid credentials (likely purchased on dark web). Infrastructure — Tor для ransom negotiation. Всё это говорит: группа не APT (не целенаправлена), но профессиональна (платформа). Рекомендация: do not pay — оплата не гарантирует decryption key, и платформа продолжит атаки. Сфокусироваться на: 1) restore from backups (Capability: commodity ransomware — known decryptors possible); 2) identify how valid credentials получены (phishing or password spray — это Victim weakness). Diamond Model prioritizes actions over panic.

## Основные концепции

### Четыре вершины Diamond: структурирование атаки

**Adversary** — не просто «злоумышленник». Diamond Model требует атрибуции по нескольким аспектам: кто (имя группы или persona), почему (мотивация: финансовая, идеологическая, шпионаж, саботаж), сколько (ресурсы: solo actor vs APT с бюджетом $50M/year), как долго (TTP evolution: новичок использует commodity tools, APT развивает custom capabilities). Критично различать **operator** (человек, нажимающий кнопки) и **author** (группа, создающая инструменты). Один и тот же вредонос (Capability) может быть использован разными Adversary — без атрибуции operator мы не знаем, кто нажал.

Атрибуция — самый сложный аспект Diamond Model. **NIST SP 800-53, AU-6** (Audit Review) и **IR-10** (Analysis) требуют анализа инцидентов для понимания threat actor — Diamond Model предоставляет структуру этого анализа. Но важно помнить: атрибуция по коду вредоноса, по времённой зоне операций, по языковым метаданным — это circumstantial evidence, не proof. Использование false flags (злоумышленник копирует TTPs другой группы) делает атрибуцию inherently uncertain. Diamond Model решает эту проблему: вместо «кто именно» фокусируется на «какой профиль» — adversary persona, а не identity.

**Victim** — не только «мы». Diamond Model требует анализа: какой тип жертвы (отрасль, размер, геополитическая значимость), почему именно эта жертва (возможности, данные, связи, временные окна), какие у жертвы были слабости (уязвимости, которые привели к компрометации). Важный аспект: victim может быть не цель, а means. Атакующий может использовать вашу организацию как pivot point для атаки на партнёра или клиента.

**Infrastructure** — всё, что связывает adversary с victim: C2-серверы, фишинговые домены, email-адреса, bulletproof hosting, VPN, proxy-сети, cryptocurrency wallets. Ключевое различение: **infrastructure controlled by adversary** (собственные серверы, арендованные VPS) и **infrastructure shared by multiple adversaries** (Bulletproof hosting, compromised domains, public Telegram channels). Разведка, фокусирующаяся только на блокировке IP первой категории, бесполезна (злоумышленник заменит за минуты). Разведка, фокусирующаяся на shared infrastructure, даёт sustainable detection (если блокируется хостинг-провайдер, все группы на нём потеряют C2).

**Capability** — не просто «вредоносное ПО». Это полный набор TTPs: initial access (фишинг, supply chain, exploit), execution (PowerShell, custom shellcode), persistence (scheduled task, WMI event subscription), privilege escalation (CVE-exploit, token manipulation), defense evasion (process injection, anti-forensics), credential access (Mimikatz, LSASS dump), discovery (network scanning, system inventory), lateral movement (PSExec, WMI), collection (file exfiltration, database dump), C2 (HTTPS beacon, DNS tunneling), exfiltration (HTTPS POST, cloud storage), impact (ransomware, wipers). Diamond Model структурирует их не как список, а как систему: какие TTPs с какой Infrastructure используются против какого Victim.

### Связи между вершинами: аналитическая ценность Diamond Model

Diamond Model — это не четыре точки, а **граф**. Связи между вершинами содержат ключевую информацию:

- **Adversary → Victim**: что злоумышленник делал с жертвой, какие данные крал, какой доступ получил. При повторении связей (один Adversary → множество Victim) — это pattern (targeting отрасли, типа данных, региона).
- **Adversary → Infrastructure**: чья инфраструктура. Если adversary использует собственные серверы — high attribution confidence. Если shared infrastructure — low confidence (возможно, сервер арендован, и его используют другие). Если adversary использует legitimate infrastructure (GitHub, Dropbox, Google Drive) для C2 — это сигнал высокой sophisticated (evasion detection).
- **Adversary → Capability**: какие инструменты применяются. Если adversary использует custom malware — high-skill, high-budget, likely APT. Если commodity tools — low skill, likely script kiddie или RaaS affiliate.
- **Victim → Infrastructure**: какие ресурсы жертвы были задействованы. Например: фишинговый email прошёл через Exchange Online (Infrastructure жертвы) → указывает на weakness в email security.
- **Infrastructure → Capability**: где размещены инструменты. Знание, что malware хостится на cloud storage provider X, позволяет блокировать доступ на уровне egress filtering.
- **Capability → Victim**: какие TTPs использовались против конкретной жертвы. Это mapping на ATT&CK (MITRE), что делает анализ comparable с другими кампаниями.

**NIST SP 800-53, IR-10** (Analysis) требует «интегрированного анализа данных для выявления паттернов угроз» — Diamond Model предоставляет шаблон этого анализа. **CIS Controls v8 17.1** (Develop and Maintain an Incident Response Plan) — использование Diamond Model структурирует план реагирования.

### Анализ кампаний (Campaign Analysis)

Diamond Model вводит понятие **Campaign** — серии связанных событий, объединённых одним или несколькими adversary. Это позволяет аналитику подняться над уровнем отдельного инцидента. Campaign analysis отвечает на вопросы: атакует ли одна группа многих (кампания широкая) или одна группа многократно одну жертву (targeted)? Менялись ли TTPs со временем (evolution)? Нужно ли ожидать новой атаки (predictive intelligence)?

Кампания структурируется в Diamond Model как множество diamond snapshots: каждый инцидент — snapshot, кампания — суперпозиция. Аналитик ищет commonality: одинаковый Adversary, общая Infrastructure, повторяющиеся Capability, похожие Victim. Если все три параметра совпадают — это одна кампания, высокая уверенность. Если Infrastructure меняется, но Adversary и Capability — те же — это evolution кампании. Если Adversary разный, но Capability и Infrastructure — одинаковые — это shared tooling (низкая атрибуция, но high detection value).

**ISO 27001 Annex A.5.7** (Threat Intelligence) требует «систематического мониторинга угроз для timely detection» — Campaign analysis Diamond Model обеспечивает systematic.

## Сравнение с другими механизмами

| Критерий | Diamond Model | MITRE ATT&CK | Kill Chain (Lockheed Martin) |
|----------|---------------|--------------|------------------------------|
| Модель безопасности | Графовый анализ атакующих | Таксономия TTP | Линейная последовательность этапов атаки |
| Сложность внедрения/поддержки | Средняя (требует аналитика) | Низкая (справочник) | Низкая (семь этапов) |
| Производительность при масштабировании | Хорошая (многие инциденты → кампании) | Хорошая (универсальная таксономия) | Плохая (линейность упрощает сложные атаки) |
| Устойчивость к обходу | Высокая (фокус на adversary, не IoC) | Средняя (TTPs меняются медленнее IoC) | Средняя |
| Фокус | Кто и зачем (профиль) | Как (TTP) | Когда и по какому пути (последовательность) |
| Типичные сценарии | Threat Intelligence, strategic planning, resource allocation | Operational detection, SOC, purple teaming | Incident response education, basic training |

Diamond Model и ATT&CK — не конкуренты, а complementors. Diamond Model отвечает на «кто и почему», ATT&CK — на «как». На собеседовании частый вопрос: «что выберете?» — правильный ответ: «Оба. Diamond Model для стратегического профилирования, ATT&CK для тактической детекции». Kill Chain — проще, но линейность делает его менее точным для non-linear атак (атака может начаться с lateral movement, минуя initial access). Diamond Model не предполагает последовательность — только связи.

**Скрытые издержки Diamond Model**: data dependency — без достаточного количества инцидентов невозможно строить кампании. Маленькая организация с одним инцидентом в год не построит meaningful diamond. **Subjectivity** — два аналитика могут приписать один IP разным adversary, если у них нет дополнительных данных. **Time-to-value** — кампания строится по накоплению данных месяцами; невозможно получить instant insight. Компромисс: для операционного SOC Diamond Model требует long-term investment.

Сценарий, когда Diamond Model не подходит: разовый инцидент с однозначным IoC (например, mass phishing — нет persistent adversary, нет infrastructure reuse, просто commodity campaign). Здесь enough блокировать IoC; строить diamond — overkill.

## Уроки из реальных инцидентов: SolarWinds (SUNBURST) как Diamond Model analysis

**Хронология**: 2020 год, APT-группа (атрибутирована SVR, российская разведка) компрометирует build-сервер SolarWinds и встраивает бэкдор в легитимное обновление SolarWinds Orion. 18,000 организаций скачивают обновление. APT-группа выбирает ~100 целей (включая Microsoft, FireEye, US Treasury, US DOJ) для второго этапа. Обнаружение: FireEye, декабрь 2020, через detection собственного инструмента red team. Масштаб: компрометация supply chain крупнейшей IT-инфраструктуры — «величайшая кибератака в истории».

**Корневая причина**: compromise of software supply chain (trust in vendor exploited). Diamond Model анализ: Adversary — SVR (state-sponsored APT, high-skill, long-term operation, мотивация — разведка госсекторов). Victim — не только SolarWinds, но и все 18,000 downstream организаций (massive victim surface, но focused secondary targeting). Infrastructure — legitimate SolarWinds update channel (not adversary-controlled C2, but hijacked legitimate infrastructure — highest stealth). Capability — SUNBURST backdoor, TEARDROP loader, Cobalt Strike, custom tooling + no malware on 99.9% of victims (only 100 selected).

**Финансовые, операционные, репутационные последствия**: SolarWinds стоимость breach — $40M+ прямых затрат на investigation + remediation, падение акций на 23%, class-action lawsuits от инвесторов. Для US Government: классифицированные данные украдены, экспертиза дипломатических каналов поставлена под угрозу. Для индустрии: paradigm shift — software supply chain как primary attack vector.

**Как Diamond Model изменил бы исход**:
1. На стадии **Adversary analysis**: Threat Intelligence должен был знать, что SVR исторически атакует supply chain (NotPetya — supply chain через MeDoc). Diamond Model профилирует adversary — и показывает pattern: если adversary изменил infrastructure, но сохранил capability (supply chain compromise), атака предсказуема.
2. На стадии **Victim analysis**: для организаций, использующих SolarWinds, Diamond Model показал бы: «вы — victim, не потому что у вас слабая защита, а потому что vendor compromised». Это переключает focus: не «какой у нас firewall», а «какие vendors у нас имеют privileged access». **NIST SP 800-53 SA-12** (Supply Chain Protection) — контроль, который требует vendor risk management. Diamond Model обнаруживает: victim weakness может быть не in-house, а inherited.
3. На стадии **Infrastructure analysis**: SUNBURST beacon отправлялся на legitimate Amazon AWS S3 domain (avsvmcloud[.]com). Для SOC это выглядело как legitimate traffic. Diamond Model требует: analyze infrastructure for legitimacy. Если adversary использует vendor domain — this is abnormal infrastructure usage, detectable via: behavioral analytics (beacon pattern, не взаимодействие пользователя), code signing verification (хотя обновление было подписано, но build process compromise). **CIS Controls v8 15.7** (Code Signing) требует verify code signing — SolarWinds показал, что signed code может быть malicious.
4. На стадии **Capability analysis**: SUNBURST — custom backdoor, не commodity. Если threat intelligence platform использует Diamond Model, custom capability для APT-группы SVR — это high-confidence indicator. Система должна была raise alert: «custom malware detected on network of organisation using SolarWinds + known APT group TTPs = possible supply chain compromise». Но это требует mature Diamond Model deployment.

**Урок**: Diamond Model применяется не только post-factum, а как proactive intelligence. Ежегодный review adversary capabilities, infrastructure and victim patterns должен генерировать: «groups X, Y, Z historically attack supply chain; what vendors have privileged access? What supply chain controls exist?» **NIST SP 800-207** (Zero Trust) требует verify all connections, включая vendor connections — Diamond Model показывает, что «trusted vendor» может быть adversary infrastructure.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| Threat Intelligence Platform (MISP, Anomali, ThreatConnect) | Centralised storage of Diamond snapshots, correlation between incidents | Continuous intelligence collection | CIS Controls v8 17.3 (Threat Intelligence Integration), ISO 27001 A.5.7 |
| Adversary Profiling Engine | Automated attribution scoring based on TTP overlap | Intelligence analysis | NIST 800-53 IR-10 (Analysis) |
| IoC-to-Diamond Enricher | Сопоставление raw IoC с adversary, infrastructure, capability database | Detection engineering | NIST 800-53 SI-4 (Information System Monitoring) |
| Campaign Visualisation Tool | Графическое представление связей между инцидентами, кампаниями, adversary | Strategic planning, reporting | ISO 27001 A.8.2 (risk visualisation) |
| ATT&CK Mapping Module | Сопоставление capability TTPs с MITRE ATT&CK framework | Operational detection | CIS Controls v8 17.4 (Training), NIST 800-53 IR-4 |

Threat Intelligence Platform (TIP) — ядро Diamond Model operation. MISP (open-source) позволяет хранить и коррелировать diamond snapshots между организациями. Anomali и ThreatConnect (commercial) добавляют adversary profiling, automated attribution и campaign timeline. Компромисс: open-source бесплатен, но требует significant setup; commercial дорог ($50K-$500K/year), но обеспечивает готовую интеграцию.

Операционные метрики: **attribution confidence score** (вероятность, что инцидент принадлежит adversary A), **IoC age** (время между обнаружением IoC и его expiration — если infrastructure меняется быстро, IoC устаревает за дни), **campaign coverage** (какой процент инцидентов организации может быть связан в кампании через Diamond Model), **adversary prediction accuracy** (из predicted campaigns — сколько materialized). **NIST SP 800-55** (Performance Measurement) требует outcome metrics — IoC age и prediction accuracy — это outcome.

## Архитектурные решения

Встраивание Diamond Model в **Defense-in-Depth**: Diamond Model работает на уровне разведки, а не на уровне perimeter defense. Его место — между detection и response: SOC детектирует IoC, Diamond Model analyst объединяет их в adversary profile, который feeds back в detection (signature для TTPs) и response (prioritization — high-confidence APT vs commodity malware).

Взаимодействие с **SIEM/SOAR**: Diamond Model snapshot — enriched event для SIEM. Не просто «IP 1.2.3.4 connected», а «IP 1.2.3.4 — infrastructure of APT29, historical TTPs: credential dumping, PowerShell execution. Priority: critical. Escalate to incident response within 15 minutes». SOAR автоматизирует: при high-confidence Diamond association — автоматический playbook: isolate endpoint, notify IR team, start memory forensics. **NIST SP 800-53 SI-4** (Information System Monitoring) требует automated alert generation — Diamond Model enrichment повышает качество этих алертов.

Взаимодействие с **Zero Trust**: в Zero Trust «никто не доверяется» — Diamond Model помогает понять, какие adversary действительно представляют угрозу. Если организация определила, что adversary X использует фишинг для initial access, Zero Trust требует: email sandboxing, MFA на все входы, behavioural analytics на endpoint — все эти контролы направлены конкретно на Capability этого adversary. Diamond Model делает Zero Trust targeted, а не generic.

Проблемы масштабирования: маленькая организация не может себе позволить dedicated Diamond analyst. Решение: managed threat intelligence service (MSSP), который предоставляет готовые diamond snapshots без in-house capability. Компромисс: cost vs control — организация не контролирует quality analysis.

Проблемы миграции: организация с unstructured threat data (списки IoC в Excel, emails с алертами) переходит к Diamond Model. Миграция: 1) import IoC в TIP (MISP/Anomali), 2) enrich каждый IoC с adversary, capability, infrastructure tags, 3) group into campaigns based on overlap, 4) build first diamond for highest-impact incident. Timeline: 3-6 месяцев для basic maturity.

**Красные флаги деградации**: IoC feed поступает, но не коррелируется (каждый инцидент рассматривается изолированно — отсутствие campaign analysis). Attribution confidence scores — random (аналитик угадывает, нет methodology). Diamond snapshots — stale (> 6 месяцев без update) — adversary TTPs изменились, профиль устарел.

## Подготовка к собеседованию: Common pitfalls & key questions

**Ошибка 1**: «Diamond Model — это просто заполнение шаблона про четыре точки». Кандидат не понимает, что ценность — в связях, а не в изолированных данных. Сильный ответ: «Diamond Model — это graph analysis. Четыре вершины без edges — это четыре факта. Значение появляется, когда аналитик строит campaign из множества diamond: один adversary → несколько victim → общая infrastructure → один набор capability. Это именно то, что NIST IR-10 требует: integrated analysis.»

**Ошибка 2**: «Diamond Model позволяет атрибутировать злоумышленника». Кандидат игнорирует uncertainty attribution. Сильный ответ: «Diamond Model — не forensic tool для proof. Это framework для adversary profiling. Атрибуция по Diamond — probabilistic, не deterministic. False flags (злоумышленник копирует TTPs другой группы) делают 100%-ую уверенность невозможной. Best practice: report confidence levels (low/medium/high) и указывать evidence (infrastructure controlled by X, capability shared with Y, victim pattern Z).»

**Ошибка 3**: «Diamond Model заменяет Kill Chain». Кандидат путает уровни анализа. Сильный ответ: «Diamond Model и Kill Chain — complement. Kill Chain отвечает на 'как атака разворачивается во времени' (стадии). Diamond Model отвечает на 'кто, с какими инструментами, против кого' (entities). Кампания может быть описана и в Kill Chain (timeline этапов), и в Diamond (entities). SOC использует Kill Chain для detection stage, Threat Intel — Diamond Model для strategic profiling.»

**Ситуационный вопрос**: «Ваша организация получает 10,000 IoC ежедневно из фидов. Как Diamond Model поможет prioritise?» Ожидание: кандидат понимает, что Diamond Model фильтрует через adversary/capability profiling. Ответ: «Большинство IoC — commodity и short-lived. Diamond Model prioritises: 1) группировать IoC по infrastructure overlap — если пять IP принадлежат одному bulletproof hosting provider, это один adversary, не пять угроз; 2) map capability TTPs to existing controls — если у нас уже есть detection для этих TTPs, IoC низкий приоритет (компетенция covered); 3) check victim similarity — если adversary historically targets healthcare, а мы финансы — lower priority (хотя не zero). Это снижает 10,000 до 20 actionable alerts. Контроль CIS 17.3 — threat intelligence integration — именно это и требует.»

## Чек-лист понимания

- [ ] Почему IoC-список без Diamond Model analysis — это noise, а не threat intelligence, и какая именно информация теряется?
- [ ] Как Diamond Model справляется с ситуацией, в которой одна группа APT использует инфраструктуру другой группы (false flag), и какой компромисс в attribution?
- [ ] В каком случае Victim анализ Diamond Model показывает, что организация подверглась атаке не по своей вине, а из-за supply chain?
- [ ] Как Capability вершина Diamond Model интегрируется с MITRE ATT&CK, и почему эта интеграция важнее, чем простое наличие обеих моделей?
- [ ] Почему связь Infrastructure → Adversary иногда сильнее, чем связь Infrastructure → Victim, для proactive defense?
- [ ] Как Diamond Model влияет на формирование security budget и roadmaps, а не только на operational incident response?
- [ ] В каком случае Diamond Model создаёт alert fatigue хуже, чем простой список IoC, и как этого избежать?
- [ ] Как маленькая организация (без dedicated Threat Intelligence team) может извлечь value из Diamond Model без full in-house capability?
- [ ] Почему Campaign Analysis требует не только текущих инцидентов, но и исторических данных, и какой стандарт это требует?
- [ ] Как Diamond Model должен взаимодействовать с Zero Trust: усиливает ли он Zero Trust, или Zero Trust делает Diamond Model менее релевантным?

### Ответы на чек-лист

1. **IoC без Diamond Model — noise**, потому что список IP, доменов и хешей не отвечает на критические вопросы: кто стоит за этим IoC (атрибуция), зачем он используется (мотивация), будет ли он использован снова (persistence), какие ещё IoC связаны с этим adversary (completeness). Без этой информации SOC блокирует один IP, но злоумышленник переключается на другой — whack-a-mole. **NIST SP 800-53, IR-10** требует integrated analysis — Diamond Model обеспечивает контекст, который превращает noise в actionable intelligence. Потерянная информация: adversary intent (они шпионы, криминалы или хактивисты?), TTP evolution (меняются ли инструменты?), campaign scale (широкая кампания или targeted?).

2. **False flag — инфраструктурное обманчество**: APT группа A использует VPS provider, который также используется APT группой B. Аналитик видит: IP → группа B (historical association). Но на самом деле группа A арендовала тот же хостинг. Diamond Model справляется через multi-factor analysis: если capability (custom malware signature) и victim profile (targets energy, while A targets finance) совпадают с группой A, а не B — attribution confidence shifts. Компромисс: 100% certainty невозможна; Diamond Model работает с confidence levels. **NIST SP 800-53, AU-6** — audit review требует correlation, но не доказательства. Фреймворк помогает аналитику transparently report: «infrastructure overlaps with B, but capability matches A — confidence: medium.»

3. **Victim как supply chain**: организация использует vendor SaaS-сервис. Adversary компрометирует vendor, а не целенаправленно атакует организацию. Diamond Model Victim-анализ показывает: «мы victim не потому, что нас выбрали, а потому что мы — downstream customer of compromised vendor». Это меняет focus remediation: не «усиливать наш perimeter» (бесполезно — vendor уже inside), а «vendor risk management», «supply chain verification», «zero trust for vendor connections». **NIST SP 800-53 SA-12** (Supply Chain Protection) — этот контроль, основанный на Diamond Model analysis, становится priority. Пример: SolarWinds — victim analysis показал, что downstream организации были victim через supply chain, а не direct targeting.

4. **Capability + ATT&CK**: Diamond Model описывает capability TTPs в domain-specific terms (custom loader, PowerShell execution). ATT&CK даёт таксономию — universal names (T1059.001 — PowerShell). Интеграция: Diamond Model snapshot, где capability = «PowerShell-based execution», mapped to T1059.001 — становится universal language для SOC, purple team, threat sharing. Важность: без ATT&CK mapping TTPs остаются организационно-specific (у нас «ScriptRunner», у них «PSExec») — sharing невозможен. С ATT&CK — TTP comparable между организациями, странами, industries. **CIS Controls v8 17.4** (Training) требует TTP-based training — integration Diamond-ATT&CK делает training evidence-based.

5. **Infrastructure → Adversary сильнее для proactive defense**, потому что: если adversary использует bulletproof hosting provider X, и мы блокируем X — мы блокируем всех adversary на X, независимо от victim. Infrastructure → Victim показывает, какие ресурсы жертвы задействованы (для remediation), но не предсказывает будущие атаки. Proactive defense: «группа APT29 использует provider X → если мы видим traffic to X, это high-confidence APT signal, regardless victim». **NIST SP 800-53 SI-4** (System Monitoring) требует anomaly detection — infrastructure-based detection (traffic to known bad provider) — это anomaly signal.

6. **Diamond Model → budget/roadmap**: ежегодный Diamond analysis показывает: adversary landscape (кто нас атакует, в какой интенсивности), capability trends (фишинг увеличился на 40%, ransomware на 60%), infrastructure shifts (больше cloud C2, меньше on-prem). Это data-driven budgeting: «фишинг вырос → invest в email security + training» (соответствует **CIS 17.4**). «Cloud C2 увеличился → invest в cloud egress monitoring» (соответствует **NIST 800-207 Zero Trust** — verify cloud connections). Без Diamond Model budget распределяется ad-hoc: «купим новый firewall» (не связан с threat landscape).

7. **Alert fatigue через Diamond Model**: аналитик, заполняющий diamond для каждого инцидента, генерирует snapshot. Если инцидентов 100/day, а аналитиков 2 — diamond устаревают, инциденты не анализируются, SOC получает raw IoC без enrichment — alert fatigue. Избежать: 1) prioritisation by impact — diamond строится только для confirmed breaches, не для every phishing attempt; 2) automation — TIP автоматически enrich IoC с adversary tags; 3) tiering — L1 SOC handles IoC blocking, L3 analyst builds diamonds for campaign-level incidents. **CIS Controls v8 17.1** (Incident Response) требует tiered response — Diamond Model применяется на L3, не L1.

8. **Маленькая организация без TI team**: managed threat intelligence service (MSSP) предоставляет diamond snapshots как deliverable. Организация получает: «ваша отрасль атакуется кампанией X; adversary: APT group Y; capability: фишинг + Cobalt Strike; infrastructure: домены .tk; recommendation: enable MFA, deploy email sandboxing». Cost: $5K-$20K/month vs $200K+/year for in-house team. Компромисс: меньше control, generic recommendations, но actionable. **ISO 27001 A.5.7** (Threat Intelligence) не требует in-house, а требует «systematic approach» — MSSP с Diamond Model methodology удовлетворяет.

9. **Campaign analysis требует исторических данных**, потому что кампания — это pattern across time. Без истории каждый инцидент isolated. Для обнаружения campaign нужно: минимум 3-5 связанных инцидентов за 12 месяцев, чтобы identify common adversary/capability/infrastructure. Historical data requirement: **NIST SP 800-53, AU-6** (Audit Review) требует retention audit records — это historical data. **CIS Controls v8 8.5** (Collect Audit Logs) требует centralised logging for analysis. Без 12-month retention Diamond Model не может build campaigns — это operational prerequisite.

10. **Diamond Model + Zero Trust**: Diamond Model усиливает Zero Trust, а не заменяется им. Zero Trust говорит «никому не доверяй»; Diamond Model говорит «вот конкретные adversary, которые пытаются обойти trust». Zero Trust без Diamond Model — generic (всех подозреваем). Diamond Model без Zero Trust — reactive (атакуем уже внутри). Вместе: Zero Trust defines enforcement (no implicit trust), Diamond Model defines threat-based prioritisation (which connections to monitor most closely). **NIST 800-207** требует dynamic risk-based access decisions — Diamond Model provides risk context (adversary capability) for these decisions.

## Ключевые выводы для собеседования

- **Diamond Model — это не IoC-фид, а graph-based adversary profiling**. Ключевой принцип: ценность не в четырёх точках, а в связях между ними. Campaign analysis — superposition множества diamond snapshots, позволяющая выявлять patterns.
- **Главный компромисс**: depth vs data dependency. Diamond Model даёт глубокий strategic insight, но требует: множества инцидентов, dedicated analyst, time (кампания строится месяцами). Без данных — shelfware; с данными — competitive advantage.
- **Связь с NIST SP 800-53**: IR-10 (Integrated Analysis) + SI-4 (Monitoring) + AU-6 (Audit Review) — Diamond Model обеспечивает структуру для выполнения этих контролов. Соответствие CIS 17.3 (Threat Intelligence Integration).
- **Связь с MITRE ATT&CK**: Diamond Model Capability + ATT&CK TTP = universal language для threat sharing и detection engineering. Без ATT&CK integration Diamond Model остаётся организационно-specific и не shareable.

---
_Статья создана на основе анализа публикаций по Diamond Model (Sergio Caltagirone et al.), материалов NIST SP 800-53/800-207, CIS Controls v8, OWASP ASVS v4, ISO 27001:2022, MITRE ATT&CK Framework, публичных post-mortem SolarWinds (2020) и отчётов FireEye/ENISA._
