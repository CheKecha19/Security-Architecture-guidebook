# Trike: Формальное моделирование угроз через матрицу доступа и анализ требований

## Что такое Trike?

Представьте закрытый научный институт с многоуровневой системой допуска. У каждого сотрудника есть бейдж, определяющий, в какие лаборатории он может войти и с какими документами — работать: лаборант видит только текущий эксперимент, научный руководитель — все эксперименты отдела, а директор — финансовые отчёты всей организации. Пропускная система реализует принцип: доступ — это не право быть в здании, а разрешённое множество действий над защищаемыми объектами. Если лаборант пытается войти в кабинет директора, турникет блокирует проход, и инцидент немедленно логируется.

**Trike** — методология моделирования угроз, которая строит точно такую же «пропускную систему» для программного обеспечения: акторы, активы, разрешённые действия и автоматический детектор нарушений этой модели. Название происходит от «Threat modeling methodology based on requirEments» — фокус на требованиях безопасности, формализованных до уровня, допускающего математическую верификацию. В отличие от STRIDE, где угрозы классифицируются по категориям, Trike формулирует угрозу как **нарушение явно заявленного правила доступа**.

Фундаментальная проблема, решаемая Trike: как доказать, что система безопасна, а не просто перечислить угрозы? STRIDE отвечает: «вот шесть категорий — проверь каждую». Это качественный подход: угрозы найдены, но нет доказательства, что найдены *все*. Trike отвечает: «вот формальная модель доступа — любое отклонение от неё есть угроза». Это quantitative approach: угроза определяется как violation of access model, и completeness модели означает completeness покрытия угроз. Аналогия: STRIDE — это проверка судна по чек-листу («корпус цел? двигатель работает?»); Trike — это построение математической модели плавучести, где любое нарушение equations означает угрозу затопления.

Методология разработана Брендой Ларком и её коллегами как академический ответ на ограниченность категориальных подходов. В отличие от industry-driven методологий (STRIDE создан Microsoft, VAST — IriusRisk), Trike вырос из research-среды и сохраняет академическую строгость: она требует от пользователя не security experience, а способность мыслить в терминах формальных систем — акторов, разрешений, инвариантов доступа. Это её сила (доказуемость) и её слабость (барьер входа).

## Эволюция и мотивация

К середине 2000-х годов threat modeling как дисциплина упёрлась в фундаментальное ограничение: completeness. STRIDE, при всей его распространённости, не мог ответить на вопрос «все ли угрозы найдены?», потому что шесть категорий — это guide, не completeness guarantee. PASTA добавила бизнес-risk dimension, но осталась качественной. Attack trees дали structured decomposition, но построение дерева — творческий процесс аналитика, completeness не guaranteed.

Два индустриальных тренда подтолкнули развитие формальных подходов. Первый: **рост сложности систем до уровня, где intuitive review fails**. Когда система имеет 50 ролей пользователей, 200 активов и 15 типов операций, количество потенциальных взаимодействий — 50 × 200 × 15 = 150,000. Человек физически не может проверить каждую комбинацию на нарушение безопасности — нужен systematic approach. Второй: **compliance pressure** — регуляторы (особенно в defence и finance) начали требовать не просто «мы проверили», а «мы доказали». Стандарт Common Criteria (ISO 15408) использует formal security models на высших уровнях Evaluation Assurance Level (EAL); Trike — попытка принести тот же уровень строгости в software threat modeling.

Продолжим аналогию с научным институтом. В 1990-х годах достаточно было, чтобы охранник на входе проверял бейджи визуально — intuitive review. К 2010-м организация разрослась до 5000 сотрудников, 200 лабораторий и 40 уровней доступа. Визуальная проверка означает: охранник видит бейдж «лаборатория 42, уровень 3» и должен мгновенно вспомнить, разрешён ли этот уровень в этой лаборатории — combinatorial explosion. Trike — это центральный сервер авторизации, который получает запрос «сотрудник X, дверь Y, операция Z» и выдаёт binary ответ на основе матрицы доступа. Ошибки исключены — система не «думает», она делает lookup.

Критическая связь со стандартами: **NIST SP 800-53**, контроль **AC-1** (Access Control Policy and Procedures) и особенно **AC-6** (Least Privilege), требуют от организаций формального определения access control policy. Trike переводит это требование из narrative в actionable: вместо текстового описания политики (которое ambiguous) строится матрица доступа (которая machine-checkable). **CIS Controls v8**, контроль **5.1** (Account Inventory and Management), предписывает инвентаризацию и управление привилегированными аккаунтами — матрица доступа Trike является одновременно inventory и policy enforcement blueprint.

## Зачем это нужно?

### Сценарий 1: Банковская система с многоуровневым RBAC

Корпоративный банк обслуживает 200 клиентов-организаций, каждая из которых имеет до 20 пользовательских ролей (бухгалтер, финансовый директор, аудитор, администратор). Система обрабатывает 15 типов транзакций с разными уровнями approval. Ad-hoc описание прав доступа («бухгалтер может инициировать платёж до $10K») неизбежно содержит ambiguity: что значит «инициировать»? Может ли он видеть историю платежей? Может ли редактировать черновик?

Trike строит трёхмерную матрицу: актор (роль) × актив (тип транзакции) × действие (create, read, update, approve, void). Для каждой ячейки — explicit allow/deny. Затем методология автоматически вычисляет threat surface: любые комбинации, где deny может быть обойдён, либо где allow создаёт unsafe delegation chain (бухгалтер create, CFO approve — что если approve без review?)

**Бизнес-риск**: unauthorised transaction → financial loss + регуляторный штраф (SOX, Basel III). **Стандарт**: **ISO 27001 Annex A.9.2.1** (User Access Provisioning) требует granular access control — матрица Trike предоставляет гранулярность до single cell.

### Сценарий 2: Медицинская информационная система с требованиями HIPAA

Больничная система содержит Protected Health Information (PHI) 500,000 пациентов. Врачи, медсёстры, администраторы, billing department, researchers — каждая роль имеет разные права на разные типы данных (диагнозы, назначения, счета, исследовательские срезы).

Trike формализует HIPAA minimum necessary principle: для каждой роли определяется минимальное множество активов и действий, необходимых для работы. Угроза — любое превышение этого минимума. Compliance evidence генерируется автоматически: для каждого пациента система может показать, какие роли имели к нему доступ и на каком основании (ячейка матрицы).

**Бизнес-риск**: HIPAA violation → $50K-$1.5M per record штраф. **Стандарт**: **Health Insurance Portability and Accountability Act (HIPAA) Privacy Rule** §164.514(d) — minimum necessary standard — напрямую отображается на матрицу доступа Trike.

### Сценарий 3: Defence-система с multi-level security (MLS)

Военная система управления обрабатывает данные с грифами Unclassified, Confidential, Secret, Top Secret. Пользователи имеют clearance levels. Потоки данных должны соблюдать Bell-LaPadula принципы: no read up, no write down.

Trike строит lattice-based access model: акторы (clearance level) × активы (classification level) × действия (read, write). Rules engine Trike автоматически детектирует violations: пользователь Confidential читает Secret (read up — violation), пользователь Secret пишет в Confidential (write down — violation, потенциальная утечка).

**Стандарт**: **NIST SP 800-53, AC-3** (Access Enforcement), явно требует enforcement mandatory access control в defence системах. Trike обеспечивает model-level enforcement.

### Сценарий 4: Платформа с динамическими разрешениями и делегированием

B2B SaaS-платформа позволяет администраторам клиента создавать кастомные роли и делегировать права. Динамическое переопределение доступа невозможно описать статическим документом — модель должна пересчитываться при каждом изменении.

Trike с formal model позволяет: при добавлении новой роли перегенерировать threat surface автоматически, детектировать indirect privilege escalation (роль A делегирует read к активу X роли B, роль B делегирует update к X роли C — C получает write через transitive delegation). **NIST SP 800-53, AC-4** (Information Flow Enforcement), требует контроля цепочек информации — Trike делает transitive relationships explicit.

## Основные концепции

### Модель «субъект-объект-действие»: формальный каркас

В основе Trike лежит трёхмерная матрица доступа: **Actor × Asset × Action → {Allow, Deny}**. Это расширение классической модели Харрисона-Руззо-Ульмана (HRU), адаптированное под практические нужды threat modeling.

**Актор (Actor)** — любая сущность, инициирующая взаимодействие: пользователь, роль, внешняя система, внутренний сервис, automation job. Важно, что актор — это не обязательно human: scheduled task, списывающая проценты в 3 AM, — тоже актор, и её права должны быть в матрице. **Актив (Asset)** — любой защищаемый объект: файлы, записи БД, API endpoints, системные ресурсы, configuration objects. Trike отличает data assets (резюме, платёжные реквизиты) от functional assets (возможность approve транзакцию, возможность deploy в production). **Действие (Action)** — операция, выполняемая актором над активом: базовый CRUD расширяется до domain-specific операций: execute, approve, delegate, export, configure, delete-audit-log.

Формальный аспект: каждая ячейка матрицы имеет explicit value: Allow или Deny. Default deny principle реализуется тем, что все не-explicitly-allowed комбинации — Deny. Это соответствует **CIS Controls v8 5.4** (Use of Dedicated Administrative Accounts) и **OWASP ASVS V4.1.1** — приложение должно верифицировать, что access control enforced на доверенной стороне.

### Угроза как violation of access model

Trike переопределяет понятие угрозы. В STRIDE угроза — это нежелательное событие, классифицируемое по категории. В Trike угроза — **любое состояние системы, в котором выполняется denied операция, либо состояние, в котором allowed операция создаёт транзитивное privilege escalation**.

Это даёт принципиальное преимущество: completeness. Если матрица доступа полна (все акторы × все активы × все действия определены), то множество угроз — это ровно те denied комбинации, для которых существует техническая возможность выполнения. Trike не пропустит угрозу, потому что угроза определена через модель, а не через эвристику.

Но здесь же — ключевой компромисс. Качество модели определяет качество угроз: если аналитик пропустил актора (не учёл фоновый сервис), Trike «не увидит» угрозы от этого актора — false negative происходит на уровне модели, а не на уровне анализа. Trike не «думает» за аналитика; он заставляет аналитика думать structured, но ошибки в модели — ошибки в security. Это принципиальное отличие от rules-based подходов (VAST), где ошибка — в rules catalogue, и от экспертных подходов (STRIDE), где ошибка — в attention.

### Анализ требований: traceability от threat к requirement

Уникальная черта Trike — backward traceability. Поскольку модель строится на основе требований безопасности, каждая угроза трассируема до конкретного security requirement, которое она нарушает. Например: «Requirement R-14: Patient Billing Data must be accessible only by Billing Department. Threat T-28: Nurse role can Read Patient Billing Data через shared API endpoint. Root cause: отсутствие row-level security на API, отсутствие проверки role claim в JWT.»

Эта traceability критична для compliance: аудитор ISO 27001 хочет видеть не просто «мы нашли угрозы», а «каждое требование Annex A покрыто threat model, и для каждого uncovered требования есть risk acceptance». Trike генерирует coverage matrix: Security Requirement × Threat → Status. **ISO 27001 Annex A.8.2** — именно этот маппинг между requirements и risks является evidence для сертификации.

### Декомпозиция модели: от архитектуры к CRUD

Trike применяет иерархическую декомпозицию системы. На верхнем уровне — high-level архитектурные компоненты (веб-фронтенд, API-шлюз, сервис заказов, база данных). Для каждого компонента определяется: его собственные акторы (кто к нему обращается), его активы (что он защищает) и его граница доверия (где заканчивается его control — после этого начинается trust in transit layer).

На нижнем уровне для каждого компонента Trike строит CRUD-матрицу: для каждой пары актор × asset определено, какие из операций C/R/U/D разрешены. Это позволяет детектировать нарушения принципа **least privilege** — если актору разрешён Update там, где ему нужен только Read, это threat (privilege creep).

Архитектурная проблема: для системы из 100 компонентов, 50 акторов и 30 активов количество ячеек матрицы — 100 × 50 × 30 × 4 = 600,000. Заполнять вручную — нереалистично. Trike предполагает автоматизацию через: наследование прав от ролей (role-based grouping), дефолтные политики на уровне trust zone (все сервисы из DMZ имеют deny к internal DB), inference engine (если сервис A вызывает сервис B, и у B есть права на asset X, то A должен иметь explicit delegation).

## Сравнение с другими механизмами

| Критерий | Trike | STRIDE | Отсутствие threat modeling |
|----------|-------|--------|----------------------------|
| Модель безопасности | Формальная access matrix, completeness | Эвристическая по 6 категориям, best-effort | Нулевая — security emergent |
| Сложность внедрения/поддержки | Очень высокая (формальный анализ) | Низкая (нужен эксперт) | Нулевая (ничего не делается) |
| Производительность при масштабировании | Хорошая (матрица machine-processable) | Плохая (expert time linear to components) | N/A |
| Доказуемость completeness | Высокая (полнота модели = полнота угроз) | Низкая (нет гарантии) | Отсутствует |
| Требование к компетенциям | Formal methods, access control theory | Security domain knowledge | Нет |
| Типичные сценарии | High-assurance, defence, compliance-heavy | General-purpose application security | Startups, legacy, immature orgs |

STRIDE эффективен в ситуациях, где completeness — nice-to-have, а не mandatory. Trike эффективен там, где цена пропущенной угрозы неприемлема. Разница — как между code review «прочитай и подумай» и формальной верификацией через model checking: первое — практично для большинства, второе — необходимо для mission-critical.

Скрытые издержки Trike: onboarding cost — аналитик без формального образования потратит недели на понимание концепции completeness модели; maintenance cost — при каждом изменении архитектуры матрица пересчитывается, что требует либо автоматизации (инвестиции в тулинг), либо ручного труда (дорого); psychological resistance — разработчики воспринимают формальные модели как «теоретическую бюрократию» и саботируют.

«Серебряной пули» нет: Trike прекрасен для high-assurance систем, но для B2C стартапа с 8 инженерами и weekly release cycle он создаст больше friction, чем value. Выбор между Trike и STRIDE — это выбор между «доказать» и «достаточно проверить»; ответ зависит от risk appetite и regulatory landscape. **NIST SP 800-30** предписывает выбирать methodology proportional to assessed risk — это явно указывает на контекстную природу выбора.

## Уроки из реальных инцидентов: Capital One Breach (2019) как сценарий нарушения модели доступа

**Хронология**: в марте 2019 года злоумышленник, бывший сотрудник AWS, эксплуатировал Server-Side Request Forgery (SSRF) в WAF-модуле, чтобы получить credentials IAM-роли, прикреплённой к EC2-инстансу. Роль имела права на List и Read всех S3-бакетов организации. Результат: эксфильтрация 106 миллионов записей кредитных заявок (personal data, SSN, bank account numbers) из S3-бакетов Capital One.

**Корневая причина**: IAM-роль (актор) имела права Read на S3-бакеты с customer data (актив), хотя сервису, на котором она была прикреплена, требовался только доступ к своим конфигурационным файлам. В терминах Trike: в матрице доступа была ячейка «WAF EC2 role × Customer Data S3 bucket × Read → Allow», которая должна была быть «Deny». Это классический **violation of least privilege** — разрыв между required access и granted access.

**Последствия**: $80M штраф от OCC, $190M settlement по class-action, отставка CISO, падение акций на 6%. Capital One также понёс $150M оценочно на remediation: миграцию в new AWS accounts, полный аудит IAM policies, внедрение Cloud Security Posture Management.

**Как Trike изменил бы исход**: методология построила бы access matrix на этапе архитектурного дизайна cloud-инфраструктуры. Для IAM-роли WAF-сервера была бы явно определена строка матрицы: какие S3-бакеты она может читать. Rules engine Trike детектировал бы, что IAM policy grants broader permissions, чем access matrix — это threat flagged как «IAM Privilege Escalation». До деплоя архитектор получил бы алерт и скорректировал IAM policy — принцип minimum necessary был бы enforced на model level.

**Урок**: threat model must include infrastructure layer — не только код приложения, но и cloud permissions, сетевые политики, IAM policies. Trike с его формальной моделью доступа идеален именно для cloud security: cloud — это giant permissions matrix (S3 bucket policies, IAM roles, security groups, VPC endpoints), и формальная модель Trike детектирует over-permissioning систематически. Стандарт **NIST SP 800-53, AC-6 (Least Privilege)** требует explicit scoping прав — модель Trike и есть formal scoping.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| Formal modeling tool (Trike tooling, custom spreadsheets) | Построение и анализ access matrix | Design phase | NIST 800-53 AC-1, AC-6, ISO 27001 A.9.2.1 |
| IAM Policy Analyzer (AWS IAM Access Analyzer, Azure Policy) | Верификация cloud permissions против модели | Pre-deployment gate | CIS Controls v8 5.4, NIST 800-53 AC-3 |
| Access Control Testing Framework | Автоматическая проверка denied операций (должны быть заблокированы) | CI/CD, QA | OWASP ASVS V4.1.1, V4.1.2 |
| Dependency & Flow Analyzer | Построение графа вызовов между сервисами для transitive trust analysis | Architecture review | NIST 800-53 AC-4 |
| Compliance Mapping Engine | Трассировка требований (ISO, PCI, HIPAA) на ячейки access matrix | Continuous compliance | ISO 27001 A.8.2, PCI DSS Req 7 |

Ключевой вызов инструментального ландшафта Trike — неразвитость. В отличие от экосистемы STRIDE (Microsoft TMT, Threat Dragon, OWASP pytm) и VAST (IriusRisk, коммерческие rules engines), Trike-инструментарий — это в основном академические прототипы и кастомные решения (Excel, Google Sheets с макросами). Это создаёт operational overhead: аналитик не просто строит модель, а одновременно строит инструмент её обработки. Практический компромисс: использовать Trike-принципы (формальная access matrix) с инструментами других методологий (DFD в Microsoft TMT, rules engine в IriusRisk) — гибридный подход, где Trike даёт теоретический framework, а industry tools — автоматизацию.

Операционные метрики: **access model completeness** (процент ячеек матрицы, заполненных explicit allow/deny), **least privilege violation count** (количество ячеек, где granted > required), **transitive trust paths** (количество цепочек длиной > 2, где доступ может быть получен косвенно), **policy drift** (расхождение между access matrix и реальными IAM/IAM-политиками в production). **NIST SP 800-55** рекомендует outcome-oriented metrics — Trike с его formal model выдаёт именно outcome (is the access matrix enforced?), а не activity (how many reviews проведено?).

## Архитектурные решения

Встраивание Trike в **Defense-in-Depth**: access matrix — это blueprint для policy enforcement на всех слоях. На network layer: матрица определяет разрешённые потоки между trust zones → security groups и firewall rules генерируются из матрицы. На application layer: матрица определяет разрешённые CRUD операции на API endpoints → API gateway policies (OAuth scopes, claims) генерируются из матрицы. На data layer: матрица определяет доступ к таблицам и колонкам → row-level security, column masking автоматизированы. Ключевой architecture decision: модель доступа одна, enforcement разный — но оба должны быть synchronised.

Взаимодействие с **Zero Trust**: Trike и Zero Trust — естественные союзники. Zero Trust требует explicit access decision для каждого запроса; Trike даёт formal definition этих decisions. Access matrix Trike — это policy decision point (PDP) в Zero Trust архитектуре, а enforcement points (API gateway, service mesh sidecar, DB proxy) — это policy enforcement points (PEP). Проблема синхронизации: матрица должна быть machine-readable и потребляться enforcement tools. Open Policy Agent (OPA) с Rego-политиками — мост между Trike-моделью и runtime enforcement. **NIST SP 800-207** требует dynamic access decisions — Trike обеспечивает decision logic.

Проблемы масштабирования: access matrix для enterprise-системы из 500 микросервисов становится combinatorial explosion (миллионы ячеек). Решение — hierarchical matrices: верхний уровень описывает trust zone interactions, средний — service-to-service, нижний — CRUD на endpoints. Наследование deny: если на верхнем уровне запрещено, нижний уровень не анализируется. Это редуцирует матрицу с O(n³) до O(n log n). Компромисс: granularity пожертвована ради manageability — но для compliance (например, PCI DSS segmentation) верхнеуровневой матрицы достаточно.

Миграция с legacy: организация с decade of ad-hoc access management имеет implicit matrix, закодированную в конфигурациях, а не задокументированную. Trike-внедрение стартует с reverse-engineering: просканировать все IAM policies, firewall rules, API endpoint authorisation checks и построить as-is матрицу. Затем провести gap analysis: где as-is разрешений больше, чем to-be (требования безопасности). Затем plan remediation: сужать разрешения итерациями, чтобы не сломать production (change management риск).

**Красный флаг деградации**: access matrix не обновлялась > 1 sprint (для agile) / > 1 quarter (для enterprise). Причина: команды добавляют новые endpoints и роли, но не обновляют матрицу → drift между моделью и реальностью → модель перестаёт быть trust source → threat modeling loses credibility. Автоматизации ради: CI/CD pipeline должен детектировать drift — если появился новый IAM role, не mapped to access matrix, билд блокируется с требованием обновить матрицу. Это жёсткое требование, но без него Trike вырождается в shelfware.

## Подготовка к собеседованию: Common pitfalls & key questions

**Ошибка 1**: «Trike — это просто Excel с ролями». Кандидат не понимает completeness guarantee. Сильный ответ: «В отличие от ad-hoc listing ролей, Trike строит formal model, где угроза определена как violation of explicit allow/deny. Полнота модели = полнота покрытия угроз. Это critical difference для high-assurance систем, где пропущенная угроза — это не баг, а certification failure. Связь с Common Criteria EAL 5-7, где формальная модель доступа — требование.»

**Ошибка 2**: приравнивание Trike к RBAC. RBAC — механизм enforcement, Trike — методология threat modeling. Разница: RBAC говорит «роль X имеет разрешение Y»; Trike говорит «роль X должна иметь разрешение Y, и если enforcement позволяет Z — это угроза». Интервьюер проверяет, понимает ли кандидат разницу между security mechanism и security assurance. Сильный ответ: «RBAC — runtime control, Trike — design-time verification. Trike валидирует, что RBAC (или ABAC, или ACL) настроен корректно и не имеет privilege escalation paths. Контроль AC-6 (Least Privilege) NIST 800-53 требует и mechanism (RBAC), и verification (Trike).»

**Ошибка 3**: игнорирование combinatorial explosion как operational blocker. Интервьюер спрашивает «сколько ячеек в вашей access matrix?» чтобы проверить, думал ли кандидат о scalability. Развёрнутый ответ: «При 200 акторах, 300 активах и 5 действиях — 300,000 ячеек. Вручную — нереалистично. Стратегия: hierarchical model (role groups, asset classes, action sets) редуцирует до ~2,000 ячеек; automation заполняет default deny; аналитик задаёт только exceptions. Плюс tool-based generation из infrastructure-as-code — Terraform, CloudFormation парсятся в access matrix автоматически. CIS Controls v8 5.4 говорит о 'automated account management' — в контексте Trike это automated access matrix generation.»

**Ситуационный вопрос**: «Как вы докажете регулятору, что access matrix полна, если система развивается ежедневно?» Ожидание: кандидат должен предложить process, а не one-time artefact. Ответ: «Полнота достигается не one-time анализом, а continuous verification. CI/CD пайплайн: 1) каждый PR, затрагивающий авторизацию, должен включать обновление access matrix fragment; 2) nightly job парсит production IAM policies, firewall rules, API endpoint annotations и сравнивает с матрицей; 3) drift report идёт в Slack security channel. Регулятору предоставляется trend full matrix completeness — не snapshot вчерашнего дня, а proof, что completeness — maintained property. ISO 27001 Annex A.8.2 требует непрерывного risk assessment — continuous verification Trike именно это и обеспечивает.»

## Чек-лист понимания

- [ ] Почему STRIDE не гарантирует completeness покрытия угроз, и как Trike пытается эту проблему решить?
- [ ] Как access matrix Trike взаимодействует с Zero Trust архитектурой, и в какой точке они могут конфликтовать?
- [ ] В каком случае формальная access matrix Trike создаёт false sense of security хуже, чем его полное отсутствие?
- [ ] Как combinatorial explosion ячеек матрицы угрожает практической применимости Trike, и какие архитектурные паттерны редуцирования существуют?
- [ ] Почему traceability от threat к security requirement — это конкурентное преимущество на аудите ISO 27001?
- [ ] Как Trike обрабатывает сценарий, в котором allowed операция создаёт transitive privilege escalation, и какой стандарт это покрывает?
- [ ] В каком случае Trike будет выдавать больше угроз, чем STRIDE, и почему это операционная проблема?
- [ ] Как эволюция IaaS/PaaS повлияла на релевантность Trike — стала ли методология более или менее востребованной?
- [ ] Почему организации, внедрившие Trike, часто возвращаются к STRIDE, и как этого избежать архитектурно?
- [ ] Как false negative на уровне модели (пропущенный актор в матрице) влияет на security posture, и кто отвечает за эту ошибку?

### Ответы на чек-лист

1. **STRIDE не гарантирует completeness**, потому что это категориальный guide, а не formal model. Шесть категорий покрывают common threat types, но эксперт может пропустить угрозу внутри категории (например, идентифицировать spoofing, но не детектировать конкретный spoofing-path). Trike решает это: угроза определяется через access matrix — completeness модели = completeness угроз. Компромисс: Trike completeness гарантирована только при условии, что модель полна (все акторы, активы, действия учтены). Ошибка в модели → потеря completeness guarantee. Стандарт Common Criteria, класс ACM (Security Architecture), требует completeness threat analysis — Trike mapping на ACM — это evidence.

2. **Trike + Zero Trust**: access matrix Trike — это blueprint для Zero Trust policy engine. В Zero Trust каждый запрос проходит explicit verification; access matrix даёт эталон, с которым сверяется запрос. Конфликт: Zero Trust требует runtime enforcement, Trike — design-time analysis. Если access matrix не синхронизирована с runtime (policy drift), Zero Trust enforcing outdated matrix — хуже, чем без матрицы, потому что создаёт illusion of control. Архитектурный bridge: Infrastructure-as-Code генерирует и access matrix (для Trike), и enforcement policies (для OPA/IdP) из одного source of truth. NIST 800-207 explicitly требует synchronised policy — это и есть Trike-to-ZT bridge.

3. **False sense of security**: access matrix полна на бумаге, но не соответствует реальным enforcement points (API gateway не интегрирован, IAM policies живут отдельно, разработчики обходят матрицу для ускорения). Организация показывает аудитору matrix, но реальный access — шире. Это хуже, чем отсутствие, потому что без матрицы risk — acknowledged; с неактуальной матрицей risk — hidden. Детектирование: continuous validation — periodic penetration testing on denied cells (попытаться выполнить denied операцию — должна быть заблокирована). NIST 800-53, CA-7 (Continuous Monitoring) требует ongoing validation controls — это применимо к access matrix.

4. **Combinatorial explosion**: система из 200 акторов × 500 API endpoints × 5 операций = 500,000 ячеек. Ручное заполнение — weeks to months, к моменту завершения модель устарела. Паттерны редуцирования: hierarchical grouping (roles → permission sets → assets classes — заполняется на верхнем уровне, наследуется на нижний), default-deny with explicit allow-list (аналитик задаёт только allow-ячейки, all else — deny), tool-assisted inference (service-to-service call graph → transitive trust matrix генерируется автоматически). Компромисс: редуцирование снижает детализацию — некоторые edge-case угрозы могут быть скрыты группировкой. CIS Controls v8 5.1 балансирует manageability и security.

5. **Traceability — competitive advantage**: аудитор ISO 27001 Annex A.8.2 проверяет, что все риски идентифицированы, оценены и связаны с активами. Trike даёт bidirectional trace: requirement → asset → access rule → threat → risk → mitigation. Аудитор может выбрать random ячейку матрицы «patient data × billing role × delete» и запросить evidence, что это deny enforced. Ответ «вот test case, доказывающий блокировку» — закрывает finding за минуты. Без traceability — ручная реконструкция, занимающая дни. ISO 27001 Annex A.8.2 требует documented risk assessment — Trike генерирует documentation as by-product.

6. **Transitive privilege escalation**: роли A allowed read asset X, роли B allowed copy X to Y (где Y — public). По отдельности правила безопасны; вместе — утечка данных. Trike детектирует через построение transitive closure матрицы: для каждой allowed операции engine вычисляет reachable assets через цепочки операций. Если reachable set содержит asset класса «confidential» в зоне «public» — угроза. Стандарт: NIST 800-53 AC-4 (Information Flow Enforcement) требует контроля flow across trust boundaries — transitive closure Trike и есть flow enforcement на model level.

7. **Больше угроз, чем STRIDE**: Trike флагует все denied ячейки матрицы как потенциальные угрозы — включая те, где техническая возможность exploit отсутствует. STRIDE-эксперт отфильтрует нереалистичные угрозы интуитивно. Trike без additional risk qualification (likelihood, техническая сложность) генерирует «шум» — threat overload. Операционная проблема: команда получает 5000 угроз, не знает, с каких начинать, отключает Trike. Решение: threat qualification layer — каждая угроза оценивается по likelihood (техническая достижимость), impact (чувствительность актива) и existing mitigations — Trike должен быть не просто threat generator, а threat × risk modeler. NIST SP 800-30 требует risk-based prioritization — Trike без likelihood — incomplete.

8. **Эволюция IaaS/PaaS**: cloud сделал Trike более востребованным. IaaS-поверхность — это гигантская permissions matrix (IAM + security groups + bucket policies). Trike — родная модель для анализа этой поверхности. PaaS (Lambda, Cloud Run) добавили short-lived, auto-scaled акторов, требующих dynamic access model — Trike с continuous generation из IaC справляется. Вывод: Trike, казавшийся академическим излишеством в 2005, стал практически необходимым в cloud-native 2025 из-за permission complexity. Cloud Security Alliance Cloud Controls Matrix (CCM) explicitly maps на NIST — Trike bridges CCM и access model.

9. **Возврат к STRIDE**: организации внедряют Trike, сталкиваются с: complexity (access matrix требует formal skills), maintenance burden (команды не обновляют матрицу), tooling gap (нет enterprise-grade Trike tools). Через 6-12 месяцев matrix устарела, process broken, возврат к STRIDE как к «просто работает». Как избежать: start small (один high-value сервис), prove value через compliance audit success, invest в tooling (интеграция Trike с IaC — Terraform policies генерируют матрицу автоматически), hire formal methods specialist. NIST 800-53, PM-10 (Security Authorization) process требует sustained effectiveness — это означает, что без maintenance Trike не certification-worthy.

10. **False negative на уровне модели**: пропущенный актор (например, background job, не учтённый в матрице) — его действия не анализируются Trike. Если этот актор имеет excessive permissions в production (как в Capital One breach — SSRF-exposed IAM role), Trike не детектирует, потому что актора нет в модели. Ответственность: model owner (security architect) — это его ошибка в scoping. Mitigation: cross-validation — continuous monitoring сравнивает все IAM principals из cloud provider в production с акторами в access matrix; unmatched principals — immediate finding. Shared Responsibility Model cloud security — организация отвечает за permissions configuration; Trike помогает не допустить misconfiguration, но не заменяет monitoring.

## Ключевые выводы для собеседования

- **Trike — единственная threat modeling методология, которая предлагает completeness guarantee**: если access matrix полна и корректна, множество угроз полно и корректно. Это качественное отличие от STRIDE и VAST, критичное для high-assurance систем. Цена — высокий барьер входа: formal modelling skills, time investment, tooling gap.
- **Главный компромисс**: assurance level vs agility. Trike даёт доказуемость — за счёт длительности и сложности анализа. STRIDE даёт скорость — за счёт incompleteness. Выбор определяется regulatory landscape и risk appetite — Common Criteria EAL 5+ требует Trike-уровня формальности; SaaS-стартап — STRIDE-уровня.
- **Связь с NIST SP 800-53**: AC-1, AC-3, AC-4, AC-6 — все access control controls map на access matrix Trike. Capital One breach — textbook пример failure AC-6 (least privilege). Trike предотвращает такие failure через explicit model vs actual permissions comparison.
- **Связь с ISO 27001**: Annex A.8.2 (risk assessment) + A.9.2.1 (access provisioning) = Trike coverage. Traceability Trike (requirement → threat → risk → mitigation) генерирует evidence, который аудитор не может оспорить — это competitive advantage на сертификации.

---
_Статья создана на основе анализа академических публикаций по Trike (Brenda Larcom et al.), материалов NIST SP 800-53/800-207, CIS Controls v8, OWASP ASVS v4, ISO 27001:2022, Common Criteria ISO 15408, public post-mortem Capital One Breach (2019) и отчётов Cloud Security Alliance._
