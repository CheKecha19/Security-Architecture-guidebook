# Облачный мониторинг и SIEM: Архитектура обнаружения угроз, логирования и реагирования в облачной среде

## Что такое облачный мониторинг безопасности?

Представьте современный аэропорт. Десятки тысяч людей проходят через терминалы ежедневно. У службы безопасности нет физической возможности обыскать каждого. Вместо этого работает многоуровневая система: камеры с распознаванием лиц на входе, сканеры багажа, собаки-детекторы, автоматические рамки металлоискателей, операторы в диспетчерской, анализирующие десятки мониторов в реальном времени. Если человек оставил сумку без присмотра — система video analytics генерирует alert, и через 30 секунд сотрудник проверяет. Если пассажир прошёл через три зоны, но не сел на рейс — anomaly detection. Если кто-то дважды неправильно ввёл код доступа в служебную зону — блокировка и вызов охраны. Всё это работает 24/7, без выходных, с эскалацией по уровням severity.

**Облачный мониторинг безопасности** — ровно такой же многоуровневый конвейер для облачной инфраструктуры: сбор данных (логи, метрики, события) со всех компонентов, нормализация в единый формат, корреляция между разрозненными событиями, автоматическое обнаружение аномалий (detection), приоритизация и оповещение (alerting), и автоматизированное реагирование (response). Фундаментальная проблема: в облаке тысячи ресурсов создаются и уничтожаются ежедневно, каждый API-вызов — потенциальный вектор атаки, и без непрерывного мониторинга организация не может ответить даже на базовый вопрос «нас взломали или нет?». Мониторинг превращает облако из «чёрного ящика» в прозрачную среду, где каждое действие оставляет след, и каждый след анализируется.

## Эволюция и мотивация

Десять лет назад мониторинг безопасности был реактивным: администратор видел alert в SIEM (часто — через несколько часов после события), открывал тикет, начинал расследование вручную. Это работало для сред с сотней серверов и десятком изменений в день. В облаке количество событий выросло на три порядка: каждая Lambda-инвокация, каждое открытие S3-объекта, каждое изменение Security Group — это событие, которое должно быть обработано. Ручной SOC (Security Operations Center) образца 2015 года утонул бы в облачном объёме телеметрии за первый час.

Три технологических сдвига фундаментально изменили мониторинг. Во-первых, нативные облачные сервисы аудита (CloudTrail в 2013, Azure Activity Log, GCP Cloud Audit Logs) сделали логирование API-вызовов автоматическим и бесплатным: теперь каждый CreateUser, DeleteBucket, AttachRolePolicy автоматически записывается с полным контекстом (identity, source IP, request parameters, response). Во-вторых, появление ML-driven anomaly detection (GuardDuty в 2017, Azure Sentinel UEBA, GCP Chronicle) позволило перейти от signature-based (ищем известное плохое) к behavior-based (ищем отклонения от нормы) — находить неизвестные атаки, которые не имеют известных сигнатур. В-третьих, SOAR (Security Orchestration, Automation and Response) — автоматизация реагирования: Lambda-функция, которая при обнаружении открытого S3-бакета не просто алертит, а немедленно применяет `s3:PutPublicAccessBlock` и отзывает IAM-права, закрывая уязвимость до того, как SOC-аналитик прочитает email.

Стандарты закрепили переход: NIST SP 800-53 CA-7 (Continuous Monitoring) требует непрерывного мониторинга состояния безопасности; AU-2 и AU-3 определяют объём и содержание аудит-логов; SI-4 (System Monitoring) требует мониторинга системы на признаки атак и несанкционированной активности. CIS Controls v8, Control 8 (Audit Log Management) и Control 13 (Network Monitoring and Defense), операционализируют эти требования в конкретные measurable практики.

## Зачем нужен облачный мониторинг?

**Сценарий 1: Компрометация администраторских credentials.** CISO просыпается от SMS: «Root account AWS выполнил вход в 03:47 MSK из IP 181.214.x.x (Бразилия). Обычно входы: 09:00-18:00 из московского IP. Запрошено создание 50 EC2-инстансов в регионе ap-southeast-1.» Без мониторинга — организация узнала бы об этом из утреннего счёта за облако (+$50,000 за ночь криптомайнинга). С мониторингом — GuardDuty anomaly detection обнаруживает: impossible travel (вход из Москвы в 22:00 и из Бразилии в 03:47), atypical time, unusual API calls (RunInstances никогда не вызывалась root-аккаунтом), генерирует finding severity CRITICAL, CloudWatch Event триггерит Lambda, которая: отзывает все active sessions root-аккаунта, запрещает создание инстансов через SCP и отправляет SMS CISO. Время от компрометации до блокировки: 90 секунд.

**Сценарий 2: Постепенная эксфильтрация данных.** Сотрудник, планирующий увольнение к конкуренту, в течение двух недель скачивает CSV-файлы из S3 с customer data через AWS CLI из корпоративной сети — по 2-3 файла в день, объёмом до 100MB в сумме. Без мониторинга — единичные события теряются в шуме, утечка обнаруживается через месяц, когда конкурент запускает идентичный продукт. С мониторингом — VPC Flow Logs + CloudTrail Data Events показывают: пользователь `ivan.petrov` выполнил `s3:GetObject` для 47 объектов в бакете `customer-data-prod` за 14 дней, при том что его роль — разработчик и он не должен иметь доступ к этому бакету. UEBA (User and Entity Behavior Analytics) строит baseline: типичный разработчик обращается к 0 объектам из этого бакета. Alert на deviation + автоматический IAM Access Analyzer finding: избыточные права.

**Сценарий 3: Уязвимость в CI/CD, ведущая к production-деплою с backdoor.** Злоумышленник через уязвимость в Jenkins модифицирует Docker-образ, внедряя reverse-shell, и пушит его в production ECR. Мониторинг: CI/CD pipeline логгирует `docker push` с неожиданным SHA256 (образ не соответствует тому, что был собран из коммита — integrity check), Container Registry scan (Trivy/AWS ECR Image Scanning) обнаруживает подозрительный бинарник в слое, SIEM коррелирует: push от Jenkins + high-severity vulnerability finding — за 3 минуты до того, как ECS Fargate начал deployment. Блокировка через ECR policy prevention tag `immutable`, и образ не попадает в production.

**Сценарий 4: Compliance-аудит логирования.** Аудитор PCI DSS запрашивает: «Покажите, что каждый доступ к cardholder data залогирован, логи защищены от модификации и хранятся не менее 12 месяцев.» С архитектурой мониторинга: CloudTrail Organisation Trail включён на все аккаунты, S3-бакет для логов с enabled Object Lock (WORM — Write Once Read Many, предотвращает удаление/модификацию), lifecycle policy автоматически архивирует логи в Glacier через 90 дней, Cross-Region Replication на случай потери региона. Аудитор получает доступ только к бакету с логами (через отдельную роль для аудитора с read-only правами на логи). Время подготовки evidence: 20 минут против 2 недель manual сборки.

**Сценарий 5: DDoS с переходом в брутфорс.** E-commerce платформа получает HTTP flood. На первом уровне CloudFront + AWS Shield поглощают volumetric атаку, WAF блокирует SQL injection в параметрах запроса. Но злоумышленник параллельно начинает credential stuffing на login-эндпоинт. Мониторинг: CloudWatch Metric — резкий рост 403-ответов на /login (2000/мин при норме 10/мин), ALB access logs показывают один IP для тысяч запросов с разными email, WAF rate-based rule автоматически блокирует IP на 12 часов. SOC даже не нужно вмешиваться — автоматизированная цепочка detection → response сработала без человека.

## Основные концепции

### Типы телеметрии в облаке: три столпа данных

Эффективный мониторинг строится на трёх типах данных, каждый из которых отвечает на свой класс вопросов и требует своей стратегии сбора и хранения.

**Логи (Logs)** — неизменяемые хронологические записи дискретных событий: API-вызов (CloudTrail — «кто, когда, какой API вызвал, с какого IP, с каким результатом»), сетевой поток (VPC Flow Logs — «source IP, dest IP, port, bytes, accept/reject»), доступ к данным (S3 Data Events — «кто прочитал объект X в бакете Y»), работа приложения (CloudWatch Logs — «ошибка в Lambda, stack trace»), операционная система (syslog, Windows Event Log — «пользователь залогинился, процесс запущен»).

Архитектурное правило: логи должны быть immutable и centralized. Immutable — чтобы злоумышленник, получивший доступ к облачному аккаунту, не мог стереть следы своего присутствия (CloudTrail Log File Validation с цифровой подписью, S3 Object Lock). Centralized — логи со всех аккаунтов организации (50, 500, 5000) стекаются в один security account с strict access control, чтобы расследование могло коррелировать события из development и production аккаунта без переключения контекста.

**Метрики (Metrics)** — числовые временные ряды: CPU utilisation, количество запросов к ALB, latency p99, количество незашифрованных S3-бакетов (compliance metric), количество IAM-пользователей без MFA. Метрики используются для: визуализации трендов (dashboards), автоматического алертинга при превышении порога (CPU > 90% в течение 5 минут), и как evidence compliance (процент покрытия шифрованием должен быть > 99%). Контроль NIST SP 800-53 SI-4 требует системного мониторинга, что реализуется через сбор метрик и определение аномалий на основе baseline.

**События (Events)** — изменения состояния: ресурс создан (EC2 instance launched), конфигурация изменена (Security Group rule added), доступ получен (IAM role assumed), ошибка произошла (Lambda timeout). События — это триггеры для автоматизации: «событие S3 Bucket Created → проверить через 30 секунд, что Public Access Block включён → если нет, принудительно включить и отправить alert». CloudWatch Events / EventBridge — шина событий, через которую проходят все изменения, и на которую подписаны автоматические реакции (Lambda, Step Functions).

### SIEM: корреляция и контекст

**SIEM (Security Information and Event Management)** — система, агрегирующая все три типа данных из десятков источников, нормализующая их в единую модель, выявляющая связи между разрозненными событиями (корреляция) и предоставляющая интерфейс для расследования и реагирования. SIEM не заменяет нативные облачные сервисы мониторинга, а надстраивается над ними.

Архитектура современного облачного SIEM: слой инжеста (принимает VPC Flow Logs, CloudTrail, GuardDuty findings, Security Hub findings, application logs) → слой нормализации (приводит к общей схеме: timestamp, event_type, identity, source_ip, target_resource, action, result) → слой аналитики (корреляционные правила: «если GuardDuty finding:UnauthorizedAccess:IAMUser + CloudTrail:CreateAccessKey → CRITICAL», ML-модели, UEBA) → слой визуализации (dashboards, investigation timelines) → слой реагирования (webhook → Lambda, ticket creation в Jira, PagerDuty alert).

Пример корреляции: CloudTrail показывает `iam:CreateUser` от администратора. Само по себе — норма. VPC Flow Logs показывают исходящее соединение от IP этого администратора на порт 3389 (RDP) хоста в production-подсети. Correlation rule: «CreateUser + RDP-соединение в течение 10 минут от одного identity = возможный privilege escalation через нового пользователя для lateral movement». SIEM генерирует correlated alert с severity HIGH.

### UEBA: Анализ поведения пользователей и сущностей

**User and Entity Behavior Analytics (UEBA)** — ML-driven подход, который строит baseline типичного поведения для каждого пользователя и ресурса (identity) и алертит на отклонения. В отличие от signature-based detection («alert, если IP из threat intelligence feed»), UEBA находит неизвестное: новый вектор атаки, который не имеет известной сигнатуры.

Примеры UEBA-аналитики в облаке:
- Пользователь Anna обычно выполняет `s3:GetObject` для 5-10 объектов в день из бакета `logs`. Внезапно — 5000 объектов за час, включая бакет `customer-data-prod`. UEBA: deviation по объёму и по scope доступа → alert.
- EC2-инстанс `web-server-17` обычно генерирует 100MB исходящего HTTPS-трафика в день. Внезапно — 50GB на порт 8080 неизвестного IP в России. UEBA: deviation по объёму, порту и географии → alert.
- IAM-роль `JenkinsDeploy` обычно ассумится между 08:00-18:00 UTC из IP-диапазона CI/CD. Внезапно — ассумпт в 02:00 из consumer VPN IP. UEBA: deviation по времени и источнику → alert.

UEBA не silver bullet — false positive rate высок при старте (baseline ещё не собран, минимум 2-4 недели данных), и она не даёт context (почему это аномалия — нужно расследование). Поэтому UEBA — второй эшелон после signature-based detection, а не замена ему.

### SOAR: автоматизированное реагирование

**SOAR (Security Orchestration, Automation and Response)** — слой автоматизации, превращающий detection в action без человеческого вмешательства. Ключевой принцип: если реакция на инцидент известна и однозначна (например, «публичный S3-бакет → применить Public Access Block»), она должна выполняться автоматически, а не ждать пробуждения SOC-аналитика.

Компоненты SOAR в облаке:
- **Playbook** — формализованная последовательность шагов для конкретного типа инцидента: «Finding: Publicly Accessible S3 Bucket → 1. Apply Block Public Access, 2. Notify bucket owner with remediation details, 3. Create Jira ticket, 4. If not acknowledged in 4 hours → escalate to Security Team lead».
- **Event-Driven Execution** — Security Hub custom action или EventBridge rule триггерит Lambda с playbook execution при получении finding определённого типа.
- **Feedback loop:** автоматическая ремедиация логируется в тот же SIEM, что и оригинальный finding — для audit trail.

Автоматизированная ремедиация — не для всего. Для обратимых действий (применить Public Access Block) — auto-remediate. Для потенциально разрушительных (отозвать IAM-роль у production-сервиса) — human-in-the-loop: автоматический alert + предложение remediation, но исполнение — после human approval.

## Сравнение подходов к облачному мониторингу безопасности

| Критерий | Нативные облачные сервисы (GuardDuty, Security Hub, Azure Security Center) | Традиционный Enterprise SIEM (Splunk, QRadar) | Cloud-Native SIEM + SOAR (Azure Sentinel, Google Chronicle, Panther) |
|----------|-----------------------------------------------------------------------------|----------------------------------------------|---------------------------------------------------------------------|
| Время до первой детекции после deployment | Минуты: предварительно обученные ML-модели, threat intelligence feeds | Дни-недели: нужно настраивать корреляционные правила, data sources | Часы: предзагруженные детекшн-правила для облачных источников |
| Качество детекции облачных угроз | Высокое: native understanding облачных API (GuardDuty знает, что RunInstances + CreateAccessKey = признак компрометации) | Низкое без кастомных правил: понимает общие сигнатуры (brute force), но не облачную специфику (IAM privilege escalation через PassRole) | Высокое: облачно-ориентированные правила и ML из коробки |
| Стоимость хранения логов (3 года) | Низкая: S3 Standard → Glacier lifecycle, ~$1/TB/мес для архивных логов | Высокая: лицензия SIEM на ingest + storage, $50-200/TB | Средняя: зависит от платформы, Sentinel — pay-as-you-go, Chronicle — фиксированно |
| Вендор-лок | Высокий: детекшн-модели привязаны к облаку (AWS GuardDuty не работает в Azure) | Низкий: работает с любыми источниками данных | Умеренный: Sentinel оптимизирован под Azure (хотя принимает AWS/GCP), Chronicle оптимизирован под GCP |
| Overhead на внедрение | Минимальный: включить toggle в консоли | Значительный: развернуть кластер, настроить ingest pipelines, написать правила | Средний: настроить источники данных, адаптировать встроенные правила |

Выбор: для single-cloud организации — native cloud services (GuardDuty, Security Hub) + SIEM как агрегатор для compliance reporting. Для multi-cloud — cloud-native SIEM, поддерживающий все три облака (Sentinel, Chronicle, Panther, Splunk с облачными apps) как single-pane-of-glass. Нативные сервисы незаменимы как первый эшелон detection (низкая latency, cloud-native understanding), но не заменяют SIEM для cross-source correlation и long-term retention.

## Уроки из реальных инцидентов

### Инцидент Equifax (2017): Мониторинг, который не работал

Хронология: злоумышленники эксплуатировали известную уязвимость CVE-2017-5638 в Apache Struts на веб-сервере Equifax, получили initial access и в течение 76 дней незаметно эксфильтровали персональные данные 147 миллионов человек (номера социального страхования, даты рождения, адреса, номера кредитных карт).

Корневая причина с точки зрения мониторинга: (a) патч для CVE был доступен за 2 месяца до атаки, но не установлен, (b) TLS-сертификат на устройстве, инспектирующем трафик, истёк за 10 месяцев до инцидента, и эксфильтрационный HTTPS-трафик проходил без инспекции, (c) хотя IDS срабатывал на подозрительный трафик, alert не попал к людям, которые могли действовать — уведомления были сконфигурированы некорректно.

Как облачный мониторинг изменил бы исход: (a) AWS Config / Azure Policy обнаружили бы непатченный инстанс (отклонение от desired state — vulnerability management metric), (b) VPC Flow Logs показали бы аномальный объём исходящего трафика на протяжении 76 дней — ML-аномалия детекшн (50GB/день при baseline 500MB/день) сгенерировал бы CRITICAL alert в день 1 эксфильтрации, а не день 76, (c) автоматизированная ремедиация: CloudWatch Alarm на anomaly → автоматическая изоляция подсети (наложение restrict NACL) до ручного расследования.

Урок: мониторинг, который не доставляет alert до людей, способных действовать — это не мониторинг, а иллюзия безопасности. Каждый детекшн-механизм должен иметь tested end-to-end alert delivery pipeline: detection → enrichment → priority → notification channel → acknowledgment → response. Tabletop exercises обязательны для валидации этого пайплайна.

### Инцидент Codecov (2021): Supply-chain атака, обнаруженная через мониторинг

Хронология: в январе 2021 года злоумышленник получил доступ к Bash Uploader-скрипту Codecov (инструмент для CI/CD code coverage) и модифицировал его, добавив строку, которая отправляла все переменные окружения CI/CD (включая API-ключи, токены доступа, пароли) на удалённый сервер злоумышленника. Модифицированный скрипт распространялся через легитимный CDN Codecov в течение 2 месяцев, скомпрометировав тысячи организаций, включая Rapid7, Twilio и HashiCorp. Обнаружен: 1 апреля 2021 года клиентом Codecov, который заметил аномальный исходящий трафик из своего CI/CD — соединение на неизвестный IP в Германии.

Облачный мониторинг-аспект: это supply-chain атака на уровне dependencies. Ключевой detection — не signature-based (нет известной сигнатуры на этот конкретный скрипт), а behavior-based: CI/CD-инстанс, который обычно общается только с GitHub, Docker Hub и внутренним artifact registry, внезапно открывает соединение на IP в Германии на порт 443. VPC Flow Logs + Network Firewall доменная репутация → alert на unknown external destination.

Как архитектура мониторинга смягчила бы атаку: (a) Network Firewall с разрешённым egress-листом (allowed domains only: `*.github.com`, `*.docker.com`, internal domains) — трафик на неизвестный IP был бы заблокирован по умолчанию, (b) VPC Flow Logs аномалия на unknown destination сгенерировала бы alert на день 1, (c) код CI/CD пайплайнов с mandatory hash verification для всех внешних скриптов (integrity check). CIS v8 Control 2.4 (Software Inventory) требует checksum verification для ПО и скриптов.

Урок: supply-chain атаки не имеют статических сигнатур, поэтому signature-based detection слеп. Поведенческий мониторинг (egress filtering, unknown destination detection, CI/CD behaviour baseline) — единственный эффективный рубеж. И egress фильтрация — не опция, а mandatory baseline для любой CI/CD-среды с доступом к production.

## Инструменты и средства облачного мониторинга безопасности

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| Cloud Native Threat Detection (GuardDuty, Azure Security Center, GCP Security Command Center) | Автоматическое обнаружение угроз на основе ML, threat intelligence и анализа облачных API-вызовов, без настройки правил | Runtime: непрерывный анализ CloudTrail/VPC Flow Logs/DNS queries | NIST 800-53 SI-4 (System Monitoring), CIS v8 Control 13 |
| SIEM / Centralized Log Analytics (Splunk, Sentinel, Chronicle, Elastic Security) | Агрегация логов из всех источников, корреляция, долгосрочное хранение, compliance reporting, расследование инцидентов | Post-ingestion: непрерывный анализ + ad-hoc investigation | NIST 800-53 AU-3 (Audit Record Content), AU-6 (Audit Review), CIS v8 Control 8 |
| UEBA (User and Entity Behavior Analytics) | Построение поведенческого baseline для пользователей и ресурсов, обнаружение отклонений | Runtime: на основе historical data (2-4 недели baseline) | NIST 800-53 AT-2 (Insider Threat), OWASP ASVS V2.1 |
| SOAR / Automated Response | Автоматизация playbook реагирования: auto-remediation, notification, ticket creation, escalation | Post-detection: реакция на finding в реальном времени | NIST 800-53 IR-4 (Incident Handling), CIS v8 Control 17 |
| Cloud Security Posture Management (CSPM) | Мониторинг конфигураций: обнаружение misconfigurations (открытые S3, отсутствие шифрования, избыточные права), compliance checking против стандартов | Continuous + pre-deployment check | NIST 800-53 CA-7 (Continuous Monitoring), CIS v8 Control 4 |
| Network Traffic Analysis / NDR (Network Detection and Response) | Глубокий анализ сетевого трафика: VPC Flow Logs anomaly detection, IDS/IPS сигнатуры, DNS-аналитика | Runtime: на сетевых агрегационных точках (TAP, VPC mirroring) | NIST 800-53 SC-7 (Boundary Protection), CIS v8 Control 13.5 |

Ключевая метрика эффективности всего инструментального стека — Mean Time to Detect (MTTD). Организации, достигающие зрелости в мониторинге, имеют MTTD менее 5 минут для critical findings (compromised credentials, public data exposure) и менее 1 часа для high findings. Достигается это за счёт комбинации: native threat detection (минуты) + SIEM-корреляция в реальном времени + автоматическая диспетчеризация alert прямо в мессенджеры/телефон ответственному, минуя email.

CSPM в этом стеке занимает особое position: он не столько про «атаку прямо сейчас», сколько про «неправильная конфигурация, которая станет точкой атаки через неделю». Профилактический мониторинг, а не только реактивный. CSPM снижает MTTD для misconfigurations, но главное — снижает MTTR (Mean Time to Remediate), особенно с автоматической ремедиацией для детерминированных случаев (включить шифрование, заблокировать публичный доступ).

## Архитектурные решения

Построение зрелого облачного мониторинга — это проектирование пайплайна, а не выбор инструмента. Пайплайн должен работать при масштабе: тысячи аккаунтов, миллионы событий в час, retention на годы, и zero blind spots.

**Архитектурные принципы:**
- **Centralized Logging Account:** выделенный security account (AWS Organizations member), в который стекаются все логи со всех аккаунтов через Organization Trail (CloudTrail) и cross-account S3 replication. У этого аккаунта — максимальная изоляция: доступ только у security-команды, MFA mandatory, все действия алертятся в этот же SIEM (логирование логгера).
- **Immutable Storage:** S3-бакет для логов с Object Lock в governance mode (предотвращает удаление, но security-команда может изменить retention settings с MFA). Compliance: WORM storage для audit logs — прямое требование PCI DSS Req 10.5.3 и NIST SP 800-53 AU-9.
- **Tiered Retention:** Hot (S3 Standard, 30 дней — активные расследования) → Warm (S3 Infrequent Access, 90-365 дней) → Cold (Glacier Deep Archive, 1-7 лет для compliance minimum). Автоматический lifecycle management через S3 Lifecycle Policies — без этой автоматизации ручное управление retention становится операционным кошмаром.
- **Real-Time Stream for Critical Events:** помимо batch-доставки логов в S3, критичные события (CloudTrail: ConsoleLogin, CreateAccessKey, DeleteTrail, StopLogging) должны доставляться в SIEM через streaming (CloudWatch Logs Subscription Filter → Kinesis Data Firehose → SIEM) с latency менее 2 минут. Batch-доставка может задерживаться до 15 минут — для обнаружения active attack это неприемлемо.

**Метрики эффективности:**
- Log Coverage Ratio: доля облачных аккаунтов с enabled CloudTrail/Activity Log (цель: 100%)
- MTTD для critical findings (цель: менее 5 минут)
- Alert-to-Acknowledgment Time: время от генерации алерта до его просмотра человеком (цель: менее 30 минут для CRITICAL, менее 4 часов для HIGH)
- False Positive Rate: доля алертов, маркированных аналитиками как false positive (цель: менее 5% — иначе SOC десенситизируется)
- SOAR Automation Ratio: доля алертов, обработанных автоматизированно (цель: 60-80% для детерминированных случаев)

**Красные флаги деградации:** CloudTrail trail в статусе Logging: Off более 10 минут (прямая компрометация или misconfiguration); Security Hub / Defender for Cloud показывает растущий тренд failed compliance checks (систематическая деградация posture); SIEM ingest pipeline backlog > 5 минут (риск упустить real-time атаку); MTTD растёт месяц к месяцу (SOC overloaded или detection правила устарели).

**Сценарий миграции с фрагментированного мониторинга:** этап 1 — включение базового coverage: Organization Trail на все аккаунты, VPC Flow Logs на все production VPC, GuardDuty / Security Command Center на все регионы. Этап 2 — конфигурация real-time streams для критичных событий в SIEM. Этап 3 — развёртывание и тюнинг SOAR playbooks на top-5 incident types (public S3, compromised credentials, malware on EC2, brute force, privilege escalation). Этап 4 — ежеквартальный tabletop exercise: simulated attack, измерение полного времени от injection до detection до response.

## Подготовка к собеседованию: Common pitfalls & key questions

**Типичная ошибка 1: «CloudTrail включён — значит, мониторинг есть».**
CloudTrail пишет логи, но если никто не анализирует их в реальном времени, это forensic evidence для пост-инцидентного расследования, а не monitoring. Мониторинг = real-time analysis + alerting + response. Кандидат должен различать «логирование» и «мониторинг».

**Типичная ошибка 2: «Мы используем GuardDuty, значит SIEM не нужен».**
GuardDuty даёт findings, но не коррелирует их с: состоянием конфигурации (Security Hub), уязвимостями (Inspector), событиями из приложений и on-premise систем. SIEM обеспечивает cross-source correlation. Без неё: GuardDuty finding «EC2 instance communicating with known-bad IP» — не коррелирует с тем, что это инстанс с RCE-уязвимостью, обнаруженной Inspector'ом 3 дня назад — важнейший контекст.

**Типичная ошибка 3: «Мы настроили алерты — проблем нет».**
Алертинг без тестирования создаёт иллюзию. Нужна end-to-end validation: от инжектирования тестового вредоносного события до получения SMS/звонка ответственным лицом. В инциденте Equifax TLS-сертификат истёк — мониторинг не работал 10 месяцев, и никто не знал, потому что health monitoring самого мониторинга был деградирован.

**Сложный вопрос 1: «Как организовать мониторинг в multi-cloud среде (AWS + Azure + GCP) так, чтобы SOC видел единую картину? Какие компромиссы возникают?»**
Проверяется: способность масштабировать мониторинг на гетерогенную среду. Ответ: ключевая проблема — разные форматы логов и разные severity шкалы у разных облаков (AWS GuardDuty: Low/Medium/High, Azure Sentinel: Informational/Low/Medium/High, GCP: Low/Medium/High/Critical). Решение: normalization layer (на уровне SIEM ingest pipeline), где все события приводятся к единой таксономии (например, OCSF — Open Cybersecurity Schema Framework). Компромисс: единый SIEM для трёх облаков — single-pane-of-glass, но сложнее в настройке и дороже (ingest volume растёт); отдельные SIEM на облако — дешевле, но SOC переключается между тремя консолями, теряя cross-cloud корреляцию. Практический компромисс: native detection работает per-cloud локально, findings экспортируются в единый SIEM (Splunk/Sentinel) для корреляции и визуализации.

**Сложный вопрос 2: «SOC утонул в false positives — 5000 алертов в день, 95% из которых false positive. Как вы решите эту проблему, сохранив coverage?»**
Проверяется: опыт тюнинга детекционных систем. Ответ: проблема не в алертах, а в их качестве. Решение — (a) агрегация: вместо отдельных алертов на каждый failed login, aggregate за 5-минутное окно (brute force detection — не на 1 логин, а на 50 за 5 минут), (b) enrichment: каждый alert дополняется контекстом (is this IP known-corporate-VPN? if yes → suppress), (c) suppression rules: если один и тот же asset генерирует одинаковый алерт 100 раз за день — после 5-го раза suppress с summary в конце дня, (d) ML-based prioritization: модель, которая оценивает алерты и поднимает score для тех, которые похожи на реальные инциденты (на основе исторических confirmed incidents). Метрика после тюнинга: FP rate снижается с 95% до 5-10%, и SOC focus смещается на actionable alerts.

**Сложный вопрос 3: «Как мониторить serverless (Lambda, Cloud Functions) на предмет атак, если там нет агентов, нет файловой системы и время жизни функции — секунды?»**
Проверяется: понимание специфики serverless monitoring. Ответ: агентный мониторинг невозможен. Detection строится на: (a) CloudTrail Data Events + CloudWatch Logs — invocation pattern analysis (рост invocation rate, рост error rate, изменение duration p99), (b) IAM-based detection — GuardDuty анализирует Lambda IAM-роль: если роль начала выполнять неожиданные API-вызовы (S3:ListBucket, хотя раньше никогда не делала) → possible compromised function, (c) Application layer — Lambda function code должен включать structured logging (JSON) с security-relevant fields (input validation failures, authorization denials), доставляемый в SIEM через CloudWatch Logs subscription, (d) VPC Flow Logs — если Lambda в VPC внезапно генерирует unexpected egress connection → anomaly. Компромисс: в serverless нет host-based controls — вся защита вынесена в IAM layer и observability layer. NIST SP 800-53 AC-4 требует enforcement даже в serverless — реализуется через restrictive IAM-роль и monitoring.

## Чек-лист понимания

- [ ] Почему логгирование без real-time анализа и алертирования — это forensic инструмент, а не мониторинг безопасности?
- [ ] Как обеспечить иммутабельность облачных логов, чтобы злоумышленник с административным доступом не мог стереть следы атаки?
- [ ] В каком случае UEBA генерирует false positive на легитимного администратора и как архитекторы минимизируют этот риск?
- [ ] Почему SOAR-автоматизация детерминированной ремедиации критична для ограничения blast radius атаки, а не просто «экономит время SOC»?
- [ ] Как организовать мониторинг в multi-account AWS-организации (500+ аккаунтов) с single-pane-of-glass, не создавая bottleneck в security account?
- [ ] В каком случае SIEM-корреляционное правило не обнаруживает атаку, а нативные облачные detection — обнаруживают?
- [ ] Почему мониторинг CI/CD-среды и egress-трафик наравне с production — requirement для защиты от supply-chain атак?
- [ ] Как CSPM и threat detection дополняют друг друга: что одно детектирует, а другое — нет?
- [ ] В каком случае автоматическая ремедиация (auto-remediation) создаёт больший ущерб, чем атака, которую она предотвращает?
- [ ] Почему каждый детекшн-пайплайн должен проходить регулярный end-to-end validation через tabletop exercises?

### Ответы на чек-лист

1. **Ответ:** Логгирование записывает события для будущего forensic анализа — постфактум можно восстановить картину. Но без real-time analysis организация не знает о компрометации в момент её развития. Атака, длящаяся 76 дней (Equifax), была залогирована (IDS alerts), но не проанализирована в реальном времени — залогированность не предотвратила ущерб. Мониторинг = непрерывный анализ logs + автоматическое оповещение о finding + response action до того, как атака нанесёт ущерб. CIS v8 Control 8.2 требует automated real-time alerting.

2. **Ответ:** Иммутабельность — многоуровневая: (a) S3 Object Lock (WORM) запрещает удаление/перезапись в течение retention period, (b) S3 MFA Delete — для удаления объекта требуется MFA-код от security-администратора, (c) CloudTrail Log File Validation — цифровая подпись каждого доставленного log file, позволяющая доказать, что файл не был модифицирован после создания, (d) Replication в отдельный security account, где у production-аккаунта нет прав на удаление, (e) дополнительно — stream logs в SIEM real-time, где внутреннее хранилище SIEM имеет собственный retention и access control. NIST SP 800-53 AU-9 (Protection of Audit Information) требует защиты от modification и deletion.

3. **Ответ:** Легитимная административная активность может быть аномальной с точки зрения pure ML: администратор работает в выходные из-за maintenance window, использует новый инструмент (что создаёт new API calls pattern) или меняет регион (командировка) — UEBA детектирует это как anomaly. Минимизация: (a) UEBA модель, обученная с labeled exceptions (maintenance window = normal), (b) suppression rules для заранее анонсированных maintenance windows (change management integration), (c) human feedback loop: аналитики marking alerts as false positive → модель адаптируется и снижает sensitivity для данного behaviour pattern.

4. **Ответ:** Ручной response: SOC-аналитик получает alert → открывает тикет → через 30 минут начинает расследование → через 2 часа выполняет remediation. Blast radius за это время: данные эксфильтрованы, credentials использованы, backdoor установлен. Автоматическая ремедиация: finding «Public S3 bucket» → Lambda через 15 секунд применяет PutPublicAccessBlock. Blast radius — 0 objects скачано. Разница между «detected in 2 minutes, remediated in 15 seconds» и «detected in 2 minutes, remediated in 2 hours» — это разница между инцидентом и near-miss. NIST SP 800-53 IR-4 требует incident response capability; автоматизация — её technical enforcement.

5. **Ответ:** (a) Organization Trail — все аккаунты пишут CloudTrail в центральный logging S3-бакет (в security account), (b) Security Hub — delegated administrator account (security account) агрегирует findings из всех member аккаунтов, (c) SIEM инжестит Security Hub findings (а не сырые CloudTrail из 500 аккаунтов напрямую), снижая ingest volume в SIEM на 90% — Security Hub pre-aggregates findings, (d) для ad-hoc investigation — Athena над S3-бакетом с CloudTrail, federated queries к CloudTrail Lake. Не bottleneck: только findings (суммированные, приоритизированные) стримятся в SIEM, raw logs — в S3 для forensic query.

6. **Ответ:** SIEM правило «CloudTrail: ConsoleLogin failed from unknown IP → alert» не обнаружит атаку, если злоумышленник использует скомпрометированные валидные credentials с корпоративного VPN — логин успешный, IP известный. GuardDuty может обнаружить через: impossible travel (логин из VPN + через минуту из другого региона), или через behavioural deviation (пользователь, никогда не запускавший EC2, внезапно выполнил RunInstances ×50 раз). Это behavior-based detection, недоступный SIEM без ML-модели. Поэтому native threat detection services — mandatory первый эшелон, не заменяемый signature-based SIEM.

7. **Ответ:** CI/CD — среда с наибольшей концентрацией привилегий: деплой-ключи, доступ к production-реестру образов, IAM-роли с широкими правами. Supply-chain атака (Codecov, SolarWinds) именно через CI/CD распространяет вредоносный код/конфигурацию в production. Мониторинг CI/CD включает: (a) все egress-соединения из CI/CD-подсети — разрешены только known domains, (b) все изменения CI/CD-конфигурации (GitHub Actions, Jenkinsfile) — алерт на модификацию, (c) integrity checks на всех pull'имых артефактах (хеш-суммы). CIS v8 Control 16 (Application Software Security) и NIST SP 800-53 CM-7 (Least Functionality) требуют этих контролей.

8. **Ответ:** CSPM детектирует misconfigurations (состояние): открытый S3-бакет, отсутствие шифрования, IAM-пользователь без MFA. Threat detection детектирует активные атаки (действия): brute force, malware communication, data exfiltration. Это два комплементарных слоя: CSPM закрывает «пассивные» уязвимости (неправильные настройки), threat detection — «активные» (кто-то атакует). Инцидент Capital One: CSPM (правильно настроенный) показал бы избыточные IAM-права (misconfiguration), но threat detection (GuardDuty) показал бы SSRF-атаку к metadata endpoint (active attack). Оба слоя нужны.

9. **Ответ:** Auto-remediation применима только к action, где последствия от action меньше, чем последствия от не-действия, и action безопасно обратим. Пример провала: Lambda автоматически применяет NACL Deny All к production-подсети в ответ на false-positive anomaly detection в Flow Logs — весь production downtime. Минимизация: (a) auto-remediation только для ресурсов с тегом `auto-remediate:true` (гарантия, что команда явно opted in), (b) circuit breaker: если remediation action затрагивает > N ресурсов за 5 минут — остановиться и отправить human approval alert, (c) canary: перед ремедиацией 100% ресурсов — remediate 1%, проверить, что нет негативного impact, затем остальные.

10. **Ответ:** Tabletop exercise (симуляция инцидента) проверяет не технологии, а end-to-end процесс: alert generated → кто получил? → открыл за 5 минут или через час? → правильно ли эскалировал? → кто принимал решение о блокировке? → сколько заняла ремедиация? Технически всё настроено правильно — но если SOC-аналитик ночью не имеет доступ к PagerDuty или не знает процедуру, alert останется непросмотренным, и весь технический стек бесполезен. NIST SP 800-53 IR-8 (Incident Response Plan) и CIS v8 Control 17 требуют регулярного тестирования IR-процедур.

## Ключевые выводы для собеседования

- **Самый критичный принцип:** Мониторинг без real-time alerting и tested response — иллюзия безопасности. Каждый день задержки между компрометацией и обнаружением (dwell time) — это день, когда злоумышленник оперирует в вашей среде. MTTD — главный KPI зрелости мониторинга.
- **Главный компромисс:** Увеличение coverage (больше логов, больше источников, больше detection правил) увеличивает visibility, но также увеличивает noise (false positives) и стоимость (ingest, storage, compute). Без тюнинга и автоматизации SOC тонет или игнорирует алерты — zero visibility на практике.
- **Связь с ключевым стандартом:** NIST SP 800-53 AU (Audit and Accountability) family и CA-7 (Continuous Monitoring) формируют полный compliance-фреймворк: что логировать, как хранить, как защищать от модификации, как использовать для обнаружения атак.
- **Практическое правило:** Если в вашей организации невозможно ответить на вопрос «кто вчера обращался к production database?» за 30 минут — у вас нет мониторинга, у вас есть иллюзия мониторинга. Single-source-of-truth (SIEM с полным ingest coverage) — не опция, а baseline.

---
_Статья создана на основе анализа стандартов NIST SP 800-53 (AU, CA, SI Controls), CIS Controls v8 (8, 13, 17), OWASP ASVS, официальной документации облачных сервисов мониторинга (AWS CloudTrail/GuardDuty/Security Hub, Azure Monitor/Sentinel, GCP Cloud Logging/Security Command Center/Chronicle), разбора публичных инцидентов (Equifax 2017, Codecov 2021) и материалов SANS Incident Response Framework._
