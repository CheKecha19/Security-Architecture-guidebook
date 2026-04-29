# Безопасность Serverless: Защита функций как сервиса, event-driven архитектур и FaaS-окружений

## Что такое безопасность Serverless?

Представьте себе ресторан с системой «шведский стол». Гости приходят, берут тарелки, накладывают еду, садятся за столики, официанты убирают посуду, повара пополняют блюда — всё работает как отлаженный конвейер. Но теперь вообразите, что ресторан решил перейти на «еду по звонку»: у вас нет столика, нет официанта, нет гарантии, что вам подадут конкретное блюдо именно тот повар. Вы просто нажимаете кнопку, и через несколько минут на ближайшем окне выдачи появляется заказ. Вы не знаете, какая плита использовалась, какой повар готовил, остывали ли продукты в холодильнике — только результат. И главное: вы платите не за столик и не за официанта, а только за каждый приготовленный обед.

**Serverless** (бессерверная архитектура, прежде всего FaaS — Functions-as-a-Service) — это именно такой «ресторан по звонку» для программного кода. Вы загружаете функцию (кусок кода, который выполняет одну задачу), определяете триггер (что запускает функцию — HTTP-запрос, файл в S3, сообщение в очереди), и облачный провайдер делает всё остальное: выделяет вычислительные ресурсы, запускает код, масштабирует при всплесках нагрузки, выключает среду после выполнения, выставляет счёт за миллисекунды использования. Безопасность serverless — это не традиционная защита сервера (сервера нет), а защита функции как изолированной единицы выполнения, её взаимодействия с событиями и её прав в облачной среде.

Фундаментальная проблема безопасности: исчезновение сервера означает исчезновение целого класса защитных механизмов — host-based IDS, файрвола на уровне ОС, патч-менеджмента, immutable infrastructure с золотыми образами. Serverless требует переноса всей защиты на уровень IAM (каждой функции — минимальная роль), код (input validation, sanitization, no secrets in code) и observability (structured logging + monitoring invocation patterns).

## Эволюция и мотивация

Serverless родился из двух потребностей: утилизация ресурсов (разработчики платили за 24/7 инстанс EC2, который обрабатывал 100 запросов в день — 99% времени idle) и скорость разработки (настройка инфраструктуры занимала дни, serverless позволяет деплоить функцию за минуты). AWS Lambda, запущенная в 2014 году, стала первой массовой FaaS-платформой, и к 2020 году serverless стал дефолтным подходом для event-driven workloads: обработка изображений, stream processing (Kinesis), API-бэкенды, IoT data ingestion, scheduled tasks.

Три сдвига сделали serverless security критичной. Во-первых, экспоненциальный рост числа функций — в крупных организациях тысячи Lambda-функций, каждая со своей IAM-ролью, своими триггерами и своими зависимостями. Управление этим «зоопарком» без автоматизации невозможно, и каждая неучтённая функция — потенциальная точка входа (shadow IT в serverless). Во-вторых, осознание нового класса атак: event injection — традиционные SQL injection и XSS, но доставляемые через event payload функции (S3 PutObject event, API Gateway body, SQS message). Если функция доверяет event без валидации — эксплуатация тривиальна. В-третьих, cold start data leakage — остаточные данные в рантайме от предыдущей инвокации (Lambda reuse container). Хотя теоретическая уязвимость, практические инциденты с утечкой sensitive данных между tenant'ами (AWS Lambda, 2017-2020) привели к hardening isolation (Firecracker microVM).

Стандарты адаптировались: NIST SP 800-53 требует тех же защитных мер для serverless (AC-6 Least Privilege, AU-2 Audit Events, SI-4 System Monitoring), но реализация иная — нет хоста для агента, нет файловой системы для file integrity monitoring. OWASP Top 10 Serverless (2018, 2023) перечисляет специфические serverless threats: injection, broken authentication, sensitive data exposure, insecure deployment, function event data injection.

## Зачем нужна серверлес-безопасность?

**Сценарий 1: Избыточная IAM-роль функции.** Команда обработки изображений создала Lambda-функцию `image-resizer`, которая скачивает файл из S3-бакета, ресайзит и сохраняет обратно. «На всякий случай» роль функции получила политику `s3:*` на `Resource: *`. Через полгода уязвимость в image-processing library позволяет злоумышленнику внедрить вредоносный код через crafted JPEG-файл (event injection). Он получает execution в Lambda-окружении и, используя `s3:*`, скачивает все данные из `customer-data-prod` бакета. С least-privilege IAM: роль `image-resizer` имеет `s3:GetObject` только на `arn:aws:s3:::upload-bucket/*` и `s3:PutObject` только на `arn:aws:s3:::processed-images/*`. Даже при exploitation — доступ ограничен двумя бакетами.

**Сценарий 2: Event injection через API Gateway.** E-commerce Lambda `checkout` принимает JSON-body от API Gateway: `{"cart_id": "...", "coupon_code": "SAVE20"}`. Код функции: `const discount = await db.query("SELECT discount FROM coupons WHERE code = '" + event.coupon_code + "'")`. Классический SQL injection — злоумышленник отправляет `coupon_code: "'; DROP TABLE coupons; --"`, и промо-таблица дропается. С безопасностью: input validation library (Joi/Zod) проверяет, что `coupon_code` соответствует `/[A-Z0-9]{5,10}/`, иначе — reject. Parameterised queries (Prepared Statements) — даже валидный `SAVE20` не интерпретируется как SQL-код. OWASP Top 10 Serverless Injection (A1) требует input validation + parameterisation для всех event sources.

**Сценарий 3: Секреты в environment variables функции.** Lambda `payment-processor` использует `STRIPE_API_KEY` из environment variable. Переменная видна в консоли AWS, в `aws lambda get-function-configuration`, и в CloudWatch Logs (при console.log всего event payload). Злоумышленник через misconfigured IAM-роль другого сервиса получает `lambda:GetFunctionConfiguration` и читает API-ключ. С архитектурой: Lambda использует AWS Secrets Manager — при cold start Lambda Extension кэширует секрет в in-memory защищённом хранилище, не экспонируя в environment. Альтернативно — `AWS_Parameters_and_Secrets_Lambda_Extension`, который кэширует параметр локально и ротирует. NIST SP 800-53 IA-5 требует защиты аутентификаторов на протяжении всего жизненного цикла.

**Сценарий 4: DDoS через отсутствие concurrency limit.** Serverless-функция `public-api` не имеет reserved concurrency limit. Злоумышленник запускает distributed invocation атаку — тысячи параллельных запросов. Lambda scale'ится до 10000 concurrent executions, все они обращаются к backend RDS с connection pool из 50 — база данных падает под connection storm. Sibling-сервисы, использующие ту же Lambda account, испытывают throttling (account concurrency limit exhausted). С архитектурой: (a) API Gateway throttling per client (API key + usage plan: 100 req/sec), (b) Lambda reserved concurrency для `public-api` = 500 (кап на параллельные инвокации), (c) WAF rate-based rule блокирует IP при > 500 запросов/мин. CIS v8 Control 13 (Network Monitoring and Defense) требует DDoS mitigation.

**Сценарий 5: Supply-chain через dependencies.** Serverless-функция `pdf-generator` использует npm-пакет `wkhtmltopdf` из публичного репозитория. Атакующий через typosquatting публикует пакет `wkhtmltopdf` (с кириллической «о»), который содержит reverse-shell. CI/CD pipeline без SCA-проверки подтягивает вредоносный пакет. С архитектурой: (a) Software Composition Analysis (SCA) — npm audit / Snyk / Dependabot сканирует dependencies на known vulnerabilities and typosquatting patterns на этапе PR, (b) Dependency lock file (package-lock.json) с integrity hashes блокирует substitution — npm install сверяет SHA-512 integrity, (c) Private package registry (AWS CodeArtifact, JFrog Artifactory) — разрешены только packages из внутреннего прокси, внешний npmjs.org недоступен.

## Основные концепции

### Функция как единица доверия: IAM-роль и execution environment

В serverless каждая функция — изолированная единица выполнения с собственным execution environment, собственными credentials (через IAM-роль) и zero state между инвокациями (stateless by design). IAM-роль функции — это «личность» функции в облаке. Через неё функция аутентифицируется при вызовах других сервисов, и она определяет максимальный blast radius при компрометации.

Execution Environment (рантайм контейнер): создаётся при cold start (~100-500ms), живёт для нескольких последовательных инвокаций (warm start, ~1-10ms overhead), уничтожается после периода idle (~5-45 минут). Два архитектурных следствия: (a) runtime агенты — не persistent, нужны Lambda Extensions (внешние процессы, работающие внутри execution environment параллельно функции, но с separate lifecycle), (b) данные в памяти между инвокациями — потенциальный leakage. Глобальные переменные, инициализированные на cold start, доступны в warm invocations. Если функция кэширует sensitive данные в global scope — следующий invocation (потенциально от другого пользователя) может прочитать их.

Принцип least privilege для IAM-роли функции: не один «super-role» для всех функций, а per-function granular role. Функция `image-resizer` имеет одну роль, `email-sender` — другую, `report-generator` — третью. CloudFormation/Terraform template генерирует role per function автоматически с explicit actions + resources. CI/CD проверяет через IAM Access Analyzer simulation, что роль имеет exactly необходимые права. NIST SP 800-53 AC-6 (Least Privilege) требует granular per-principal policies.

### Event Injection: почему serverless особенно уязвим

Каждая serverless-функция запускается событием: HTTP-запрос, S3-уведомление, SQS-сообщение, CloudWatch scheduled event, DynamoDB Stream record, IoT button press. С точки зрения безопасности, событие — untrusted input, поступающий в функцию извне. И serverless имеет два усугубляющих фактора по сравнению с традиционными приложениями: (a) больше источников (десятки event sources в облаке — каждое нужно валидировать), (b) разработчики склонны trust'ить облачные события — «S3:PutObject event от AWS — значит достоверный», что ошибочно (AWS доставляет event, но содержимое объекта — от пользователя, upload'ившего файл).

Классы injection в serverless:
- SQL/NoSQL injection (через непроверенные параметры в запросах к DynamoDB, RDS, Elasticsearch), OWASP ASVS V5 (Validation) требует input validation.
- OS command injection (функция, обрабатывающая filename и передающая его в imagemagick/graphicsmagick через `exec`) — должен быть prohibited в serverless; use native libraries.
- Object injection / deserialization attacks (Python pickle, Java deserialization) — если функция десериализует event payload без safe parser (JSON вместо pickle).
- Event source spoofing: S3 bucket notification — bucket должен иметь IAM policy, разрешающую Lambda invocation only from that bucket, иначе другой bucket (злоумышленника) может trigger'ить вашу функцию.

Архитектурное решение: event validation layer. Перед function handler — validation middleware, проверяющая: структура event соответствует ожидаемой схеме (JSON Schema/OpenAPI Spec для API Gateway), поля имеют корректные типы и значения (input sanitization), event source проверен (S3 bucket ARN в event records соответствует expected bucket). Если validation fail — event rejected (для async — DLQ Dead Letter Queue, для sync — HTTP 400). CIS v8 Control 3.2 (Data Protection) требует валидации ввода.

### Security per Cloud Provider: Lambda, Functions, Cloud Functions

Хотя концепция serverless общая, три основных провайдера имеют архитектурные различия, влияющие на безопасность:

AWS Lambda: execution environment — Firecracker microVM (строгая изоляция между tenant'ами). IAM-роль через execution role. Environment variables — зашифрованы at rest через AWS KMS (с помощью Lambda service key), но расшифровываются при выводе (в консоли, SDK). Lambda@Edge — выполняется в CloudFront POP, отдельный security model (execution role ограничена, нет доступа к VPC). Lambda Extensions — internal/external processes для observability, security агентов, secrets injection.

Azure Functions: execution environment — Azure App Service sandbox (менее строгая изоляция, чем Firecracker, но достаточная для single-tenant). Managed Identity вместо IAM-роли (Azure AD-based, no static credentials). Durable Functions — stateful workflow поверх stateless functions, требующие защиты intermediate state (Azure Storage encryption). Azure API Management + Functions для API Gateway security.

Google Cloud Functions: execution environment — gVisor (application kernel sandbox). Cloud Run Functions (2nd gen, 2022+) — build'ятся поверх Cloud Run, тот же execution model (Knative). IAM через service account (GCP IAM). Eventarc для unified event routing (включая third-party sources через Cloud Pub/Sub).

### Observability и мониторинг: serverless специфика

Традиционные мониторинговые подходы (агент на хосте) не работают для serverless. Вся телеметрия — через CloudWatch / Application Insights / Cloud Logging:

- **Invocation Metrics:** количество вызовов, latency (duration), error count, throttles (превышение concurrency limit). Baseline — per-function weekly patterns. Anomaly: 10x рост invocation rate с errors — возможный DDoS или exploit attempt.
- **Structured Logging:** каждая функция должна писать JSON-логи с: request ID, source IP, user identity (Cognito/IAM), action result, error stack trace. CloudWatch Logs Insights позволяет query: «все invocation'ы функции payment-processor за последние 15 минут с ошибкой» — для incident response.
- **Distributed Tracing (X-Ray, Cloud Trace):** прослеживание запроса через: API Gateway → Lambda → DynamoDB → Lambda → SNS (межсервисная propagation). Критично для обнаружения slow-burn атак: anomaly в latency на segment «обращение к Secrets Manager» может означать credential probing.
- **Dead Letter Queue (DLQ, SQS/SNS):** failed async invocations → DLQ → alert: функция не смогла обработать событие. DLQ должен иметь retention policy и monitoring на queue depth.

## Сравнение архитектурных подходов к защите Serverless

| Критерий | Внешний API Gateway + Lambda | Lambda внутри VPC + Private API Gateway | Lambda@Edge + CloudFront |
|----------|------------------------------|----------------------------------------|---------------------------|
| Сетевая экспозиция | Lambda — без VPC (AWS network), IAM-based auth only | Lambda в VPC: доступ к private resources (RDS, ElastiCache), защищена Security Groups | Выполняется в Edge POP (близко к пользователю), нет доступа к VPC, ограниченный runtime |
| Latency (p99) | Низкая (без VPC — Lambda без ENI overhead) | Средняя-высокая (Lambda в VPC = ENI attachment добавялет cold start) | Самая низкая (Edge = минимальный round-trip пользователя) |
| Защита от DDoS | WAF + Shield на API Gateway, Lambda concurrency cap | WAF на API Gateway, плюс SG ограничения на уровне VPC | Shield + WAF на CloudFront, Lambda@Edge scale ограничена CloudFront capacity |
| Секреты и конфигурация | Secrets Manager / Parameter Store через Lambda SDK | Плюс Secrets Manager доступ через VPC Endpoint (нет интернет) | Только environment variables (ограниченный размер), Secrets Manager не доступен напрямую (Edge = separate AWS region service) |
| Типовой сценарий | Публичный REST API, webhook handler, auth-функции, mobile backends | Внутренние сервисы с доступом к managed DB/cache, compliance workloads (PCI DSS) | A/B testing, request routing, authentication at edge, bot mitigation, image resize on the fly |
| Стандарты | NIST 800-53 AC-3, AC-4 (Access & Flow Enforcement через IAM + API Gateway authorizer) | NIST 800-53 AC-4 (Isolation), PCI DSS Req 1.3 (Network Segmentation) | NIST 800-207 (Zero Trust — проверка на границе сети) |

Внутренний VPC deployment — mandatory для regulated workloads. Но он дороже: Lambda в VPC требует ENI (Hyperplane ENI в современных accounts — shared per Security Group, снижает overhead), cold start дольше (ENI provisioning), и routing через NAT Gateway / VPC Endpoints требует careful planning.

Lambda@Edge — специфичный случай: функция выполняется не в регионе, а в CloudFront Point of Presence. В результате: (a) environment variables — максимум 4 KB (недостаточно для secrets), (b) нет доступа к VPC, KMS, Secrets Manager (только через internet API, если разрешён egress), (c) IAM-роль имеет дополнительные ограничения (не может запускать Lambda-функции в регионе). Применяется строго для request/response manipulation на границе CloudFront, не для business logic.

## Уроки из реальных инцидентов

### Инцидент AWS Lambda Data Leakage через reuse container (2017)

Хронология: security researcher обнаружил, что AWS Lambda может reuse execution container между инвокациями разных пользователей в multi-tenant окружении. Если tenant A создавал функцию, которая кэширует sensitive данные в global memory или writes в `/tmp`, и Lambda reused тот же контейнер для функции tenant B (в том же аккаунте, но разных user/role) — tenant B мог прочитать данные tenant A.

Корневая причина: недостаточная изоляция между execution environments одного аккаунта. Lambda в 2017 году не гарантировала, что контейнер будет очищен между инвокациями разных функций (не только разных вызовов одной функции).

Как архитектура была изменена: (a) Firecracker microVM для каждого execution environment (AWS announced in 2018, fully deployed by 2020) — гарантирует изоляцию на уровне VM (own kernel, memory), (b) `/tmp` isolation: каждый execution environment имеет собственный `/tmp`, очищенный перед reassignment, (c) mandatory non-persistence: Lambda documentation explicitly warns, что state между инвокациями не гарантирован.

Урок: serverless execution environment — managed runtime; организация не может доверять, что sensitive данные в глобальных переменных не утекут. Security через design: никогда не полагаться на persistent state между инвокациями; secrets — только через внешний Secrets Manager с per-invocation retrieve; глобальные переменные — только для connection pooling, не sensitive данных.

### Типичный сценарий инцидента: CloudTrail event injection leading to privilege escalation

Хронология: компания использует Lambda-функцию `cloudtrail-log-processor`, которая триггерится CloudTrail events через CloudWatch Events Rule. Функция парсит event, extract'ит `userIdentity.arn` и выполняет `iam:SimulatePrincipalPolicy` для анализа effective permissions этого пользователя (security tooling). Злоумышленник, имеющий ограниченные права в аккаунте, выполняет необычный API-вызов, генерирующий CloudTrail event, и в payload inject'ирует вредоносные данные в полe `userAgent` (максимально длинная строка, содержащая NUL byte). Lambda-функция не экскейпит `userAgent` при передаче в IAM API → параметр ломает `SimulatePrincipalPolicy` и вызывает неожиданный результат — function throws exception, но после retry достигает dead-letter и пишет error log со всем payload в CloudWatch. Лог виден команде, и кто-нибудь замечает подозрительный `userAgent`.

Последствия: инцидент не привёл к privilege escalation (в данном примере), но продемонстрировал: невалидация event — blind spot. В реальном exploit: функция парсит `requestParameters` из CloudTrail event (которые содержат произвольные user-provided строки — имя бакета, тег, ключ) и использует их без эскейпинга — классический second-order injection.

Урок: даже «доверенные» события (CloudTrail event от AWS сервиса) содержат user-controlled данные (`requestParameters`, `userAgent`, `sourceIPAddress`, `userIdentity` attributes). Каждое поле из event, которое передаётся в последующие API-вызовы или SQL-запросы, должно проходить ту же валидацию, что и input от внешнего пользователя.

## Инструменты и средства защиты Serverless

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| IAM Policy Analyzer & Simulator (IAM Access Analyzer, Zelkova, Parliament) | Анализ IAM-ролей на избыточные права: поиск wildcard `*`, `iam:PassRole`, cross-account access, public resource exposure | Pre-deployment (CI) + continuous scan deployed functions | NIST 800-53 AC-6 (Least Privilege), CIS v8 Control 5 |
| Serverless Vulnerability Scanner (ServerlessGoat, Prowler, tfsec, checkov) | Специализированный сканирование serverless-конфигураций: проверка наличия DLQ, encryption at rest, environment variable secrets patterns, minimum timeout, tracing enabled | CI/CD pipeline + continuous infrastructure scan | OWASP Top 10 Serverless, CIS AWS Serverless Benchmark |
| API Security / WAF (AWS WAF, Azure WAF, Cloud Armor, Lambda@Edge authorizer) | Защита API Gateway от injection, rate limiting, DDoS, schema validation (OpenAPI Spec enforcement) | Runtime: перед доставкой запроса к Lambda | OWASP ASVS V4 (Access Control), V5 (Validation), V11 (API Security) |
| Serverless Observability (Lumigo, Thundra, Epsagon, CloudWatch Lambda Insights) | Distributed tracing, cold start monitoring, invocation profiling, metric aggregation, anomaly detection on invocation patterns | Runtime: per-invocation trace context propagation | NIST 800-53 AU-2, SI-4 (Monitoring), CIS v8 Control 8 |
| Lambda Extensions / Sidecar for Security (Datadog Lambda Extension, AWS Secrets Lambda Extension) | Инжекция security-функциональности в execution environment: secrets caching, code scanning in /tmp, network egress monitoring, agent-based runtime protection | Runtime: extension lifecycle alongside Lambda function | NIST 800-53 SA-10 (Developer Security Testing), CIS v8 Control 16 |
| Software Composition Analysis (SCA) for Serverless (Snyk, Dependabot, npm audit, OWASP Dependency Check) | Сканирование зависимостей функции на известные CVE, лицензионные конфликты, вредоносные пакеты (typosquatting) | Build-time: CI/CD pipeline before deployment | OWASP A6:2021 (Vulnerable Components), CIS v8 Control 2.5 |

Serverless Vulnerability Scanner — emerging класс. В отличие от традиционных облачных секьюрити-сканеров (CSPM), они проверяют serverless-specific misconfigurations: (a) Function timeout слишком большой (макс 900 секунд для Lambda — злоумышленник может использовать для long-running C&C beacon), (b) Environment variables содержат patterns API-ключей (AKIA, azure-storage-key pattern), (c) Dead Letter Queue не настроен — async invocation failures теряются, (d) Tracing not enabled — distributed tracing off, blind spot для incident response, (e) Reserved concurrency = 0 или не настроен. CIS AWS Serverless Benchmark v1.0 (2023) документирует эти проверки.

Lambda Extensions — ключевой architectural pattern для serverless security. Extension — процесс, работающий рядом с Lambda runtime (in same execution environment), с собственным lifecycle (Init, Invoke, Shutdown phases). Используется для: (a) получение secrets из Secrets Manager на Init и кэширование in-memory (не в env vars, не в /tmp), (b) сбор телеметрии (logs, metrics, traces) и отсылка в SIEM/APM до того, как Lambda runtime завершит shutdown (guarantee log delivery even on crash/timeout), (c) eBPF-based runtime security monitoring (Falco Lambda Extension) — мониторинг неожиданных syscalls внутри функции.

## Архитектурные решения

Построение защищённой serverless-архитектуры — это integration на уровне IAM, input validation, secrets management и observability.

**Шаблон защищённой serverless-функции (golden path):**
- Deployment: Infrastructure as Code (CloudFormation/Terraform/CDK), функция и IAM-роль создаются вместе (per-function role).
- IAM: роль имеет explicit actions + resources с condition `aws:SourceArn` (запретить invocation от других ресурсов). Используется Permission Boundary (потолок прав) и IAM Path separation (функции в `/service-roles/lambda-prod/`, отделимые от human roles).
- Code: dependency scan (Trivy/Snyk) → image build → deployment to Lambda. Unit tests включают injection tests (OWASP ZAP API scan, SQLMap against API Gateway).
- Secrets: Lambda Extension получает секреты из Secrets Manager на Init, never in env vars. Secrets rotat'ятся через Secrets Manager automatic rotation.
- Configuration: environment variables contain only non-sensitive configuration. Function timeout = минимально необходимое (30 секунд для API, 5 минут для batch processing). Reserved concurrency настроен. DLQ настроен для async trigger.
- Observability: X-Ray tracing enabled, structured JSON logging with correlation ID, CloudWatch Alarm на резкий рост errors/throttles/duration.

**Метрики эффективности:**
- IAM Role Granularity Ratio: доля Lambda-функций с отдельной IAM-ролью (цель: 100% для production, запрет shared roles)
- Secrets in Environment Variables: количество функций, хранящих secrets в env vars (цель: 0 для production-critical)
- Function Timeout Compliance: процент функций с таймаутом ≤ рекомендуемого (цель: 95%+)
- DLQ Coverage: процент async-функций с настроенным DLQ (цель: 100%)
- Cold Start Anomaly Detection: время обнаружения аномального поведения функции (цель: менее 5 минут)

**Красные флаги деградации:** рост functions с `Resource: "*"` in IAM policy; появление Lambda functions с таймаутом 900 секунд (max) без documented justification; function environment variables содержат AKIA- или Azure-Key patterns; функция без CloudWatch Logs / tracing; DLQ message count растёт без consumption.

**Сценарий миграции Express-приложения на serverless:** этап 1 — разбиение монолита Express на отдельные Lambdas per route (API Gateway Lambda proxy integration). Этап 2 — внедрение validation layer (Middy.js для Node.js, Pydantic для Python) для каждого endpoint. Этап 3 — миграция статических credentials в Secrets Manager через Lambda Extension. Этап 4 — внедрение observability (Lambda Insights, X-Ray, structured logging). Этап 5 — IAM hardening: ревизия каждой роли через IAM Access Analyzer, установка Permission Boundary.

## Подготовка к собеседованию: Common pitfalls & key questions

**Типичная ошибка 1: «В serverless нет серверов — значит, нет атак на сервер, значит безопаснее».**
Атаки смещаются выше по стеку: с эксплуатации CVE в ОС/софте на сервере на exploitation injection уязвимостей в коде функции, избыточные IAM-права и компрометацию credentials. Serverless уменьшает поверхность атаки на инфраструктуру, но увеличивает на application layer — это trade-off, а не absolute improvement.

**Типичная ошибка 2: «IAM-роль с `*` на dev-стадии — норма, перед production урежем».**
Dev-роль с `AdministratorAccess` — привычка, переносимая из EC2-мира. Для Lambda это особенно опасно, потому что функция в Dev может триггериться через публичный API Gateway, и exploitation в Dev = доступ ко всему account (не только Dev resources). CIS v8 Control 4.3 требует consistent security configurations через все среды, включая Dev.

**Типичная ошибка 3: «CloudWatch Logs автоматически маскирует sensitive данные».**
Нет. CloudWatch Logs — безразличное хранилище байтов. Lambda пишет в CloudWatch Logs всё, что разработчик отправляет в `console.log(event)` — включая PII, credentials, IP адреса. Data Protection Policy в CloudWatch Logs (2022+) позволяет определить masking rules на уровне log group, но это explicit configuration, не default.

**Сложный вопрос 1: «Как обеспечить compliance (GDPR, PCI DSS) для персональных данных, обрабатываемых в serverless-функциях, где нет постоянного хранилища?»**
Проверяется: понимание data lifecycle в serverless. Ответ: данные в serverless существуют в пяти состояниях: (a) в event payload (in transit к функции), (b) в environment variables (at rest в сервисе Lambda), (c) в runtime памяти (in use), (d) в `/tmp` storage (временное локальное хранилище, 512 MB-10 GB), (e) в логах и tracing-записях. Для каждого: (a) TLS между источников события и Lambda, (b) шифрование env vars через KMS + no secrets in vars, (c) confidential computing (Lambda Nitro Enclaves) для PI, (d) `/tmp` очищается после каждого execution, автоматическое шифрование через Lambda managed key, (e) CloudWatch Logs Data Protection Policy маскирует PII patterns (email, SSN, PAN). Стандарт PCI DSS Req 3.4 требует защиты PAN везде, где он хранится — включая transient storage Lambda `/tmp` и CloudWatch logs. GDPR Art. 30 требует data flow documentation — serverless data flow map per function.

**Сложный вопрос 2: «Как защитить serverless от 'cryptojacking' — злоумышленник внедряет крипто-майнинг в Lambda через dependency compromise. Как это обнаружить и предотвратить, учитывая что Lambda может работать 15 минут макс?»**
Проверяется: знание Lambda limits и detection patterns. Ответ: Lambda execution — максимум 900 секунд (15 минут). Cryptojacking в Lambda неэффективен для злоумышленника (короткое время, нет GPU), но возможен как часть botnet для sustained low-effort mining. Detection: (a) anomaly in execution duration — функция, обычно выполняющаяся 100ms, внезапно 900 секунд каждую инвокацию (max timeout), (b) spike in CPU credits / billed duration (cost anomaly detection — ваш счёт за Lambda вырос в 10×), (c) Falco Lambda Extension обнаруживает unexpected process внутри function execution environment (crypto miner binary `xmrig`) через eBPF trace. Prevention: read-only filesystem (Lambda `/var/task` — read-only by default), dependency scanning blocking known-crypto-miner packages, Lambda code signing (Code Signing Config — deployment только из signed code, злоумышленник не может заменить код на вредоносный без доступа к signing key).

**Сложный вопрос 3: «Serverless функция обрабатывает sensitive данные и должна быть изолирована от других функций в том же аккаунте. Какие isolation mechanisms доступны на уровне облака?»**
Проверяется: глубина понимания serverless isolation. Ответ: (a) IAM isolation: separate execution role с narrow permissions, SCP на уровне Organization, запрещающий cross-function invocation (Lambda:InvokeFunction для не-whitelisted функций), (b) VPC isolation: функция в separate VPC с отдельной подсетью, NACL запрещает трафик в другие VPC, (c) Lambda@Edge / CloudFront Functions — separate environment от regional Lambda, (d) Execution environment isolation: Firecracker microVM обеспечивает аппаратную изоляцию между execution environments разных функций, (e) Resource isolation: функция работает с отдельным KMS-ключом (decrypt доступ только этому ключу в IAM-политике), отдельным Secrets Manager secret path (`/my-sensitive-function/*`). Архитектурный компромисс: VPC для функции добавляет latency, но mandatory для compliance (PCI DSS network segmentation). NIST SP 800-53 SC-7 (Boundary Protection) и AC-4 (Info Flow Enforcement) требуют изоляции информационных потоков между системами разного уровня.

## Чек-лист понимания

- [ ] Почему serverless требует переноса защиты с host-based на IAM-based контроли, и какой класс атак от этого не закрывается?
- [ ] В каком случае event injection через API Gateway может обойти WAF, защищающий Lambda?
- [ ] Как cold start reuse контейнера может привести к data leakage, даже если разработчик не использует глобальные переменные?
- [ ] Почему Lambda execution role с `iam:PassRole` — одна из самых опасных misconfigurations в serverless?
- [ ] В каком случае serverless observability (CloudWatch/X-Ray) не даст достаточно информации для post-incident forensics?
- [ ] Как Lambda@Edge и CloudFront Functions архитектурно ограничены в безопасности по сравнению с regional Lambda?
- [ ] Почему максимальный Lambda timeout (900 секунд) — security risk, а не только cost risk?
- [ ] В каком случае использование Lambda Extension для security-агента хуже, чем его отсутствие?
- [ ] Как обеспечить, что одна команда не может invoke (вызвать) Lambda-функцию другой команды в том же AWS-аккаунте?
- [ ] Почему serverless-функции особенно уязвимы к supply-chain атакам через зависимости, и как SBOM помогает?

### Ответы на чек-лист

1. **Ответ:** Serverless убирает ОС и хост из зоны видимости — HIDS, файловый integrity monitoring, kernel hardening не применимы. Защита переносится на IAM (роль функции), код (input validation) и observability (structured logging). Но этот подход не закрывает: runtime уязвимости (RCE в managed runtime — Java RCE через deserialization, Node.js `eval` injection), supply-chain атаки (вредоносные зависимости в npm/pip), и уязвимости в облачных сервисах-триггерах (CloudTrail event injection). Эти требуют дополнительных контролей: SAST/DAST, SCA, vulnerability scanning на уровне кода. NIST SP 800-53 SA-11 (Developer Testing) требует проверки на все классы уязвимостей.

2. **Ответ:** WAF — inline перед API Gateway (или интегрирован в API Gateway). Он инспектирует HTTP-запрос и блокирует известные attack patterns (SQL injection keywords, XSS scripts). Если атакующий использует: (a) encoding обход (URL-encoded, double URL-encoded) — например `%27` вместо `'`, и WAF не выполняет recursive decoding, (b) custom protocol (WebSocket upgrade — WAF инспектирует только initial handshake, не последующие messages), (c) чрезмерно длинный payload (WAF имеет body size limit, после чего пропускает без инспекции). API Gateway должен конфигурироваться с strict OpenAPI schema validation (request body must match schema) как second layer.

3. **Ответ:** Даже без глобальных переменных: (a) Lambda runtime может кэшировать state в internal structures (AWS SDK connection pool, HTTP keep-alive connections с заголовками предыдущего запроса), (b) `/tmp` — если функция использует `/tmp` как кэш и не очищает его явно между инвокациями, следующий invocation может читать файлы, оставленные предыдущим (Lambda `/tmp` не гарантирует очистку между инвокациями одной функции), (c) Extension кэши — Lambda Extensions имеют own жизненный цикл и могут хранить данные между инвокациями. Решение: не полагаться на persistent `/tmp` или глобальный state; явно очищать `/tmp` в начале каждого invocation (или использовать per-invocation temp directory с random UUID).

4. **Ответ:** `iam:PassRole` позволяет функции передать её IAM-роль другому сервису (например, создать новую Lambda или EC2 с текущей ролью). Если функция с правами `s3:*`, `iam:PassRole` и `lambda:CreateFunction` скомпрометирована, злоумышленник может: создать новую Lambda с той же ролью (доступ к тем же ресурсам), под своим контролем, и запустить её — escalation до persistent access. В serverless это critical, потому что функция работает с временными credentials (STS), и PassRole — путь к персистенции. IAM-политика роли должна иметь `Deny iam:PassRole` (или разрешать только для строго whitelisted сервисов с condition `iam:PassedToService`). OWASP Serverless Top 10 (Injection) требует внимания к ролевым политикам.

5. **Ответ:** Forensics требует: (a) что произошло (event payload, function response), (b) кто инициировал (identity, source IP), (c) временная шкала (когда, сколько выполнялась), (d) downstream effects (какие другие сервисы были вызваны, с какими параметрами). CloudWatch Logs + X-Ray дают (a-c), но для (d) нужно: VPC Flow Logs (если функция в VPC и вызывала другие сервисы по сети), CloudTrail Data Events (если функция обращалась к S3/DynamoDB — видно, какие объекты), и downstream сервис-логи (RDS query logs, если функция делала SQL-запросы). Serverless forensics — distributed tracing, а не одна функция.

6. **Ответ:** Lambda@Edge выполняется в CloudFront PoP, не в регионе. Ограничения: (a) нет доступа к VPC — функция не может обращаться к regional RDS/ElastiCache или VPC Endpoints, (b) environment variables — максимум 4KB (недостаточно для больших конфигураций), (c) IAM-роль имеет дополнительные guardrails (не может invoke regional Lambda, не может запускать EC2, не может access большинство сервисов через VPC), (d) execution environment — отдельный от regional Lambda, с отдельными лимитами. CloudFront Functions — ещё более ограничены: JavaScript ECMAScript 5.1, только HTTP manipulation, duration < 1ms. Применение: только request/response manipulation на границе; business logic, database access — только regional Lambda.

7. **Ответ:** (a) Max timeout = злоумышленник, получивший execution в функции, может держать connection открытым 15 минут (C&C beacon, data exfiltration через медленный канал), (b) 900 секунд × 1000 concurrent execution = 900000 секунд compute в минуту для атакующего (cryptojacking, хотя неэффективно, но объем), (c) Billed Duration: 900-секундная функция × много инвокаций = огромный Lambda bill (Denial of Wallet), (d) если Lambda делает синхронные external API вызовы (HTTP request к third-party), timeout 900 секунд позволяет использовать Lambda как proxy для атаки с вашего legit IP. Best practice: timeout = 3× ожидаемое время выполнения (для API — 30 секунд, для batch — 5 минут).

8. **Ответ:** Lambda Extension добавляет overhead: (a) холодный старт дольше (Lambda runtime + extension init), (b) extra cost (billed for extension execution time + extension requests/response lifecycle), (c) shared execution environment — уязвимость в extension может скомпрометировать функцию (например, RCE в Datadog Lambda Extension), (d) если extension выполняет синхронные outbound calls (SIEM ingestion) и network задержка большая — function latency ухудшается. Extension оправдан, когда добавляет unique security value (secrets caching, eBPF runtime protection) и overhead acceptable; иначе — native Lambda observability достаточно.

9. **Ответ:** (a) IAM Policy per function role: разрешить `lambda:InvokeFunction` только для whitelisted source ARNs (например, API Gateway конкретного API + specific S3 bucket), (b) SCP на Organization level: deny `lambda:InvokeFunction` cross-team аккаунтам (если команды в разных аккаунтах), (c) Lambda Resource Policy: function URL / invocation policy контролирует, кто может trigger (API Gateway, specific EventBridge rule, specific S3 bucket) — злоумышленник не может invoke напрямую через AWS CLI даже с IAM-ролью другого сервиса, если resource policy deny, (d) Tag-based condition в IAM-политике: разрешить invocation только функций с tag `team: "checkout-team"`, deny остальным.

10. **Ответ:** Serverless функции — минималистичный runtime (Alpine, Node.js), где все зависимости, кроме стандартной библиотеки, — через package manager (npm/pip). Supply-chain attack: вредоносный пакет в npm, подтянутый через `npm install`, даёт злоумышленнику RCE внутри Lambda (no OS hardening, no binary whitelisting). SBOM (Software Bill of Materials) — per-function manifest всех зависимостей с версиями и provenance. При обнаружении CVE в библиотеке (Log4Shell) — SBOM позволяет query: «в каких функциях эта библиотека?» за секунды. Также SBOM интегрируется с admission control: Lambda deployment blocked, если SBOM включает unapproved package (not in allowlist). Executive Order 14028 и CIS v8 Control 2.5 требуют SBOM.

## Ключевые выводы для собеседования

- **Самый критичный принцип:** Serverless меняет trust boundary: вы отдаёте провайдеру всё от железа до runtime, оставляя себе только код и IAM. Безопасность смещается в две плоскости — качество кода (input validation, dependencies) и гранулярность IAM-политик. Провал в любой = компрометация.
- **Главный компромисс:** Serverless уменьшает операционную нагрузку (не нужно патчить ОС, настраивать автоскейлинг) ценой уменьшения контроля (меньше visibility в runtime, нет host-based защиты). Архитектор должен компенсировать это через observability и тестирование на этапе CI/CD.
- **Связь с ключевым стандартом:** OWASP Top 10 Serverless (event injection, broken auth, sensitive data exposure) и NIST SP 800-53 AC-6 (Least Privilege per function IAM role) — два ориентира. Без least-privilege IAM любая function exploit = account takeover.
- **Практическое правило:** Serverless — не legacy app, перенесённый в Lambda. Эффективная безопасность требует архитектуры, ориентированной на функции: stateless design, external secrets, mandatory input validation на каждый event type, и IAM-роль ровно на 1 функцию.

---
_Статья создана на основе анализа OWASP Top 10 Serverless (2023), NIST SP 800-53 AC-6/SI-4/AU-2, CIS AWS Serverless Benchmark v1.0, официальной документации серверлес-сервисов (AWS Lambda, Azure Functions, GCP Cloud Functions, Cloud Run), материалов по инцидентам (AWS Lambda data leakage 2017, Codecov supply-chain 2021), open-source инструментов (Serverless Goat, Prowler, Falco Lambda Extension) и архитектурных best practices AWS Well-Architected Serverless Lens._
