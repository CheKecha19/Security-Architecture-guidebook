# STRIDE: Классификация угроз Microsoft для систематического моделирования угроз

## Что такое STRIDE?

Представьте себе средневековый замок, который проектирует архитектор. Прежде чем заложить первый камень, он составляет **каталог уязвимостей замка** — систематический перечень всего, что может пойти не так. Кто-то может **подделать ключ** и выдать себя за союзника (spoofing). Злоумышленник может **незаконно проникнуть** через подземный ход и испортить запасы провизии (tampering). Лазутчик способен **украсть планы** обороны и передать их врагу (information disclosure). Предатель из гарнизона может совершить диверсию, а затем **отрицать свою причастность** — «это не я, меня там не было» (repudiation). Слуга, имеющий доступ только к кухне, вдруг получает **доступ к оружейной** (elevation of privilege). Наконец, враг может осадить замок и **перекрыть поставки**, лишив гарнизон воды и продовольствия (denial of service).

STRIDE — это точно такой же каталог, но не для замка, а для программных систем. Аббревиатура STRIDE расшифровывается как **Spoofing identity** (подделка личности), **Tampering with data** (несанкционированное изменение данных), **Repudiation** (отказ от авторства), **Information disclosure** (раскрытие информации), **Denial of service** (отказ в обслуживании) и **Elevation of privilege** (повышение привилегий). Эту классификацию разработали Лорен Конфельдер и Правир Чандра из Microsoft в 1999 году — она стала одним из краеугольных камней Microsoft Security Development Lifecycle (SDL) и выдержала проверку временем на протяжении более двух десятилетий.

Ключевое отличие STRIDE от простого списка угроз — это систематический подход. Каждая буква аббревиатуры соответствует определённой категории нарушений одного из свойств безопасности: Spoofing нарушает **аутентичность** (authenticity), Tampering — **целостность** (integrity), Repudiation — **неотказуемость** (non-repudiation), Information Disclosure — **конфиденциальность** (confidentiality), Denial of Service — **доступность** (availability), Elevation of Privilege — **авторизацию** (authorization). Таким образом, STRIDE покрывает все шесть фундаментальных свойств информационной безопасности, известных как гексада CIAAN (Confidentiality, Integrity, Availability, Authenticity, Authorization, Non-repudiation).

Методология STRIDE не существует в вакууме: она теснейшим образом связана с **Data Flow Diagrams (DFD)** — диаграммами потоков данных. Каждый элемент DFD (процесс, хранилище данных, внешняя сущность, поток данных) имеет свой уникальный профиль угроз по STRIDE. Например, хранилище данных особенно уязвимо к Tampering и Information Disclosure, тогда как процесс — к Spoofing и Elevation of Privilege. Эта привязка «угроза-на-элемент» превращает абстрактную классификацию в практический инструмент, который можно применять итеративно, пройдя по каждому элементу архитектурной диаграммы и последовательно рассмотрев все шесть категорий.

## Эволюция и мотивация

История STRIDE начинается в конце 1990-х годов, когда Microsoft столкнулась с лавиной уязвимостей в своих продуктах. Эпидемия червей — Melissa (1999), ILOVEYOU (2000), Code Red и Nimda (2001) — показала, что традиционный подход «напишем код, а потом попробуем взломать» катастрофически неэффективен. В ответ Microsoft запустила инициативу Trustworthy Computing (2002) под знаменитым меморандумом Билла Гейтса, требовавшую встроить безопасность в сам процесс разработки.

В рамках этой трансформации родился **Microsoft Security Development Lifecycle (SDL)** — и STRIDE стал его неотъемлемой частью. Лорен Конфельдер и Правир Чандра опубликовали статью «The Threats to Our Products» (1999), в которой впервые формализовали шесть категорий. Мотивация была проста: разработчикам нужен был структурированный, воспроизводимый способ думать об угрозах — без требования быть экспертами по безопасности. STRIDE можно объяснить джуниору за час, и он начнёт находить реальные проблемы.

С течением времени STRIDE эволюционировал от чисто Microsoft-внутреннего инструмента к индустриальному стандарту. В 2014 году Microsoft выпустила **Microsoft Threat Modeling Tool** — бесплатное приложение, автоматизирующее построение DFD и анализ по STRIDE. Сообщество OWASP включило STRIDE в руководство **OWASP Threat Modeling**, признав его одним из трёх основных подходов наряду с PASTA и Trike. Сегодня STRIDE интегрирован в стандарты: NIST SP 800-53 (контроль SA-11 — Security and Privacy Engineering Principles), CIS Controls v8 (CIS Control 16 — Application Software Security), ISO 27001:2022 Annex A.8.25 (Secure Development Lifecycle).

Эволюция продолжается: в 2022 году Microsoft обновила STRIDE до модели **STRIDE-LM** (Lateral Movement), добавив седьмую категорию для облачных и гибридных сред. Однако классическая шестикатегорийная версия остаётся золотым стандартом для большинства организаций.

## Зачем это нужно?

Рассмотрим практические сценарии, где STRIDE незаменим.

**Сценарий 1: Проектирование нового микросервиса.** Команда рисует DFD: внешний API-шлюз (процесс), база данных пользователей (хранилище), запрос от клиента (поток данных), мобильное приложение (внешняя сущность). Проход по STRIDE-per-element: Spoofing — злоумышленник выдаёт себя за легитимного клиента (нет аутентификации между внешней сущностью и шлюзом). Information Disclosure — логи передаются в открытом виде (поток данных между шлюзом и лог-коллектором). Denial of Service — один медленный клиент забивает все соединения (процесс API-шлюза). За час команда находит 15 угроз, которые архитектор пропустил бы при ad-hoc анализе.

**Сценарий 2: Оценка безопасности при поглощении компании (M&A).** Due diligence: команда безопасности получает 200-страничную документацию на целевую систему. Вместо чтения всего подряд — строят DFD высокого уровня и прогоняют STRIDE. Вопросы: «Как обрабатывается Elevation of Privilege на админ-панели?», «Есть ли защита от Tampering данных в транзите между сервисами?». STRIDE даёт структуру для аудита, превращая хаос документации в управляемый процесс.

**Сценарий 3: Комплаенс и аудит.** Аудитор требует доказательств, что организация выполняет моделирование угроз (как того требуют NIST SP 800-53 SA-11 и ISO 27001 A.8.25). STRIDE-отчёт с привязкой каждой угрозы к элементу DFD, с присвоением severity и списком mitigation — это аудиторский proof, который невозможно оспорить.

**Сценарий 4: Обучение разработчиков.** В компании 100 разработчиков, и только 2 специалиста по безопасности. STRIDE — это **ментальная модель**, которую можно упаковать в трёхчасовой воркшоп. После обучения каждый разработчик способен провести базовое моделирование угроз для своего компонента, разгрузив команду безопасности.

**Сценарий 5: Agile-разработка и threat-driven design.** В спринте перед реализацией фичи команда проводит 30-минутную STRIDE-сессию по DFD фичи. Найденные угрозы конвертируются в user stories: «Как злоумышленник, я могу подделать JWT-токен... → внедрить валидацию подписи». Это — threat-driven design, когда угрозы определяют архитектурные решения, а не наоборот.

## Основные концепции

### Spoofing identity — подделка личности

**Spoofing** — это нарушение свойства **аутентичности**, когда злоумышленник выдаёт себя за другого пользователя, процесс или внешнюю систему. Это не просто «украли пароль» — spoofing охватывает подделку IP-адресов (ARP spoofing), подделку доменов (DNS spoofing), подделку сертификатов, подделку JWT-токенов, replay-атаки и многое другое. В модели STRIDE-per-element, spoofing наиболее актуален для **внешних сущностей** (external entities) и **процессов** (processes) на DFD. Классическая защита — **многофакторная аутентификация (MFA)** и **взаимный TLS (mTLS)**. Стандарты: NIST SP 800-63 (Digital Identity Guidelines), CIS v8 Control 6 (Access Control Management), ISO 27001 A.9.4.2 (Secure log-on procedures).

### Tampering with data — несанкционированное изменение данных

**Tampering** нарушает **целостность** — как в покое (data-at-rest), так и в движении (data-in-transit). Представьте курьера, который перевозит конверт с документами: если конверт не запечатан и нет контрольной суммы, курьер может незаметно подменить страницы. В программных системах tampering — это: модификация записей в БД через SQL-инъекцию, подмена параметров HTTP-запроса (parameter tampering), атаки man-in-the-middle с изменением трафика, модификация бинарных файлов (binary patching), подделка логов. STRIDE-per-element: tampering угрожает прежде всего **потокам данных** (data flows) и **хранилищам данных** (data stores). Защита: **HMAC** (Hash-based Message Authentication Code), **цифровые подписи**, **WAF** (Web Application Firewall), **валидация входных данных**. Стандарты: NIST SP 800-57 (Key Management), OWASP ASVS V5 (Validation, Sanitization and Encoding), CIS v8 Control 3 (Data Protection).

### Repudiation — отказ от действия

**Repudiation** — самая недооценённая буква STRIDE. Это угроза нарушения **неотказуемости**: пользователь совершает действие (переводит деньги, удаляет запись, одобряет контракт), а затем отрицает это — «я этого не делал, система ошиблась, логи подделаны». В юридическом контексте repudiation делает цифровые доказательства несостоятельными в суде. STRIDE-per-element: repudiation угрожает прежде всего **процессам** (processes), которые совершают критические действия без достаточного аудита. Защита: **журналирование с защитой от подделки** (tamper-proof logging — например, append-only blockchain-подобные структуры), **электронная подпись** транзакций (ГОСТ Р 34.10-2012 в РФ, ECDSA в международной практике), **двухфакторная авторизация** критических операций. Стандарты: NIST SP 800-92 (Log Management), ISO 27001 A.12.4 (Logging and Monitoring), CIS v8 Control 8 (Audit Log Management).

### Information disclosure — раскрытие информации

**Information disclosure** нарушает **конфиденциальность** — это не только утечка данных вовне, но и раскрытие информации пользователю с недостаточными привилегиями внутри системы (horizontal disclosure). Это самая «богатая» категория: сюда попадают ошибки конфигурации (S3 bucket с публичным доступом — аналог «оставленного открытым окна»), трассировочные сообщения в production (stack traces), избыточные данные в API-ответах (over-fetching), side-channel-атаки (timing attacks). STRIDE-per-element: угрожает **хранилищам данных** (data stores) и **потокам данных** (data flows), особенно когда они пересекают границу доверия (trust boundary). Защита: **шифрование** (TLS 1.3, AES-256-GCM), **data masking/tokenization**, **DLP** (Data Loss Prevention), **принцип наименьших привилегий** (least privilege) на чтение. Стандарты: NIST SP 800-52 Rev2 (TLS Guidelines), OWASP ASVS V9 (Communications), CIS v8 Control 3 (Data Protection).

### Denial of service — отказ в обслуживании

**Denial of Service (DoS)** нарушает **доступность**. В отличие от классического «завалить трафиком» (volumetric DDoS), STRIDE рассматривает DoS шире: истощение ресурсов приложения (connection pool exhaustion, thread starvation), алгоритмические атаки (Zip-бомбы, XML-бомбы/Billion Laughs), блокировка учётных записей (account lockout abuse), асимметричные атаки (один медленный клиент держит дорогой ресурс — Slowloris). STRIDE-per-element: DoS угрожает **процессам** (processes) и **потокам данных** (data flows). Защита: **rate limiting**, **circuit breakers**, **таймауты**, **WAF с anti-DDoS**, **graceful degradation**. Стандарты: NIST SP 800-61 Rev2 (Incident Handling), CIS v8 Control 9 (Email and Web Browser Protections), ISO 27001 A.17.1 (Information Security Continuity).

### Elevation of privilege — повышение привилегий

**Elevation of Privilege (EoP)** нарушает **авторизацию** — это атака, при которой пользователь получает права выше назначенных. Горизонтальное повышение (доступ к данным другого пользователя того же уровня — IDOR/Insecure Direct Object Reference), вертикальное (user → admin), эскалация через уязвимости ОС (Dirty COW, PwnKit), через неправильную конфигурацию (sudo misconfiguration). STRIDE-per-element: наиболее критично для **процессов** (processes), особенно тех, что работают с повышенными привилегиями (root/Administrator). Защита: **RBAC/ABAC** (Role/Attribute-Based Access Control), **input validation**, **principle of least privilege** (PoLP), **sandboxing/containerization**. Стандарты: NIST SP 800-162 (ABAC), OWASP Top 10 2021 A01 (Broken Access Control), CIS v8 Control 5 (Account Management).

### STRIDE per element — привязка угроз к элементам DFD

Это самое мощное практическое применение STRIDE. Каждый элемент DFD имеет **предопределённый профиль угроз**:

| Элемент DFD | S | T | R | I | D | E |
|---|---|---|---|---|---|---|
| **Process** (процесс) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Data Store** (хранилище) | | ✓ | ✓ | ✓ | ✓ | |
| **Data Flow** (поток данных) | | ✓ | | ✓ | ✓ | |
| **External Entity** (внешняя сущность) | ✓ | | ✓ | | | |
| **Trust Boundary** (граница доверия) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

Trust Boundary — самый насыщенный угрозами элемент: при пересечении данных через границу доверия (например, из DMZ во внутреннюю сеть) необходимо проверять ВСЕ шесть категорий, потому что именно здесь злоумышленник получает наибольшие возможности. Потоки данных не подвержены Repudiation и Elevation of Privilege (сами по себе — если атака работает на endpoint, это моделируется на процессе). Внешние сущности не подвержены Tampering, Information Disclosure и DoS на диаграмме — они находятся вне скоупа системы (хотя реальные атаки на них возможны, это выходит за границы DFD).

### STRIDE mitigations — контрмеры

Каждой категории STRIDE соответствует стандартный набор mitigation:

| Категория STRIDE | Primary Mitigation Techniques | Ключевые стандарты |
|---|---|---|
| Spoofing | MFA, mTLS, Kerberos, Client Certificates, API Keys + HMAC | NIST SP 800-63, ISO 27001 A.9.4.2 |
| Tampering | HMAC, Digital Signatures, WAF, Input Validation, Checksums | NIST SP 800-57, OWASP ASVS V5 |
| Repudiation | Tamper-proof logging, Digital Signatures, Dual Authorization, Append-only audit trails | NIST SP 800-92, CIS v8 Control 8 |
| Information Disclosure | TLS 1.3, AES-256-GCM, Tokenization, DLP, Least Privilege (read) | NIST SP 800-52 Rev2, ISO 27001 A.10.1 |
| Denial of Service | Rate Limiting, Circuit-breakers, WAF anti-DDoS, Graceful degradation, Autoscaling | NIST SP 800-61 Rev2, CIS v8 Control 9 |
| Elevation of Privilege | RBAC/ABAC, Sandboxing, Least Privilege (execute), Input Validation, Principle of Least Astonishment | NIST SP 800-162, OWASP ASVS V4 |

### STRIDE limitations — ограничения

STRIDE — мощный инструмент, но не серебряная пуля. Главное ограничение: STRIDE **классифицирует угрозы**, но не оценивает их **риск** (вероятность × ущерб). STRIDE скажет «здесь возможен Spoofing», но не скажет, насколько это вероятно и насколько дорого обойдётся. Для оценки риска нужны дополнительные методологии — **DREAD** (тоже от Microsoft, но сейчас вытесняется CVSS), **CVSS v3.1/v4.0** (Common Vulnerability Scoring System), или **OWASP Risk Rating Methodology**.

Второе ограничение: STRIDE завязан на DFD, а DFD хорошо описывают **потоки данных**, но плохо — **бизнес-логику**. Угрозы «клиент заказывает 1000 товаров, отменяет 999, и получает скидку как оптовик» (business logic abuse) STRIDE не поймает — нужны abuse-case-модели.

Третье: STRIDE систематичен, но систематичность порождает **шум**. На большой DFD STRIDE-per-element даёт сотни угроз — большинство из них тривиальны или неприменимы. Без опытного фасилитатора команда тонет в false positives. Решение: triage с severity-фильтром (оставляем только High/Critical).

Четвёртое: STRIDE — статический анализ на этапе проектирования. Он не заменяет SAST/DAST, penetration testing, bug bounty — это комплементарные активности, работающие на разных этапах SDLC.

## Сравнение с другими механизмами

STRIDE — не единственный подход к моделированию угроз. Вот как он соотносится с альтернативами:

| Методология | Тип | Фокус | Когда применять | Сильные стороны | Слабости |
|---|---|---|---|---|---|
| **STRIDE** | Threat classification by category | Все 6 свойств безопасности (CIAAN) | Проектирование, архитектурный review | Прост для обучения, покрывает всю CIAAN-гексаду, tight coupling с DFD | Не оценивает риск, шум на больших DFD, пропускает business-logic угрозы |
| **PASTA** (Process for Attack Simulation and Threat Analysis) | Risk-centric, threat-intelligence-driven | Бизнес-воздействие, имитация атак зрелого противника | Зрелые организации, high-stakes системы (финтех, банки) | Оценивает реальный risk appetite, учитывает threat intelligence | Требует maturity: нужны threat intel feeds, risk management framework, ≈10× дороже STRIDE |
| **VAST** (Visual, Agile, and Simple Threat modeling) | Developer-centric, agile-friendly | Скорость, интеграция в DevOps | Agile-команды, CI/CD pipeline | Мгновенная обратная связь, интегрируется в Jira, меньше церемоний | Меньшая глубина анализа, зависимость от качества архитектурной документации |
| **Trike** | Academic, quantitative-risk-driven | Формализованная оценка риска с матрицами | Исследования, госсектор с высокими compliance-требованиями | Математически строгая оценка риска | Крутая кривая обучения, избыточен для большинства коммерческих проектов |
| **LINDDUN** | Privacy-focused (расширение STRIDE) | Privacy threats (GDPR, CCPA): Linkability, Identifiability, Non-repudiation, Detectability, Disclosure of information, Unawareness, Non-compliance | Системы, обрабатывающие PII (персональные данные) | Покрывает privacy, чего STRIDE не делает | Нишевый фокус, не заменяет STRIDE, а дополняет его |
| **Attack Trees** / MITRE ATT&CK | Attacker-behavior-driven | Цели злоумышленника и тактики их достижения | Threat intelligence, red-teaming, SOC-аналитика | Конкретные техники (T1589, T1190), mapping на реальные APT-группы | Реактивно (описывает известные атаки), не помогает спроектировать защиту с нуля |

**Когда STRIDE — лучший выбор:** если организация только начинает modelling threat program («зрелость 0 → 1»), STRIDE — идеальная точка входа. Низкий порог входа, легко встроить в SDLC, хорошая поддержка тулингом (Microsoft Threat Modeling Tool, OWASP Threat Dragon, IriusRisk). STRIDE хорошо работает на уровне компонентов (component-level threat modelling), в то время как PASTA или VAST — на уровне системы.

**Компромисс STRIDE-vs-PASTA:** STRIDE за час даст вам 15 угроз, из которых 10 — Low/Informational, 4 — Medium, 1 — High. PASTA за день даст вам 3 угрозы, но все 3 — High/Critical, привязанные к бизнес-воздействию. Что ценнее — зависит от контекста. Для стартапа с ограниченными ресурсами STRIDE выигрывает. Для банка с регуляторными штрафами в миллионы — PASTA.

**Компромисс STRIDE-vs-VAST:** STRIDE требует DFD, которую кто-то должен нарисовать и поддерживать в актуальном состоянии (threat modelling drift). VAST автоматически парсит архитектурную документацию из CI/CD — но если архитектурная документация неполна (а она почти всегда неполна), VAST даст неполную картину. STRIDE + ручная DFD дают более полное покрытие, но дороже в поддержке.

## Уроки из реальных инцидентов

**Инцидент 1: SolarWinds SUNBURST (2020) — Elevation of Privilege + Tampering через supply chain**

Атака на SolarWinds Orion Platform — один из самых разрушительных инцидентов в истории. Злоумышленники (APT29/Cozy Bear, приписывается СВР РФ) внедрили **tampered**-библиотеку SolarWinds.Orion.Core.BusinessLayer.dll в легитимный процесс сборки (supply chain attack). Обновление Orion, подписанное валидным сертификатом SolarWinds (Spoofing — легитимный код оказался троянизированным), распространилось на 18 000 организаций.

Применяя STRIDE post-mortem к DFD процесса Orion Update: **Spoofing identity** — злоумышленники выдали свой вредоносный код за легитимный, воспользовавшись доверием к процессу CI/CD. **Tampering with data** — DLL была модифицирована на этапе сборки (data store «Build Server» подвергся tampering). **Information Disclosure** — после активации SUNBURST эксфильтровал данные через поддомены C2, замаскированные под легитимный трафик Orion. **Elevation of privilege** — получив доступ к Orion-серверу, атакующие lateral movement: Golden SAML attack (EoP до уровня global admin Office 365). **Denial of service** — не применялся (stealth >> disruption).

STRIDE-анализ на этапе проектирования CI/CD pipeline («Может ли build-server быть скомпрометирован? Elevation of privilege на build-agent → Tampering артефактов») мог бы выявить необходимость reproducible builds и binary transparency (binary transparency log типа Go checksum database).

**Инцидент 2: Capital One S3 Data Breach (2019) — Information Disclosure + Elevation of Privilege через SSRF**

Бывший сотрудник Amazon Пейдж Томпсон обнаружил **Server-Side Request Forgery (SSRF)** в WAF-конфигурации Capital One, размещённой на AWS. SSRF позволил обратиться к **instance metadata endpoint** AWS (169.254.169.254) — endpoint, который выдаёт временные IAM-credentials EC2-инстанса. Полученные credentials давали доступ к S3-бакету с данными 106 миллионов заявителей на кредитные карты.

STRIDE post-mortem: **Spoofing** — SSRF позволил запросу «притвориться» EC2-инстансом перед metadata endpoint. **Information Disclosure** — (а) раскрытие IAM-credentials через metadata endpoint, (б) раскрытие данных из S3-бакета. **Elevation of Privilege** — WAF имел разрешения выше необходимых (over-privileged IAM role), горизонтальное повышение от WAF-процесса к S3-read.

STRIDE-per-element на DFD AWS-развёртывания: «Process: WAF → Information Disclosure (какие data flows пересекают trust boundary к metadata endpoint?)», «Process: WAF → Elevation of Privilege (каков радиус поражения IAM role?)». Mitigations, которые предотвратили бы брешь: IMDSv2 (Instance Metadata Service v2 — session-oriented, невосприимчив к SSRF), least-privilege IAM-роль (S3 bucket, но не ListAllBuckets), блокировка outbound-трафика от WAF к 169.254.169.254 security group'ой (Defense in Depth).

## Инструменты и средства защиты

Моделирование угроз — не бумажное упражнение. Современный threat modelling toolchain автоматизирует значительную часть STRIDE-анализа:

| Класс инструмента | Название | Позиция в SDLC | Ключевая возможность | Стандарты/совместимость |
|---|---|---|---|---|
| **Desktop Threat Modeling Tool** | Microsoft Threat Modeling Tool | Design phase (Phase 1 SDL) | Drag-and-drop DFD-редактор, автоматическая генерация STRIDE-угроз, генерация mitigation-отчётов | STRIDE-per-element, SDL compliance, NIST SP 800-53 SA-11 |
| **Open-source / Web-based** | OWASP Threat Dragon | Design phase, Agile | Web-интерфейс, DFD + STRIDE автоматизация, экспорт в Jira/Confluence, open-source (Node.js) | STRIDE, LINDDUN, OWASP ASVS, CIS v8 |
| **Enterprise Threat Modeling Platform** | IriusRisk | Design → Production (continuous) | Интеграция в CI/CD, correlation с SAST/DAST findings (ThreadFix), API-first для DevOps, compliance mapping (ISO 27001, PCI DSS) | STRIDE, PASTA, CWE/CVE mapping, ISO 27001 A.8.25 |
| **Code-level threat modelling** | ThreatSpec, Pytm (Python), ThreatPlaybook | Within IDE (shift-left) | Генерация threat models из аннотаций/комментариев в коде, автоматическое построение DFD из кодовой базы | STRIDE, MITRE ATT&CK mapping |
| **Agile / DevSecOps threat modelling** | OWASP Threat Modeling (manual), ThreatPlaybook | Sprint planning (Agile) | 30-минутный ритуал STRIDE-per-element спринта, интеграция в Definition-of-Done | NIST SP 800-53 SA-15, ISO 27001 A.6.1 |
| **Risk quantification overlay** | CVSS v4.0 calculator, FAIR (Factor Analysis of Information Risk) | Post-threat identification | Количественная оценка риска: вероятность эксплуатации × ущерб = risk exposure в долларах | NIST SP 800-30 Rev1 (Risk Assessment), ISO 27005 |

**Microsoft Threat Modeling Tool** (бесплатный, Windows) — самый быстрый способ начать: загружаете DFD, инструмент автоматически генерирует «STRIDE per element» — матрицу из сотен потенциальных угроз. Встроенная база знаний threat properties подсказывает mitigation для каждой. Минус: генерирует много шума (тривиальные угрозы), который нужно фильтровать.

**OWASP Threat Dragon** — opensource-альтернатива, работает в вебе, не привязана к Windows. Позволяет коллаборативно редактировать DFD (Google Docs для threat modelling), имеет REST API для интеграции в pipelines.

**IriusRisk** — enterprise-grade: threat model становится «живым» артефактом. Если позже SAST найдёт XSS в компоненте, IriusRisk автоматически свяжет XSS с Information Disclosure-веткой STRIDE-анализа и поднимет severity. Это — **continuous threat modelling**, сдвиг от «делаем threat model раз в год» к «threat model — живой документ, синхронизированный с реальностью».

## Архитектурные решения

### STRIDE в Defense-in-Depth

**Defense-in-Depth** (эшелонированная оборона) — это размещение множественных защитных слоёв так, чтобы компрометация одного слоя не приводила к компрометации всей системы. STRIDE прекрасно ложится на эту концепцию:

**Уровень 1 — Периметр (Network Layer):** Data Flow через trust boundary Интернет→DMZ. STRIDE: Spoofing (MITM, подделка IP), Tampering (изменение пакетов), Information Disclosure (сниффинг), DoS. Mitigations: mTLS, WAF, DDoS protection (Cloudflare/AWS Shield), network segmentation (VLAN, AWS VPC).

**Уровень 2 — Хост/Контейнер (Host Layer):** Process — веб-сервер (nginx) или контейнер (K8s pod). STRIDE: Elevation of privilege (если nginx скомпрометирован, escape из контейнера в хост), Tampering (изменение бинарников), Information Disclosure (environment variables с секретами). Mitigations: read-only root filesystem, AppArmor/SELinux mandatory access control, Drop capabilities (CAP_NET_BIND_SERVICE), принцип наименьших привилегий для Pod Security Admission (PSA), не запускать от root.

**Уровень 3 — Приложение (Application Layer):** Process — бизнес-логика (API). STRIDE: Elevation of privilege (IDOR, vertical privilege escalation), Tampering (SQL injection, mass-assignment), Repudiation (удаление заказа без audit trail). Mitigations: RBAC (role-based), input validation, parameterised queries, tamper-proof logging.

**Уровень 4 — Данные (Data Layer):** Data Store — PostgreSQL, S3. STRIDE: Information Disclosure (data dump через SQL injection), Tampering (изменение данных без авторизации), DoS (database connection exhaustion). Mitigations: TLS 1.3 for data-in-transit, AES-256-GCM for data-at-rest (TDE), least privilege DB account (SELECT-only для read-replica, DML-only для app-user, DDL только admin/migration-tool), backup immutability (WORM — Write-Once-Read-Many).

Применяя STRIDE на **каждом слое**, команда гарантирует, что mitigation'ы не выстраиваются исключительно на периметре (castle-and-moat anti-pattern), а распределены эшелонированно. Это означает, что даже если злоумышленник прорвал perimeter (уровень 1), он сталкивается с новыми, независимыми мерами на уровнях 2, 3 и 4.

### STRIDE и Zero Trust

**Zero Trust Architecture (ZTA)** — «никогда не доверять, всегда проверять» — это архитектурный принцип, который прямо вытекает из STRIDE. STRIDE отвечает: «Какие угрозы реализуются, если я доверяю этому элементу DFD?» Zero Trust отвечает: «Не доверяй ни одному элементу DFD по умолчанию».

Возьмём типичную microservice DFD: Service A → Service B через mutual TLS (mTLS). С точки зрения STRIDE: Data Flow между A и B подвержен Spoofing (если сертификат украден), Tampering (изменение payload), Information Disclosure (если трафик не шифруется). Zero Trust добавляет: (а) **непрерывную верификацию** сессии — mTLS + SPIFFE (Secure Production Identity Framework for Everyone), где каждый сертификат имеет короткий TTL (5 минут), (б) **микросегментацию** — если Service A скомпрометирован, он может общаться только с Service B (не с Service C или D), потому что network policy (Calico, Cilium, Istio AuthorizationPolicy) ограничивает east-west трафик на уровне L7, (в) **continuous inspection** — запросы проходят через sidecar-прокси (Envoy/Istio mesh), который логирует и может блокировать аномалии.

Таким образом, Zero Trust не заменяет STRIDE — он **реализует mitigation'ы**, которые STRIDE идентифицирует. STRIDE говорит: «Data Flow между A и B подвержен Spoofing: mitigation — взаимный TLS». Zero Trust говорит: «...и ещё добавь SPIFFE с короткоживущими сертификатами, network policy deny-all default, и continuous monitoring, потому что если mTLS сертификат утёк, ты узнаешь об этом быстро, а не через 6 месяцев».

## Подготовка к собеседованию

### Три типичные ошибки при ответе на вопросы о STRIDE

**Ошибка 1: «STRIDE — это просто список из шести угроз».** Это поверхностный ответ, демонстрирующий отсутствие практического опыта. Правильно: «STRIDE — это классификация угроз, привязанная к элементам DFD. Ключевая сила — в **STRIDE-per-element**: каждый элемент DFD (процесс, хранилище, поток, сущность) имеет предопределённый набор категорий, которые к нему применимы. Это превращает modelling из «придумай угрозу» в систематический, воспроизводимый процесс — как checklist пилота перед вылетом».

**Ошибка 2: «STRIDE оценивает риск».** STRIDE не оценивает риск — он только классифицирует угрозу по типу (S/T/R/I/D/E). Для оценки риска нужно поверх STRIDE наложить методологию — DREAD, CVSS или FAIR. Ответ кандидата «STRIDE покажет, насколько опасна угроза» — признак книжного знания без практики.

**Ошибка 3: «STRIDE — устаревшая Microsoft-вещь, сегодня используют MITRE ATT&CK».** Это ложная дихотомия. STRIDE — proactive (проектируем безопасность до того, как система построена). MITRE ATT&CK — reactive (анализируем, как реальные APT-группы реализуют угрозы). Они комплементарны: STRIDE помогает спроектировать систему, устойчивую к целым классам угроз; ATT&CK помогает приоритезировать mitigation'ы на основе реальной threat intelligence. Senior-специалист скажет: «Мы используем STRIDE на design phase, а ATT&CK — для threat modelling threat-actor-centric (PASTA) и red team exercises».

### Три вопроса с подробными ответами

**Вопрос 1: «В вашей системе DFD показывает поток данных User Data между двумя микросервисами через Kafka. Какие категории STRIDE применимы, и какие mitigation'ы вы предложите?»**

**Ответ:** Data Flow между сервисами — это элемент DFD, на который STRIDE-per-element предписывает: Tampering, Information Disclosure, Denial of Service (но не Spoofing/Repudiation/Elevation — они на endpoint'ах). По каждой категории: **Tampering** — злоумышленник может модифицировать сообщение в Kafka-топике. Mitigation: producer подписывает сообщение HMAC-SHA256 с shared secret или приватным ключом; consumer верифицирует подпись и отбрасывает невалидные сообщения. **Information Disclosure** — если Kafka не использует TLS 1.3 + SASL, злоумышленник может перехватывать трафик между producer/consumer и брокером. Mitigation: Kafka TLS encryption + SASL SCRAM-SHA-512 (аутентификация между клиентом и брокером). Плюс data-at-rest encryption — Kafka не шифрует данные на диске по умолчанию; ставим LUKS/dm-crypt на broker-нодах. **Denial of Service** — злоумышленник может флудить топик, вызывая backpressure: producer rate limiting, consumer lag alerting (Prometheus + Kafka Exporter), partition autoscaling. Дополнительно: trust boundary между сервисами (разные security domains) — STRIDE на boundary включает все 6 категорий; поэтому Spoofing тоже проверяем (mTLS между сервисами).

**Вопрос 2: «STRIDE нашёл 70 угроз на DFD из 15 элементов. Как вы решите проблему false positives/шума?»**

**Ответ:** Первое: STRIDE-per-element генерирует много Low-severity угроз («пользователь может зарегистрироваться дважды» — относится к Spoofing? Формально да, практически нет). Нужен **triage с severity-фильтром**: используем CVSS v4.0 или DREAD (Damage, Reproducibility, Exploitability, Affected users, Discoverability), оставляем High/Critical (CVSS ≥7.0). Второе: **threat model consolidation** — группируем угрозы по mitigation'у. Если 15 угроз «Information Disclosure — TLS not enforced» на разных data flows — это одна запись: «Enforce TLS 1.3 on all data flows crossing trust boundaries». Третье: **tool-assisted noise reduction** — Microsoft Threat Modeling Tool позволяет настроить threat property categories: отключаем категории, которые неприменимы к скоупу (например, «Physical tampering» для cloud-native). Четвёртое: **context-based exclusion**: угроза «Denial of Service via Zip Bomb — Upload Resume» применима, только если upload field принимает .zip (проверяем implementation). Если нет — исключаем.

**Вопрос 3: «Что такое Repudiation threat, и почему это важно для compliance? Приведите пример.»**

**Ответ:** Repudiation — угроза, при которой субъект может отрицать совершённое действие, а система не может доказать обратное. Классический пример: пользователь переводит деньги через интернет-банк, затем звонит в банк и говорит «я не делал этот перевод, ваша система ошиблась». Если система логирует транзакции, но логи не имеют **non-repudiation properties** (подпись + timestamp + целостность), банк не сможет доказать, что пользователь инициировал перевод — ни внутреннему compliance-отделу, ни в суде.

Значимость для compliance: ISO 27001 A.12.4 (Logging and Monitoring), PCI DSS Req 10 (Track and monitor all access), SOX Section 404 (IT General Controls — integrity of financial transaction logs), GDPR Article 30 (Records of Processing Activities — must be tamper-proof). Repudiation mitigation'ы: (1) **Digital signature** транзакции — пользователь подписывает запрос на перевод своим приватным ключом (например, через hardware token / FIDO2), сервер верифицирует подпись. (2) **Tamper-proof logging** — append-only audit trail (WORM storage, blockchain anchoring — GuardTime KSI), (3) **Dual authorization** — для операций >N рублей требуется подтверждение второго сотрудника (maker-checker), система логирует обе стороны.

## Чек-лист понимания

1. Расшифруйте аббревиатуру STRIDE. Какое свойство безопасности нарушает каждая буква? Приведите аналогию с физическим замком для каждой категории.

2. Как STRIDE связан с DFD? Что такое STRIDE-per-element? Почему Trust Boundary — самый насыщенный угрозами элемент?

3. Чем Repudiation отличается от остальных категорий STRIDE? Почему это важно для compliance, и какие mitigation'ы обеспечивают Non-Repudiation?

4. STRIDE классифицирует угрозы, но не оценивает риск. Как вы восполните этот пробел? Назовите минимум две методологии оценки риска.

5. Назовите три ключевых ограничения STRIDE. В каких сценариях вы бы применили другую методологию (PASTA, LINDDUN, Attack Trees) вместо или поверх STRIDE?

6. Опишите процесс threat modelling с STRIDE для нового микросервиса: с чего начинаете, какие артефакты создаёте, как интегрируете в Agile-процесс (sprint planning/definition-of-done)?

7. Elevation of Privilege — что это за угроза, и чем горизонтальное повышение привилегий отличается от вертикального? Приведите пример IDOR-атаки и её mitigation.

8. STRIDE и Zero Trust — как эти концепции соотносятся? Приведите пример microservice-архитектуры, где STRIDE идентифицирует угрозы, а Zero Trust предоставляет mitigation'ы.

9. Назовите минимум три инструмента для автоматизации STRIDE-анализа. Чем Microsoft Threat Modeling Tool отличается от IriusRisk или Threat Dragon? Когда выбирать enterprise-решение, а когда бесплатное/opensource?

10. STRIDE vs MITRE ATT&CK — это конкуренты или комплементарные методологии? Обоснуйте ответ. В каком сценарии вы используете обе?

### Ответы на чек-лист

**1. Расшифровка STRIDE.** S — **Spoofing identity** (нарушает **аутентичность**): аналог — подделка ключа замка. T — **Tampering with data** (нарушает **целостность**): незаконное проникновение и порча припасов. R — **Repudiation** (нарушает **неотказуемость**): диверсант отрицает, что это он оставил открытыми ворота. I — **Information disclosure** (нарушает **конфиденциальность**): кража планов обороны. D — **Denial of service** (нарушает **доступность**): враг перекрывает поставки воды и продовольствия замку. E — **Elevation of privilege** (нарушает **авторизацию**): слуга с кухни получает доступ в оружейную. Ключевой нюанс — STRIDE покрывает гексаду CIAAN (Confidentiality, Integrity, Availability, Authenticity, Authorization, Non-repudiation), что шире, чем классическая триада CIA. Компромисс: STRIDE не различает accidental vs malicious угрозы — повреждение данных из-за грозы (accidental integrity loss) тоже попадает в Tampering, но mitigation'ы будут другими (backup, а не HMAC).

**2. STRIDE-per-element.** Каждый элемент DFD имеет предопределённый набор угроз: Process ↔ все 6 категорий, Data Store ↔ Tampering, Information Disclosure, DoS (целостность, конфиденциальность, доступность), Data Flow ↔ Tampering, Information Disclosure, DoS (то же самое, но для данных в движении), External Entity ↔ Spoofing, Repudiation (кто это вообще?), Trust Boundary ↔ ВСЕ 6 — потому что это место, где уровень доверия меняется (DMZ→Internal). Trust Boundary — самый насыщенный угрозами, потому что злоумышленник, пересекающий trust boundary, получает наибольшие возможности: из недоверенной сети в доверенную. Mitigation trust boundary — это «шлюз», где происходит (а) аутентификация (кто ты?), (б) авторизация (что тебе можно?), (в) валидация (payload корректен?), (г) шифрование (TLS termination + re-encryption). Компромисс: STRIDE-per-element на Trust Boundary генерирует много false positives — не всякий элемент trust boundary реально атаковать. Нужен triage.

**3. Repudiation — уникальная категория.** В отличие от остальных (S/T/I/D/E), которые имеют относительно стандартные технические mitigation'ы, Repudiation требует организационно-технических мер: digital signature + tamper-proof logging + dual authorization. Это важно для compliance, потому что (а) SOX — финансовые транзакции должны быть доказательными, (б) PCI DSS Req 10 — все операции с cardholder data должны быть отслежены до конкретного пользователя, (в) GDPR Art 30 — Records of Processing Activities должны быть защищены от tampering. Технически: append-only audit trail (WORM storage, blockchain anchoring), транзакции подписываются ECDSA/ГОСТ Р 34.10, логи хэшируются цепочкой (hash chain — каждая запись содержит хэш предыдущей). Компромисс: tamper-proof logging дороже и сложнее обычного — баланс между стоимостью compliance и риском non-compliance штрафов (4% годового оборота под GDPR).

**4. Оценка риска поверх STRIDE.** STRIDE отвечает «ЧТО может случиться» (what can go wrong), но не «НАСКОЛЬКО это вероятно и НАСКОЛЬКО разрушительно». Восполнение: (а) **CVSS v4.0** — стандартизированный калькулятор severity (Attack Vector, Attack Complexity, Privileges Required, User Interaction, Scope, CIA Impact) → score 0.0–10.0 (None to Critical), используется в CVE database; (б) **DREAD** (Damage, Reproducibility, Exploitability, Affected users, Discoverability) — от Microsoft, каждый параметр 1–10, total / 5 → qualitative risk (Low/Medium/High/Critical), хорошо для ad-hoc threat models, но субъективен; (в) **FAIR** (Factor Analysis of Information Risk) — количественная оценка: Loss Event Frequency × Loss Magnitude = risk exposure в долларах. CVSS — самый портируемый (NVD NIST, Mitre CVE), FAIR — самый бизнес-ориентированный (CIO понимает «$500K annualized loss» лучше, чем «CVSS 7.5»). Компромисс: CVSS быстрее и легче, FAIR точнее, но требует больше maturity данных.

**5. Три ограничения STRIDE.** (1) **STRIDE не моделирует бизнес-логику**: угроза «купить товар, получить скидку лояльности, отменить покупку, скидка осталась» — это abuse case, который STRIDE пропустит, потому что на DFD это легитимный поток данных. Нужны abuse-case-модели (OWASP Abuse Cases). (2) **STRIDE не покрывает privacy**: threat model, требующая оценки «linkability, identifiability» (LINDDUN), — GDPR compliance, медицинские данные. STRIDE может дополнить LINDDUN, но не заменить. (3) **STRIDE — это proactive (design-time)**: он не заменит SAST (статический анализ кода), DAST (динамическое тестирование), penetration testing (manual exploit). STRIDE → threat identification, penetration test → threat validation — комплементарны. Компромисс: организация, которая делает ТОЛЬКО STRIDE, закрывает design-time угрозы, но пропускает implementation-time баги (buffer overflow, race condition). Полный SDLC: STRIDE → SAST → DAST → pentest → red team.

**6. Agile-процесс с STRIDE.** (1) Sprint Planning (вторник): команда выбирает фичу («Payment API»); security champion рисует DFD (ручка + белая доска / Threat Dragon) — 15 минут. (2) STRIDE-per-element проход: по каждому элементу DFD задаём вопрос «Может ли эта категория угроз здесь реализоваться?» — 30 минут. (3) Результат: список из ~20 угроз. Triage: DREAD/CVSS → отсекаем Low; остаётся 5 Medium/High. (4) Medium/High угрозы конвертируются в **security user stories** в Jira: «Как злоумышленник, я могу подделать JWT-токен...», Acceptance Criteria: JWT signature validation + expiration check. (5) Эти user stories попадают в sprint backlog вместе с функциональными (не отдельно!). Definition-of-Done: acceptance criteria security stories выполнены + automated security test passed. Компромисс: на первый взгляд STRIDE добавляет overhead в спринт — но он заменяет ad-hoc «security review», которые всё равно происходят, просто бессистемно и позже (когда исправлять дороже).

**7. Elevation of Privilege: горизонтальное vs вертикальное.** Вертикальное: user → admin (изменение роли через манипуляцию параметром `role=admin`). Горизонтальное: user A → user B (видит чужие данные через IDOR — Insecure Direct Object Reference, `/invoices/123` → `/invoices/124` без проверки, что invoice 124 принадлежит тебе). Пример IDOR: интернет-банк отображает выписку по `/statement/accountId=12345`. Злоумышленник меняет `12345` на `12346` — видит чужие транзакции (Information Disclosure + EoP горизонтальное). Mitigation: **Authorisation on every request** — не полагаться на «скрытость» ID. Сервер проверяет: «принадлежит ли `accountId=12346` аутентифицированному пользователю?» Доступ только к owned-resources. Дополнительно: random/unpredictable IDs (UUID4 вместо autoincrement integer) — obscurity ≠ security, но усложняет enumeration. Компромисс: UUID защищает от enumeration (нельзя пройтись `12345, 12346, 12347...`), но не заменяет authorization check — UUID4 угадать сложно, но если утекает (HTTP Referrer header, logs), IDOR всё равно работает.

**8. STRIDE + Zero Trust.** STRIDE идентифицирует угрозы; Zero Trust реализует mitigation'ы, вытекающие из принципа «никогда не доверять — всегда проверять». Пример microservice A → B: STRIDE-per-element Data Flow: Tampering (изменение payload), Information Disclosure (сниффинг), Spoofing (если злоумышленник выдаёт себя за service A). Mitigations STRIDE: mTLS + HMAC. Zero Trust добавляет: (а) **SPIFFE/X.509 short-lived certificates** — сертификат A действителен 5 минут, скомпрометированный сертификат быстро истекает; (б) **network microsegmentation**: Kubernetes NetworkPolicy deny-all default + разрешить `A → B:443` — даже если сертификат украден, злоумышленник не может lateral movement в service C; (в) **continuous inspection**: Istio AuthorizationPolicy проверяет не только mTLS identity, но и JWT-claims («service A может вызывать `POST /payment`, но service B — только `GET /health`»). STRIDE говорит «ЧТО нужно защищать»; Zero Trust говорит «КАК защищать без слепого доверия к сети/периметру». Компромисс: Zero Trust implementation сложнее и дороже (SPIFFE, Istio mesh, network policies) — для internal low-risk сервисов можно ограничиться STRIDE классическим (mTLS), для high-stakes (платежи, PII) — STRIDE + Zero Trust.

**9. Инструменты STRIDE.** (1) **Microsoft Threat Modeling Tool** (бесплатно, Windows): самый быстрый способ начать, drag-and-drop DFD, авто-STRIDE-генерация. Плюс: встроенная threat property knowledge base (подсказывает mitigation). Минус: шум (много Low/trivial), платформозависим, DFD не экспортируются в стандартные форматы (Visio/OmniGraffle). (2) **OWASP Threat Dragon** (бесплатно, open-source, web): кросс-платформенность, коллаборативное редактирование DFD, REST API. Плюс: community-driven, легче интегрировать в CI/CD. Минус: меньше «из коробки» threat knowledge, чем MTMT. (3) **IriusRisk** (enterprise, платно): continuous threat modelling — correlation с SAST/DAST findings, compliance mapping, API-first. Плюс: «живая» threat model (обновляется, когда меняется код/инфраструктура). Минус: дорого (enterprise licence), сложный onboarding. Выбор: стартап/опенсорс → Threat Dragon, Microsoft-heavy org → MTMT, mature enterprise с compliance-driven risk management → IriusRisk. Компромисс: MTMT быстрее внедрить, но порождает «threat model drift» (DFD устаревает через месяц после создания); IriusRisk поддерживает threat model в актуальном состоянии, но требует DevSecOps maturity.

**10. STRIDE vs MITRE ATT&CK.** Это **комплементарные**, а не конкурирующие инструменты. STRIDE работает на **design phase** (proactive): «Какие угрозы возможны в принципе для этого DFD-элемента?» ATT&CK работает на **threat-actor modelling phase** (proactive+reactive): «Как конкретно APT29 (Cozy Bear) или FIN7 реализуют Elevation of Privilege? ATT&CK T1068 (Exploitation for Privilege Escalation) — вот их технические детали».

Сценарий совместного использования: организация проектирует новую платформу (fintech). Design phase: STRIDE-per-element на DFD — находим 30 угроз, приоритезируем. Threat actor modelling: берём MITRE ATT&CK матрицу для APT38 (Lazarus Group, финансово-мотивированная). Смотрим: APT38 специализируется на Spearphishing Attachment (T1566.001) → Initial Access. На платформе есть External Entity «Email Gateway»; STRIDE говорит — Spoofing/Information Disclosure возможны. ATT&CK добавляет: «APT38 будет фишить через Email Gateway — добавьте email sandboxing (FireEye/Mimecast) как mitigation именно для этого вектора».

Без ATT&CK: mitigation'ы generic (TLS, RBAC). С ATT&CK: mitigation'ы threat-informed, приоритезированы под реальные TTP конкретных APT-групп, которые атакуют вашу индустрию. Компромисс: ATT&CK требует threat intelligence maturity (кто атакует fintech? APT38, Cobalt Group, Silence). Организация без TI-команды всё равно выигрывает от STRIDE — generic mitigation'ы покрывают больше, чем ничего.

## Ключевые выводы для собеседования

**STRIDE — это не список, а система.** Главное, что нужно донести на собеседовании: STRIDE работает не как checklist из шести пунктов («проверили Spoofing? ок, дальше»), а как **матрица STRIDE × DFD-elements**. Именно **STRIDE-per-element** — привязка каждой категории угроз к каждому элементу диаграммы потоков данных — превращает STRIDE из «мнемоники» в воспроизводимый, систематический процесс threat modelling. Senior-специалист понимает, что Data Store не подвержен Spoofing (некого подделывать), а External Entity не подвержена Elevation of Privilege (у неё нет ролей в системе).

**STRIDE — proactive, но не самодостаточен.** STRIDE находит угрозы на этапе проектирования — это shift-left в безопасности. Но организация, останавливающаяся на STRIDE, получает ложное чувство безопасности: STRIDE не найдёт implementation-level vulnerability (buffer overflow, race condition) и не проверит, что mitigation'ы действительно реализованы. Полноценная security assurance требует threat modelling (STRIDE/PASTA) → SAST → DAST → penetration testing → red teaming — и STRIDE здесь — первый, но не единственный шаг.

**STRIDE и стандарты.** STRIDE напрямую маппится на compliance-требования: NIST SP 800-53 SA-11 (Developer Testing and Evaluation — threat modelling required for high-impact systems), ISO 27001 A.8.25 (Secure Development Lifecycle), CIS Control 16 (Application Software Security — threat modelling at design). На собеседовании покажите, что вы понимаете не только технику, но и то, как STRIDE встраивается в governance, risk, and compliance (GRC) framework.

**STRIDE — точка входа, не конечная станция.** STRIDE идеален для организаций, которые только начинают threat modelling journey (зрелость 0 → 1): низкий порог входа, хороший тулинг, быстрый ROI. Но mature security programme со временем добавит PASTA (risk-centric, threat-intelligence-driven), LINDDUN (privacy threats), Attack Trees (red team exercises), continuous threat modelling (IriusRisk, ThreatPlaybook). STRIDE — не конкурент этим методологиям, а фундамент, на котором они строятся. На собеседовании это покажет, что вы мыслите архитектурно: «Я знаю, с чего начать и как эволюционировать программу threat modelling, когда maturity вырастет.»
