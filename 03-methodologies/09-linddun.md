# LINDDUN: Анализ угроз приватности

Представьте, что вы владелец банковского хранилища. Ваш консультант по безопасности, STRIDE, прекрасно знает, как защитить сейф от взлома: он проверяет замки, сигнализацию, стальные двери. Но однажды вы замечаете, что каждый вечер один и тот же неприметный человек заходит в хранилище, проводит там ровно 12 минут и выходит — причём он вообще не прикасается к деньгам. Ваш консультант пожимает плечами: «Сейфы целы, тревога не срабатывала — угрозы нет». Однако вы понимаете: кто-то методично наблюдает за вашими операциями, строит профили, выявляет закономерности. Это и есть разрыв между безопасностью и приватностью. LINDDUN — тот самый консультант, который поднимает голову от замков и спрашивает: «Кто наблюдает, что связывается, чему не препятствуют?»

## Что такое LINDDUN

**LINDDUN** — это методология систематического анализа угроз приватности, разработанная исследовательской группой DistriNet (KU Leuven, Бельгия) под руководством Kim Wuyts и Wouter Joosen. Впервые представлена в техническом отчёте CW 675 в 2014 году как LIND(D)UN — privacy threat tree catalog, а в 2015 году издана как полноценный tutorial (CW 685): "LINDDUN privacy threat modeling: a tutorial". Аббревиатура LINDDUN раскладывается на семь категорий угроз: Linkability (связывание), Identifiability (идентифицируемость), Non-repudiation (неотказуемость), Detectability (обнаружимость), Disclosure of information (раскрытие информации), Unawareness (неосведомлённость), Non-compliance (несоответствие). В отличие от STRIDE — классической модели угроз безопасности Microsoft, LINDDUN специально спроектирована для анализа угроз приватности в программных системах.

Принципиальное отличие заложено в самом фундаменте: STRIDE спрашивает «как злоумышленник может скомпрометировать систему», а LINDDUN — «как сама система, даже работая корректно, может нарушить приватность пользователей». Это напоминает разницу между оценкой прочности моста (STRIDE) и анализом того, как часто и с какой целью по нему проходят люди (LINDDUN): мост может быть инженерно совершенным, но при этом стать инструментом массовой слежки.

**Privacy-by-design** (приватность в архитектуре) — философский фундамент, на котором покоится LINDDUN. Концепция, сформулированная Ann Cavoukian в 1990-х годах, утверждает, что приватность не должна быть «надстройкой» над готовым продуктом — она должна встраиваться на уровне архитектурных решений. LINDDUN операционализирует privacy-by-design, превращая абстрактный принцип в воспроизводимый инженерный процесс: модель → угрозы → митигация.

## Эволюция LINDDUN

Эволюция LINDDUN отражает траекторию развития самой дисциплины privacy engineering: от академического фреймворка к инструменту, признанному NIST и OWASP. Этапы развития:

2014–2015: рождение. Kim Wuyts, Riccardo Scandariato, Wouter Joosen публикуют LIND(D)UN privacy threat tree catalog (CW 675, сентябрь 2014) и через год — полноценный tutorial (CW 685, июль 2015). Фреймворк содержит деревья угроз для каждой из семи категорий — иерархические структуры, позволяющие аналитику систематически спускаться от общей категории к конкретным сценариям реализации угрозы.

2018: LINDDUN GO. Kim Wuyts, Laurens Sion и Wouter Joosen публикуют "LINDDUN GO: A Lightweight Approach to Privacy Threat Modeling" (EuroSPW 2020, но разработка с 2018). Это карточная игра из 33 карт угроз, превращающая анализ приватности в структурированный мозговой штурм. Революция здесь — снижение порога входа: если PRO требует DFD, mapping table и threat trees, то GO достаточно эскиза системы и колоды карт.

2024: обновлённые LINDDUN GO threat cards — полностью переработанная колода с расширенными фокусными областями приватности, улучшенным руководством и примерами.

2025–2026: LINDDUN MAESTRO — архитектурный фреймворк для privacy threat modeling, опубликованный в Software and Systems Modeling (Springer, ноябрь 2025). MAESTRO использует enriched DFD (обогащённую DFD) для более точного выявления угроз, позволяя проводить threat-specific анализ.

Это эволюция от каталога угроз → к карточной игре → к архитектурному фреймворку. LINDDUN PRO остаётся «золотой серединой»: систематический, exhaustive подход с опорой на DFD, mapping table и threat trees, совместимый с STRIDE (можно переиспользовать одну DFD для обоих анализов).

## Зачем нужен LINDDUN

Представьте себе фармацевтическую компанию, разрабатывающую приложение для мониторинга диабетиков. STRIDE-анализ покажет: данные шифруются при передаче (защита от Tampering), сервер аутентифицирует клиент (против Spoofing), API rate-limited (против Denial of Service). С точки зрения безопасности — идеально. Но LINDDUN задаст другие вопросы:

Почему приложение запрашивает геолокацию, если достаточно показаний глюкометра? Это **Data Disclosure** — избыточный сбор данных.
Можно ли по паттерну запросов к API определить, что пользователь — диабетик, даже не читая сами показания? Это **Detectability** — дедукция участия через наблюдение.
Информирован ли пользователь о том, что его данные агрегируются для «аналитики рынка»? Это **Unawareness** — недостаточная прозрачность.

**GDPR** (General Data Protection Regulation, Regulation EU 2016/679) в статье 5 устанавливает семь принципов обработки персональных данных: законность, справедливость и прозрачность; ограничение цели; минимизация данных; точность; ограничение хранения; целостность и конфиденциальность; подотчётность. LINDDUN проецирует каждый из этих принципов на техническую архитектуру системы, выявляя места, где архитектурное решение вступает в конфликт с GDPR.

**ISO 27001** (Information Security Management Systems) в Annex A.8 (Asset Management) и A.18 (Compliance) требует, чтобы организация идентифицировала применимые законодательные и регуляторные требования — включая требования приватности. LINDDUN предоставляет структурированный процесс такой идентификации для программных систем.

**NIST SP 800-53** (Security and Privacy Controls) в Revision 5, Appendix J, содержит целое семейство privacy controls (PT — Personally Identifiable Information Processing and Transparency). LINDDUN выступает мостом между архитектурным анализом и выбором конкретных NIST-контролов: угроза Linking → контроль PT-3 (Purpose Specification), угроза Data Disclosure → контроль PT-5 (Data Minimization).

**NIST Privacy Framework** (Version 1.0, 2020) структурирован вокруг пяти функций: Identify-P, Govern-P, Control-P, Communicate-P, Protect-P. LINDDUN занимает место в Control-P и Protect-P: выявление того, какие контролы приватности необходимы для конкретной архитектуры.

**CIS Controls v8**: хотя Controls ориентированы на кибербезопасность, Control 3 (Data Protection) и Control 14 (Security Awareness and Skills Training) перекликаются с LINDDUN-категориями Data Disclosure и Unawareness соответственно.

## Основные концепции

### Семь категорий угроз LINDDUN

Как хирургический набор из семи инструментов, каждый из которых предназначен для своей анатомической области приватности, категории **LINDDUN** покрывают весь спектр потенциальных нарушений.

**L — Linkability (Связываемость)**

Аналогия: детектив, восстанавливающий личность человека по разорванным на клочки фотографиям. Каждый клочок сам по себе безобиден — но вместе они формируют узнаваемый портрет.

Принцип: возможность ассоциировать два или более элемента данных или действий пользователя между собой, даже если идентичность субъекта не раскрыта. Связывание — это inference без identification. Браузерный fingerprint, location trace, комбинация псевдонимов — всё это инструменты связывания.

Техническая реальность: веб-сайт бронирования авиабилетов отслеживает browser fingerprint посетителя. Пользователь возвращается к тому же рейсу через час — цена повышена. Система не знает имени пользователя, но связала два визита в одну сессию, извлекла поведенческий инсайт (повышенный интерес) и применила динамическое ценообразование.

Архитектурный компромисс: полная изоляция сессий (unlinkability) конфликтует с бизнес-требованиями: персонализацией, аналитикой, обнаружением мошенничества. Компромисс — сегрегация данных по целям обработки: сессионные идентификаторы для аутентификации ≠ идентификаторы для аналитики.

Стандарт: NIST Privacy Framework, Category CT.DM-P1: "Data processing is limited to the identified purposes in the notice."

**I — Identifiability (Идентифицируемость)**

Аналогия: человек в маске на карнавале. Достаточно услышать его голос, заметить характерную походку, увидеть уникальное кольцо на пальце — и маска становится бесполезной.

Принцип: возможность раскрыть идентичность индивида через утечки, дедукцию или инференцию — в ситуациях, где это не предусмотрено дизайном системы. Идентифицируемость тесно связана со связываемостью: чем больше данных связано, тем ýже anonymity set, тем выше риск ре-идентификации.

Техническая реальность: запрос средней зарплаты женщин в маленькой компании, где работает одна женщина — anonymity set размера 1 превращает агрегированный ответ в раскрытие индивидуальной информации. Это statistical disclosure, известный как differencing attack.

Архитектурный компромисс: удаление прямых идентификаторов (имя, email) недостаточно — quasi-identifiers (дата рождения, почтовый индекс, пол) могут быть скомбинированы для ре-идентификации. Компромисс — k-anonymity, l-diversity, differential privacy — каждый подход жертвует точностью данных ради гарантий приватности.

Стандарт: GDPR Article 4(1) определяет "personal data" через понятие "identifiable natural person"; Recital 26 explicitly introduces the test of identifiability.

**Nr — Non-repudiation (Неотказуемость)**

Аналогия: нотариально заверенный документ, подписанный кровью. Он не просто доказывает, что подпись ваша — он делает невозможным отрицание: «я этого не подписывал».

Принцип: возможность атрибутировать утверждение или действие индивиду — loss of plausible deniability. В контексте приватности неотказуемость становится угрозой, когда индивид не может отрицать действие, которое system хранит как evidence: лог-файлы, цифровые подписи, метаданные документа, водяные знаки.

Техническая реальность: word processor сохраняет автора и ревизии в метаданных документа. Whistleblower, анонимно передавший документ прессе — деанонимизирован через эти метаданные. Read notifications в мессенджерах служат evidence того, что пользователь открыл сообщение.

Архитектурный компромисс: audit trails (необходимые для security) против plausible deniability (необходимой для privacy). Решение — не бинарное «логировать / не логаровать», а градуальное: разграничение того, какие evidence система хранит, как долго, и кто имеет к ним доступ.

Стандарт: GDPR Article 17 (Right to erasure) — право требовать удаления данных, что косвенно ограничивает и non-repudiation evidence.

**D — Detectability (Обнаружимость)**

Аналогия: вы сидите в кафе с книгой. Вы не читаете вслух, книга закрыта, никто не видит содержимого. Но сам FACT, что вы держите книгу «Как бросить курить» в руках — уже раскрывает информацию о вас каждому посетителю кафе.

Принцип: возможность сделать вывод о вовлечённости индивида через наблюдение — даже без доступа к содержанию данных. Передача данных оставляет metadata (источник, назначение, время, размер), которого достаточно для умозаключений.

Техническая реальность: попытка доступа к медицинской карте знаменитости в реабилитационном центре возвращает «insufficient access rights» (а не «record not found»). Разница в ответах системы — side-channel, раскрывающий существование карты, а значит, и факт лечения.

Архитектурный компромисс: uniform response — на любой неавторизованный запрос отвечать одинаково, независимо от существования данных. Но это замедляет диагностику отказов легитимными администраторами. Компромисс — rate limiting и anomaly detection на попытки доступа, не раскрывающие existence.

Стандарт: NIST SP 800-53, PT-4: "Consent" — требует информированного согласия до того, как система начнёт действия, которые detectable.

**Dd — Data Disclosure (Раскрытие данных)**

Аналогия: врач, который для диагностики насморка требует полную генетическую карту, историю всех путешествий за 10 лет и выписку с банковского счёта. Формально — он собирает медицинские данные. Фактически — это excessive collection, несоразмерное цели.

Принцип: excessive collection, storage, processing или sharing of personal data — нарушение data minimisation. Четыре характеристики угрозы: unnecessary data types (типы), excessive data volume (объём), unnecessary processing (обработка), excessive exposure (распространение).

Техническая реальность: dietary app, которое также собирает heart rate measurements — unnecessary data type. Newsletter service, сохраняющий email после unsubscribe — excessive storage. Website, передающий данные посетителя десяткам third-party analytics и tracking services — excessive exposure.

Архитектурный компромисс: полный data minimisation («собираем только то, что нужно прямо сейчас») конфликтует с future-proofing («соберём всё — вдруг пригодится»). GDPR Article 5(1)(c) требует data minimisation — и это технологический constraint, а не организационный.

Стандарт: GDPR Article 5(1)(c) — "adequate, relevant and limited to what is necessary" (data minimisation); CIS Controls v8, Control 3.4: "Enforce data retention."

**U — Unawareness and Unintervenability (Неосведомлённость и невозможность вмешательства)**

Аналогия: вы живёте в доме, спроектированном архитектором, который не предусмотрел окон. Вы не видите, что происходит снаружи (unawareness). А дверь открывается только снаружи — вы не можете ни выйти, ни впустить гостя по своему выбору (unintervenability).

Принцип: insufficient informing, involving or empowering of individuals. Три подкатегории: insufficient transparency (неосведомлённость о факте сбора), insufficient feedback (неосведомлённость о privacy impact на других), insufficient intervenability (невозможность управлять своими данными).

Техническая реальность: traffic cameras, распознающие лица через face recognition — сбор данных без информирования участников дорожного движения. Social media platform, где пользователь не информирован о privacy implications публикации фотографий, включающих других людей. Service, где настройки приватности запрятаны на восьмом уровне меню — формально intervenability есть, практически — нет.

Архитектурный компромисс: полная transparency («показываем каждый data flow пользователю») вызывает notification fatigue и paradoxically снижает реальную осведомлённость. Решение — layered notices (короткий summary + детальный report), just-in-time notification (информирование в момент действия, а не на registration screen).

Стандарт: GDPR Article 13–15 (Right to information and access); Article 20 (Right to data portability) — operability through technology.

**N — Non-compliance (Несоответствие)**

Аналогия: идеально спроектированный автомобиль, проходящий все краш-тесты безопасности — но собранный без учёта экологических норм выбросов. Он безопасен физически, но нарушает регуляторные требования — и в долгосрочной перспективе это создаёт больший риск для производителя, чем разбитый бампер.

Принцип: deviation from security and data management best practices, standards and legislation. LINDDUN фокусируется только на non-compliance, directly deriving from L, I, Nr, D, Dd, U угрозах. Три характеристики: unlawfulness of processing, lacking data lifecycle management, lack of proper cybersecurity risk management.

Техническая реальность: система с корректно работающим функционалом, но data retention policy, реализованная с ошибкой — данные хранятся дольше необходимого, нарушая storage limitation principle GDPR Article 5(1)(e). Large-scale database leak, увеличивающий likelihood of identification — cybersecurity failure, порождающий privacy non-compliance.

Архитектурный компромисс: compliance-by-checklist («у нас есть DPO, значит compliance done») против compliance-by-architecture («система спроектирована так, что deviation от compliance требует conscious override, а не происходит по умолчанию»).

Стандарт: ISO 27001 A.18.1.4 (Privacy and protection of personally identifiable information) — explicit requirement to integrate privacy into ISMS. GDPR Article 25 (Data protection by design and by default).

### Таблица 1: LINDDUN-категории → GDPR-принципы → архитектурные паттерны

| LINDDUN Threat | GDPR Principle | Архитектурный паттерн | Индикатор нарушения |
|---|---|---|---|
| Linkability | Purpose limitation (Art 5(1)(b)) | Data segregation per purpose | Cross-purpose identifiers |
| Identifiability | Data minimisation (Art 5(1)(c)) | k-anonymity; differential privacy | Small anonymity set |
| Non-repudiation | Storage limitation (Art 5(1)(e)) | Ephemeral processing; client-held evidence | Immutable audit log without deletion |
| Detectability | Lawfulness, fairness, transparency (Art 5(1)(a)) | Uniform response; metadata obfuscation | Side-channel response difference |
| Data Disclosure | Data minimisation (Art 5(1)(c)) | On-demand data collection; local processing | Field X present but unused by core feature |
| Unawareness | Transparency (Art 5(1)(a)) | Layered notice; just-in-time consent | Consent bundled; data flows hidden |
| Non-compliance | Accountability (Art 5(2)) | Privacy-by-design audit trail; compliance automation | Policy-documented ≠ code-implemented |

### Data Flow Diagram для приватности

LINDDUN PRO использует **Data Flow Diagram** (**DFD**) — тот же формализм, что и STRIDE, но с принципиально иной аналитической линзой. DFD состоит из четырёх типов элементов: External Entities (внешние сущности — пользователи, сторонние сервисы), Data Stores (пассивные хранилища данных), Processes (вычислительные единицы), Data Flows (потоки данных между элементами). Trust Boundaries обозначают логические или физические границы доверия.

Аналогия: DFD — это топографическая карта информационных потоков системы. STRIDE-аналитик ищет на ней места потенциальных «обвалов» (взломов, инъекций). LINDDUN-аналитик изучает те же потоки, но спрашивает: какие данные вообще не должны покидать эту долину? Где поток входит на территорию, жители которой не были уведомлены? Какие два ручья, сливаясь, позволяют реконструировать личность стоящего у истока?

Процесс LINDDUN PRO — iteration-based threat elicitation. Для каждого взаимодействия в DFD команда рассматривает три точки возникновения угроз:

Source (источник): угроза на уровне элемента, который передаёт данные. Браузер, ретранслирующий cookies или иные linkable identifiers каждому получателю.

Data Flow (поток): угроза на уровне передачи данных. Metadata о source и destination могут быть использованы для связывания потоков или идентификации сторон коммуникации.

Destination (получатель): угроза на уровне принимающего элемента. Данные могут обрабатываться или храниться способом, создающим privacy threat — insecure storage, insufficient data minimisation.

**LINDDUN mapping table** — ключевой инструмент навигации. Это матрица, строки которой — типы DFD-взаимодействий (Process→DataFlow→Process, Process→DataStore, Process→ExternalEntity, DataStore→Process, ExternalEntity→Process — каждое с разбивкой S-fl-D: source, flow, destination), а столбцы — семь категорий LINDDUN. Ячейка отмечена, если конкретная комбинация «DFD-элементы × threat type» релевантна. Mapping table отвечает на вопрос: «Для этого конкретного взаимодействия — какие угрозы приватности я должен рассмотреть?»

### Privacy-by-design: философия и операционализация

Privacy-by-design — не список требований, а архитектурный императив. Его семь foundational principles (Cavoukian, 2011) включают: proactive not reactive, privacy as the default, privacy embedded into design, full functionality (positive-sum, not zero-sum), end-to-end security, visibility and transparency, respect for user privacy.

LINDDUN операционализирует три из них напрямую:

Proactive not reactive → LINDDUN — proactive threat analysis, выполняемый на стадии design, а не post-mortem audit.

Privacy as the default → Data Disclosure analysis в LINDDUN выявляет excessive collection, возвращая систему к default-minimisation.

Visibility and transparency → Unawareness analysis выявляет informational asymmetries между системой и пользователем.

Архитектурный компромисс здесь — «positive-sum, not zero-sum»: приватность не обязательно жертвует функциональностью. Pseudonymisation позволяет сохранить связывание действий пользователя для функциональности, но предотвращает связывание для profiling — это positive-sum tradeoff, а не zero-sum (функциональность минус приватность).

### LINDDUN GO: автоматизированный (gamified) вариант

LINDDUN GO — это не «автоматизированный» анализ в смысле AI/ML-driven pipeline, а lean, gamified approach: 33 карты угроз, представляющие наиболее распространённые LINDDUN privacy threats и hotspots системы. Игра проводится как structured brainstorm в кросс-функциональной команде: domain expert, system architect, developer, **DPO** (Data Protection Officer), legal expert, **CISO** (Chief Information Security Officer), privacy champion.

Аналогия: LINDDUN PRO — это хирургическая операция с полным набором инструментов и протоколов. LINDDUN GO — это экспресс-диагностика в полевых условиях с портативным чемоданчиком: не так глубоко, но быстро (low adoption threshold) и покрывает 80% критических случаев (most common threats).

GO не требует mapping table или threat tree catalog — self-contained threat cards направляют elicitation. Входные данные — unconstrained system sketch (не обязательно формальная DFD). Это выбор для организаций, делающих первые шаги в privacy engineering.

## Сравнение: LINDDUN vs STRIDE, LINDDUN vs PIA

### LINDDUN vs STRIDE

**STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) — методология угроз безопасности, разработанная Microsoft в 1999 году. Сравнение — это сравнение двух линз на одном объекте.

Аналогия: STRIDE — рентгеновский снимок, показывающий переломы костей (уязвимости). LINDDUN — МРТ мягких тканей, показывающее то, что рентген пропускает: inflammation, tissue patterns (паттерны приватности). Они не конкурируют — они комплементарны.

STRIDE-Repudiation ≠ LINDDUN-Non-repudiation. Первая — угроза безопасности: злоумышленник отрицает действие и система не может доказать обратного (отсутствие audit trail). Вторая — угроза приватности: система хранит НЕОПРОВЕРЖИМОЕ доказательство действия пользователя, лишая его plausible deniability. Первая — «мало evidence», вторая — «слишком много evidence».

STRIDE-Information Disclosure ≠ LINDDUN-Data Disclosure. Первая — confidential data leaks to unauthorised parties. Вторая — excessive disclosure даже authorised parties, нарушающее data minimisation. Система, раскрывающая все поля профиля пользователя колл-центру (authorised party) — это не STRIDE-нарушение (нет утечки), но LINDDUN-нарушение (excessive exposure).

### LINDDUN vs PIA (Privacy Impact Assessment)

**PIA** (**Privacy Impact Assessment**) — общий термин для оценки влияния на приватность, предшественник **DPIA** (Data Protection Impact Assessment), mandated GDPR Article 35 для high-risk processing. PIA/DPIA — это процесс управления рисками на уровне организации, тогда как LINDDUN — это threat modelling на уровне программной архитектуры.

Аналогия: PIA — это экологическая экспертиза территории перед строительством: оценивает общее воздействие на среду (права субъектов). LINDDUN — это инженерно-геологический анализ грунта конкретного здания: оценивает threats в архитектуре системы. PIA спрашивает «каков privacy risk для data subjects», LINDDUN — «какие privacy threats в DFD-потоках этой системы».

PIA использует юридические и процессные категории (necessity, proportionality, risk to rights and freedoms). LINDDUN использует технические категории (linkability identifiers, side-channel detectability, data store exposure). PIA output — DPIA report для regulator; LINDDUN output — threat catalogue для engineering team.

### LINDDUN vs NIST SP 800-53 Privacy Controls vs CIS Controls v8

NIST SP 800-53 содержит более 100 privacy-specific controls (Appendix J, family PT). Это «каталог контролов» — что должно быть внедрено. LINDDUN — «процесс выявления» — почему конкретный контроль нужен для конкретной системы.

CIS Controls v8 — operational security framework (18 controls, 153 safeguards). Его privacy coverage — indirect, через Control 3 (Data Protection) и Control 14 (Security Awareness). LINDDUN дополняет CIS, предоставляя privacy-specific threat lens там, где CIS coverage разрежен.

### Таблица 2: Сравнительный анализ LINDDUN, STRIDE, PIA/DPIA, NIST PF

| Измерение | LINDDUN | STRIDE | PIA/DPIA | NIST Privacy Framework |
|---|---|---|---|---|
| Фокус | Privacy threats (7 категорий) | Security threats (6 категорий) | Privacy risk management process | Privacy risk management framework |
| Уровень анализа | Software architecture (DFD) | Software architecture (DFD) | Organisational (processing activity) | Organisational (governance, controls) |
| Output | Threat catalogue (technical) | Threat catalogue (technical) | DPIA report (legal/process) | Privacy risk profile |
| Регуляторный alignment | GDPR (technical mapping) | Indirect (secure processing) | GDPR Article 35 (mandated) | NIST SP 800-53; GDPR-compatible |
| Инструментарий | DFD, threat trees, mapping table, GO cards | DFD, STRIDE-per-element | Template-based assessment | Profiles, tiers, implementation tiers |
| Complexity | Medium–High (PRO), Low (GO) | Low–Medium | Medium (organisational) | High (organisational framework) |
| Разработчик (author) | Kim Wuyts et al., KU Leuven (academia, 2014) | Microsoft (industry, 1999) | EU regulatory framework (2016) | NIST (government, 2020) |

## Уроки LINDDUN: архитектурные insights

**Урок 1: Приватность — это не конфиденциальность**

Самый частый architectural anti-pattern — treating privacy as confidentiality problem. Шифрование данных при передаче (TLS) и хранении (AES-256) решает confidentiality/Information Disclosure — но не решает Linkability (identifier correlation across sessions), Detectability (metadata leakage), Unawareness (lack of transparency). Архитектор, считающий «данные зашифрованы → privacy protected», совершает ту же ошибку, что и владелец банковского хранилища из начальной аналогии: замки целы, но observational threat не устранена.

Архитектурный компромисс: privacy budget. Каждый architectural decision тратит «бюджет приватности» пользователя. Encryption — baseline, не тратит бюджет. Добавление persistent identifier for analytics — тратит Linkability budget. Добавление audit log without deletion policy — тратит Non-repudiation budget.

**Урок 2: DFD — универсальный язык, но с разным синтаксисом**

STRIDE и LINDDUN используют один syntactic formalism (DFD) — и это преимущество: одна модель обслуживает два анализа. Но semantic interpretation отличается радикально: STRIDE рассматривает data flows как «каналы, которые нужно защитить от interception/modification». LINDDUN рассматривает их как «каналы, где уже сам FACT передачи несёт privacy implications».

Аналогия: почтальон (STRIDE) проверяет, что конверт запечатан (integrity), адрес правилен, отправитель верифицирован. Другой почтальон (LINDDUN) спрашивает: а почему вы вообще отправляете медицинскую карту почтой, a не через patient portal с role-based access? Оба смотрят на конверт — но видят разное.

**Урок 3: Threat trees дают precision, но требуют expertise**

LINDDUN threat trees — иерархические деревья для каждой категории угроз, детализирующие threat characteristics до уровня конкретных сценариев. Это главный source of precision в LINDDUN PRO — но они же создают adoption threshold. LINDDUN GO решает эту проблему через self-contained cards, а LINDDUN MAESTRO — через enriched DFD, позволяющий automated reasoning.

**Урок 4: LINDDUN — не аудит (PIA), а design tool**

PIA/DPIA оценивает existing processing activity. LINDDUN анализирует system design до того, как processing начался. В GDPR-терминах: DPIA — Article 35 compliance exercise. LINDDUN — Article 25 (data protection by design and by default) implementation tool.

## Инструменты LINDDUN

Экосистема LINDDUN состоит из трёх уровней инструментария:

**LINDDUN GO (Lean)**: PDF card deck, print-friendly deck, GO Digital card game (веб-приложение), GO online card browser, GO threat tracker sheet. Требуется: колода карт, эскиз системы, кросс-функциональная команда. Уровень: novices, first privacy engineering steps.

**LINDDUN PRO (Systematic)**: LINDDUN threat trees (interactive catalog), LINDDUN mapping table (матрица DFD-взаимодействий × threat types), threat modeling tool (опционально). Требуется: DFD системы, команда с privacy экспертизой, mapping table. Уровень: privacy professionals, compatibility with STRIDE (единая DFD для обоих анализов).

**LINDDUN MAESTRO (Architecture-level)**: enriched DFD methodology, threat-specific system abstraction. Уровень: advanced users, research contexts, integration with formal architecture frameworks. More info — forthcoming.

Ключевой ресурс: https://linddun.org/ — официальный сайт с downloads, инструкциями, threat type browser'ом.

Ключевая публикация (для цитирования): Kim Wuyts and Wouter Joosen, "LINDDUN privacy threat modeling: a tutorial", Technical Report CW 685, Department of Computer Science, KU Leuven, July 2015.

## Архитектурные решения: LINDDUN-informed design patterns

**Pattern 1: Identifier Partitioning (против Linkability)**

Принцип: генерировать per-purpose идентификаторы вместо единого global identifier. Сессионный cookie ≠ analytics identifier ≠ advertising identifier.

Архитектурный компромисс: increased complexity session management, risk of de-synchronisation between identity partitions. Но выгода: linkability surface area сокращена с «всё связано через один ID» до «per-purpose data silos».

**Pattern 2: Onion Encryption of Metadata (против Detectability)**

Принцип: не только payload шифруется, но и metadata о communication patterns скрывается (через padding, timing obfuscation, relay networks).

Техническая реальность: Tor — классический пример. Но для enterprise-систем: API gateway, который pad'ит все ответы до фиксированной длины и вводит random latency — устраняет side-channel через response size и timing.

**Pattern 3: Gradual Data Collection (против Data Disclosure)**

Принцип: on-demand data collection — запрашивать поле данных ТОЛЬКО в момент, когда оно впервые необходимо для операции, и не ранее.

Контраст с anti-pattern: registration form из 25 полей «на всякий случай». LINDDUN-informed redesign: базовый профиль (email, password), геолокация — только при поиске ближайших сервисов, платёжные данные — только при оформлении заказа.

**Pattern 4: Client-side Evidence (против Non-repudiation)**

Принцип: evidence действия хранится на стороне клиента (в user-controlled storage), а не на server-side audit trail. Система хранит cryptographic commitment, а не само evidence.

Аналогия: вместо того чтобы нотариус (server) хранил копию вашего документа, он хранит cryptographic hash — достаточный для verification при предъявлении документа вами, но недостаточный для раскрытия содержания без вас.

**Pattern 5: Just-in-Time Transparency (против Unawareness)**

Принцип: вместо однократного privacy notice при регистрации (который пользователь пролистывает) — контекстное информирование: когда приложение впервые запрашивает доступ к контактам — показать, зачем именно сейчас, какие данные будут извлечены, и как отказаться без потери остального функционала.

## LINDDUN для IoT

IoT-системы создают уникальное пересечение privacy threats: физический мир генерирует непрерывные сенсорные потоки, где каждый measurement несёт detectable information. LINDDUN-анализ IoT выявляет:

Linkability: сенсорные fingerprint'ы разных устройств в одном доме коррелируются → profile household members без их согласия. Smart home hub, агрегирующий данные с thermostats, cameras, voice assistants — становится linkability concentrator.

Detectability: даже если данные конкретного сенсора зашифрованы, frequency и pattern transmissions раскрывают behavioural patterns. Например, smart meter transmissions с 15-минутным интервалом выявляют, когда жильцы дома, спят или отсутствуют.

Data Disclosure: IoT-устройства — классические нарушители data minimisation. Smart TV, собирающий viewing habits. Fitness tracker, собирающий heart rate, GPS trace, sleep patterns — для «подсчёта шагов». LINDDUN-анализ выявляет gap между declared purpose и actual data collection.

Стандартное требование: ISO 27001 Annex A.11 (Physical and Environmental Security) + A.14 (System Acquisition, Development and Maintenance) требуют security design review для IoT-устройств. LINDDUN добавляет privacy dimension в этот review.

## LINDDUN для AI/ML-систем

AI/ML-системы создают угрозы приватности на трёх стадиях жизненного цикла: training data (сбор), model training (inference), model deployment (inference at runtime).

Linkability в ML: model inversion attacks восстанавливают training data members через querying trained model. Membership inference attacks определяют, был ли конкретный data point в training set. LINDDUN-linkability lens: training data → model parameters → query answers — все три могут быть связаны.

Identifiability в ML: generative models (LLM, diffusion) могут inadvertently reproduce training data verbatim. LINDDUN-identifiability lens: differential privacy guarantees (ε, δ) для training process — технический mitigation.

Detectability в ML: SAM факт, что модель доступна через API с определённым behaviour — обнаруживает involvement (model was trained on this distribution).

Публикация 2025 года: Qianying Liao, Jonah Bellemans, Laurens Sion, "A LINDDUN-based Privacy Threat Modeling Framework for GenAI" — расширение LINDDUN на генеративный AI, демонстрирующее applicability фреймворка beyond classical software systems.

## Подготовка к LINDDUN-анализу

Перед проведением LINDDUN-анализа (PRO или GO) организация должна подготовить:

1. Системную документацию: DFD (для PRO) или system sketch (для GO). Без этого analysis — guessing.

2. Команду: domain expert + architect + developer + DPO + legal expert + CISO + privacy champion. Diversity viewpoints — critical.

3. Privacy threat knowledge: familiarity с 7 категориями LINDDUN. Для GO — достаточно прочитать threat cards. Для PRO — familiarity с mapping table и threat trees.

4. Инструменты: для PRO — LINDDUN mapping table, threat trees (PDF or interactive catalog), threat modeling tool (опционально). Для GO — card deck.

5. Время: для PRO — schedule sufficient time (чем сложнее система, тем больше). Для GO — 2–4 часа structured brainstorm session.

6. Регуляторный контекст: understanding applicable regulations (GDPR — EU, CCPA/CPRA — California, CSL — China, LGPD — Brazil). LINDDUN-категории mapping to specific articles.

7. Критерии приоритизации: risk analysis framework (likelihood × impact) для prioritisation identified threats. Impact factors: type/number of data subjects, data sensitivity.

## Чек-лист: 10 вопросов перед LINDDUN-анализом

1. Имеется ли актуальная DFD системы (или system sketch для GO), отражающая реальную, а не документационную архитектуру?

2. Включены ли в команду анализа все необходимые роли: domain expert, architect, developer, DPO, legal expert, CISO, privacy champion?

3. Определён ли регуляторный контекст (GDPR, CCPA, CSL, LGPD), применимый к системе?

4. Идентифицированы ли все External Entities, включая third-party services (analytics, CDN, payment processors), получающие данные?

5. Для каждого Data Store: определён ли retention period и задокументирован ли он в коде/конфигурации, а не только в policy document?

6. Проведён ли предварительный STRIDE-анализ? (Если да — DFD может быть reused с минимальными модификациями для LINDDUN.)

7. Выбран ли уровень LINDDUN (GO — для lean/initial analysis, PRO — для systematic/exhaustive)?

8. Определены ли критерии приоритизации угроз (likelihood × impact, sensitivity scoring) до начала elicitation?

9. Задокументированы ли assumptions о системе (trust boundaries, threat model adversary capabilities) — для avoid ambiguity при revisiting?

10. Существует ли процесс tracking identified threats → mitigations, интегрированный в SDLC, а не выполняемый как разовая активность?

### Ответы

**1. Почему актуальная, а не документационная DFD критична?**
DFD — это graphical system abstraction, используемая LINDDUN PRO. Но «документационная» DFD (нарисованная архитектором год назад) расходится с runtime-реальностью системы: добавился новый third-party analytics service, process реплицирует данные в новый data store, external entity взаимодействует через undocumented channel. LINDDUN-анализ по устаревшей DFD пропускает угрозы в этих undocumented interactions. Privacy threat model должна отражать actual data flows — как они происходят в runtime, а не как они были задуманы в design doc. Рекомендация: validate DFD через runtime observation (network traffic analysis, code-level data flow tracing) перед LINDDUN-сессией.

**2. Зачем в команде одновременно DPO и developer?**
DPO (Data Protection Officer) приносит юридическую и регуляторную expertise: знание применимых статей GDPR, precedents supervisory authorities, типичных findings DPIA. Developer — implementation expertise: он знает, что конкретный data field Х на самом деле хранится в трёх таблицах, а не в одной, и что «анонимизация» сделана через reversible hash, а не через differential privacy. Gap между тем, что DPO думает о системе, и тем, как developer её реализовал — это главный source of privacy non-compliance. LINDDUN-сессия — место where this gap is surfaced and closed.

**3. Почему применимый регуляторный контекст должен быть определён до анализа?**
LINDDUN-категории имеют разный «вес» в разных юрисдикциях. GDPR: heavy emphasis on Data Disclosure (minimisation principle, Article 5(1)(c)) and Unawareness (transparency, Article 13–15). CCPA/CPRA: emphasis on Data Disclosure (sale/sharing opt-out) and Unawareness (notice at collection). CSL (China): emphasis on Identifiability and Non-compliance (data localisation, cross-border transfer assessment). LINDDUN-анализ, не учитывающий регуляторный вес категорий, рискует misprioritise threats: например, потратить disproportionate effort на Linkability в юрисдикции, где supervisory authority фокусируется на Data Disclosure.

**4. В чём риск «невидимых» External Entities?**
External Entities — third-party services, получающие данные. В современной web-архитектуре типичная страница загружает resources от 20–40 third-party origins (CDN, analytics, tag managers, social plugins, ad networks). Каждый — External Entity, получающая данные (IP address, User-Agent, referrer, cookies). LINDDUN-анализ, enumerating только «официальные» third-parties (analytics provider с signed DPA), пропускает Data Disclosure threats через остальные 15 origins. Data Disclosure threat: excessive exposure — данные утекают к parties, с которыми data sharing не обоснован necessity.

**5. Retention period: почему code/config implementation важнее policy document?**
GDPR Article 5(1)(e): "kept in a form which permits identification of data subjects for no longer than is necessary." Требование: storage limitation. Policy document: «данные хранятся 24 месяца». Code: DELETE FROM user_data WHERE last_active < now() - interval '36 months' — фактический retention 36 месяцев, non-compliance (Data Disclosure + Non-compliance). Или: cron job, удаляющий старые записи, silently fails из-за database deadlock — retention period не соблюдается без detection. LINDDUN-informed practice: automated verification — test suite проверяет, что данные старше retention period действительно физически недоступны.

**6. STRIDE → LINDDUN: reuse DFD but different threat lens**
Microsoft Threat Modeling Tool (TMT) генерирует STRIDE-угрозы per DFD-element автоматически. LINDDUN PRO не имеет automated threat generation той же maturity — но использует ту же DFD. Практическая стратегия: провести STRIDE-анализ (security lens), затем reuse DFD для LINDDUN-анализа (privacy lens). STRIDE threat catalogue даёт security baseline. LINDDUN threat catalogue добавляет privacy-specific threats, не покрытые STRIDE: Linkability (STRIDE Information Disclosure covers leaks, not legitimate-but-linkable identifiers), Non-repudiation (STRIDE Repudiation covers missing evidence — opposite concern), Unawareness and Non-compliance (no STRIDE equivalents).

**7. LINDDUN GO vs PRO: критерии выбора**
GO — для: организаций без prior privacy threat modeling expertise; early-stage проектов, где DFD ещё не стабильна; fast assessment (часы, не дни); кросс-функционального alignment (карты — conversation starter). PRO — для: mature privacy engineering programmes; систем с documented DFD (возможно, от STRIDE); exhaustive analysis, требующегося для regulatory compliance evidence (GDPR Article 25 demonstration); integration с formal threat modeling tools.

**8. Критерии приоритизации до elicitation: почему?**
Threat elicitation порождает dozens of identified threats. Без pre-defined prioritisation framework команда рискует: (a) analysis paralysis (каждая угроза обсуждается одинаково долго), (b) bias towards technical-mitigation complexity (угрозы с «интересным» mitigation обсуждаются disproportionately). Framework (likelihood × impact) + pre-scored sensitivity categories даёт systematic prioritisation, независимую от того, «интересна» ли конкретная угроза engineering team.

**9. Assumptions documentation: criticality**
LINDDUN-анализ неизбежно делает assumptions: «trust boundary между frontend и backend — HTTPS/TLS 1.3», «External Entity X has DPA signed», «authentication server is not compromised». При revisiting threat catalogue months later, эти assumptions могут быть invalidated (TLS 1.3 deprecated, DPA expired, authentication server migration). Documented assumptions — traceability: «эта угроза была оценена как low-risk потому что assumption Y — если Y изменился, reassess threat Z».

**10. SDLC integration: LINDDUN как continuous activity**
Privacy threat landscape evolves: система меняется (new features, new third-parties), регуляции меняются (new guidelines from supervisory authorities), threat knowledge evolves (new attack techniques). LINDDUN, проведённый однажды — snapshot, не continuous protection. Integration: LINDDUN analysis — gate в SDLC (design review → LINDDUN → approved for implementation), re-triggered при major architectural changes (new External Entity, new Data Store, new data flow crossing trust boundary). LINDDUN GO особенно подходит для continuous re-assessment благодаря low effort threshold.

## LINDDUN для специфических доменов

**LINDDUN для IoT систем: детальный анализ**

IoT-экосистемы характеризуются гетерогенностью устройств (сенсоры, актуаторы, хабы), постоянным потоком телеметрии и размытыми trust boundaries между физическим и цифровым. Умный дом — классический пример, где десять разнородных устройств пяти производителей обмениваются данными через cloud platforms трёх провайдеров.

Linkability в IoT: сенсорные fingerprint'ы — комбинация типов устройств, их behavioural patterns и network characteristics — создают уникальный household fingerprint, связывающий действия членов семьи даже при отсутствии явной идентификации. Умный термостат, умная камера, умный замок — три устройства трёх производителей, но их данные агрегируются в одном cloud analytics pipeline, создавая linkable household profile.

Detectability в IoT: даже зашифрованные wireless transmissions (Zigbee, Z-Wave, Wi-Fi) детектируемы соседними устройствами. Frequency и pattern этих transmissions раскрывают: количество устройств в доме, behavioural patterns жильцов (время пробуждения, ухода/возвращения, сна), присутствие/отсутствие. Это observational threat, не требующая compromise ни одного устройства.

Data Disclosure в IoT: data minimisation — самый частый architectural anti-pattern. Smart TV собирает viewing habits + voice commands (ambient recording). Fitness tracker собирает GPS trace + heart rate + sleep patterns — для функциональности «подсчёт шагов» GPS достаточен, heart rate — overcollection. LINDDUN-анализ IoT требует per-device, per-data-stream minimisation review.

**LINDDUN для AI/ML систем: threat surface**

Модель ML — это нетрадиционный Data Store: она хранит training data не напрямую (rows), а implicit (weights). Но implicit storage — всё равно storage, и LINDDUN Data Store threats применимы: какие данные retain'ятся в модели? Как долго? Кто имеет доступ к модели (query access)?

Model inversion: adversary queries trained model → reconstructs training data distributions. Membership inference: adversary queries → determines if specific point was in training set. LINDDUN-linkability + LINDDUN-identifiability: training data → model weights → query access chain — все три звена связываемы, и identity членов training set может быть deduced.

Generative AI: LLM, воспроизводящий training text verbatim → Data Disclosure (disclosure of training data, которая могла включать PII). Image generation model, генерирующий recognisable faces → Identifiability (training data identity leakage through model output).

Mitigation: differential privacy during training (formal ε, δ guarantees), output filtering, rate limiting of queries to prevent model inversion via repeated querying.

## Ключевые выводы

1. LINDDUN — не академическая абстракция, а операционный фреймворк, признанный NIST Privacy Framework, OWASP Developer Guide и цитируемый в peer-reviewed публикациях (Springer Software and Systems Modeling, 2025).

2. Семь категорий LINDDUN (Linkability, Identifiability, Non-repudiation, Detectability, Data Disclosure, Unawareness, Non-compliance) покрывают privacy threat surface полнее, чем STRIDE-анализ в одиночку: STRIDE видит confidentiality leaks, LINDDUN — linkability chains, detectability side-channels, unawareness asymmetries.

3. LINDDUN GO снижает adoption threshold: privacy threat modeling — не только для экспертов. Карточная игра из 33 карт делает privacy analysis доступным для кросс-функциональных команд, интегрируя privacy engineering в Agile/DevSecOps workflow.

4. DFD — bridge between security (STRIDE) and privacy (LINDDUN): инвестиция в DFD-моделирование окупается дважды. Одна модель, два complementary анализа.

5. LINDDUN vs PIA/DPIA — не конкуренция, а разделение труда: LINDDUN — design-time technical threat analysis для engineering team. PIA/DPIA — organisational risk assessment для supervisory authority. Оба — GDPR compliance tools, но на разных слоях.

6. IoT и AI/ML domains — стресс-тест для LINDDUN: оба домена генерируют privacy threats, невидимые через традиционную security lens. LINDDUN, применённый к IoT (per-device minimisation analysis) и AI/ML (training-data-leakage-through-model analysis), демонстрирует applicability beyond classical web systems.

7. Privacy-by-design — operationalised: Cavoukian's principles перестают быть abstract manifesto и становятся verifiable engineering criteria через LINDDUN threat elicitation → mitigation traceability.

8. ISO 27001 + NIST SP 800-53 + CIS Controls v8 + NIST Privacy Framework — все четыре стандарта имеют explicit или implicit privacy requirements, которым LINDDUN предоставляет systematic verification process. LINDDUN — инструмент демонстрации compliance, а не замена ему.

9. LINDDUN threat trees — precision tool (PRO), LINDDUN GO — accessibility tool. Выбор — архитектурный компромисс: exhaustive coverage vs adoption feasibility. Оба — legitimate approaches, выбор зависит от organizational maturity.

10. Continuous privacy threat modeling, не one-time exercise: LINDDUN-анализ должен быть интегрирован в SDLC gates (design review, major architectural change), а не выполняться как проектный артефакт, устаревающий к моменту production deployment.
