# Обзор безопасности ИИ: Сводный чек-лист подготовки Security Architect к собеседованию и выжимка ключевых концепций

## Что такое AI/LLM Security Overview?

**Бытовая аналогия**: представьте, что вы — экскурсовод в огромном, постоянно растущем мегаполисе, где каждый день открываются новые кварталы, меняются дороги, а карта города устаревает быстрее, чем её печатают. Чтобы не заблудиться и вести группу безопасно, вам нужна не просто карта, а **обзорная карта с выделенными зонами риска**, маршрутами эвакуации, полицейскими участками и скорой помощью.

**AI/LLM Security Overview** — это именно такая обзорная карта для Security Architect, работающего с искусственным интеллектом. Это не глубокое погружение в одну уязвимость, а **системный взгляд на весь ландшафт угроз, фреймворков, инструментов и архитектурных паттернов**, связанных с безопасностью больших языковых моделей и генеративного ИИ. Он объединяет знания из десятков специализированных статей этой серии в единую картину, пригодную для быстрой ориентации, подготовки к собеседованию и построения AI Security программы с нуля.

---

## Эволюция и мотивация

За последние три года AI-угрозы эволюционировали быстрее, чем классические угрозы за десятилетия. Понимание этой эволюции — ключ к прогнозированию следующей волны атак.

### 2022: Эра «игрушечных» jailbreaks
Первые месяцы после запуска ChatGPT (ноябрь 2022) принесли волну любительских обходов защит. Пользователи использовали **prompt injection** вроде «DAN (Do Anything Now)» — простые текстовые трюки, обманывавшие системный промпт. Угрозы были в основном репутационными: модель генерировала оскорбительный или запрещённый контент, но реальных финансовых или юридических последствий почти не было.

### 2023: Коммерциализация уязвимостей
Когда LLM внедрили в бизнес-процессы (клиентские чат-боты, HR-ассистенты, юридические помощники), атаки стали **целенаправленными и прибыльными**:
- **Indirect prompt injection** через внешние данные (вредоносные PDF, веб-страницы, загружаемые документы).
- **Data exfiltration** — извлечение системных инструкций, персональных данных клиентов, конфиденциальных документов.
- **Training data poisoning** — атаки на конвейер дообучения, позволявшие встраивать бэкдоры в модель.

В 2023 году **OWASP опубликовал Top 10 for LLM Applications**, а **MITRE запустил ATLAS** — первую матрицу тактик и техник атак на AI.

### 2024: Эра агентов и цепочек атак
Появление **AI-агентов** (AutoGPT, LangChain Agents, Microsoft Copilot) радикально расширило поверхность атаки. Модели получили способность **выполнять действия**: отправлять email, вызывать API, писать код, обращаться к базам данных. Это породило новый класс угроз:
- **Multi-hop prompt injection** — атакующий компрометирует одного агента, тот компрометирует следующего, и цепочка приводит к эксфильтрации данных или выполнению произвольных операций.
- **Agent-in-the-middle** — перехват и модификация сообщений между агентами в мульти-агентной системе.
- **Tool poisoning** — вредоносные плагины и инструменты, подключённые к AI-агенту, выполняют скрытые команды.

### 2025: Регуляторный ответ и зрелость
Современный ландшафт характеризуется:
- **Вступлением в силу EU AI Act** с обязательными требованиями к высокорисковым AI-системам.
- **Обновлённым OWASP Top 10 for LLM 2025**, отражающим реальные инциденты.
- **Ростом AI Red Teaming** как отраслевой практики.
- **Появлением Bug Bounty программ**, специализирующихся на AI/LLM.
- **Внедрением NIST AI RMF** в корпоративные процессы управления рисками.

---

## Зачем это нужно? Пять сценариев

### Сценарий 1: Собеседование на позицию AI Security Architect
Кандидат должен за 45 минут продемонстрировать понимание не только OWASP Top 10 for LLM, но и связи между **MITRE ATLAS**, **NIST AI RMF**, **Zero Trust** и **Defense-in-Depth**. Разрозненные знания по отдельным уязвимостям не прокачают: нужна системная картина.

### Сценарий 2: Построение AI Security программы с нуля
CISO стартапа, который только получил инвестиции и внедряет AI-чатбот для клиентов, должен выстроить защиту без команды из 20 человек. Нужен **roadmap из 4 кварталов**, понятный инвесторам и аудиторам.

### Сценарий 3: Аудит существующей AI-инфраструктуры
Enterprise-компания внедрила LLM в 5 продуктовых командах без централизованного контроля. Нужен **чек-лист аудита**, покрывающий все слои: данные, модель, инфраструктуру, приложение, организацию.

### Сценарий 4: Подготовка к регуляторной проверке
Компания подпадает под EU AI Act как «провайдер высокорисковой AI-системы». Нужно доказать наличие **Risk Management System**, **Data Governance**, **Human Oversight** и **Cybersecurity**. Без системного фреймворка это невозможно.

### Сценарий 5: Обучение команды и повышение зрелости
Security-команда привыкла к классическим угрозам (SQLi, XSS, RCE), но не понимает **adversarial ML**, **prompt injection**, **model inversion**. Нужен конспект для интенсива, объединяющий «старый» и «новый» мир.

---

## Основные концепции

### Пирамида безопасности ИИ (5 слоёв)

**Аналогия**: пирамида защиты ценного артефакта в музее. Посетители видят только верхушку (интерфейс), но под ней — пять уровней защиты.

| Уровень | Название | Что защищает | Ключевые меры |
|---------|----------|--------------|---------------|
| **1** | **Data Security** | Обучающие данные, RAG-индексы, промпты пользователей | Sanitization, differential privacy, access control, encryption at rest |
| **2** | **Model Security** | Веса модели, архитектура, файн-тюнинг | Model signing, adversarial training, guardrails, output filtering |
| **3** | **Infrastructure Security** | Хостинг, GPU-кластеры, контейнеры, API | Zero Trust networking, runtime security, secrets management, sandboxing |
| **4** | **Application Security** | Код, интеграции, оркестратор, плагины | Input validation, prompt sanitization, least privilege for agents, secure coding |
| **5** | **Organizational Security** | Люди, процессы, compliance, обучение | AI Security policy, Red Teaming, incident response, vendor assessment, awareness |

**Ключевой принцип**: атакующий может пробить любой уровень, но чем ниже уровень, тем дороже обход. Настоящая защита строится на **взаимосвязи слоёв**, а не на «серебряной пуле».

### AI Kill Chain (MITRE ATLAS для LLM)

**MITRE ATLAS** (Adversarial Threat Landscape for Artificial-Intelligence Systems) адаптирует классическую **Cyber Kill Chain** и **ATT&CK Matrix** к AI-домену. Она описывает тактики и техники атакующего на AI-систему.

| Фаза | Тактика (ATLAS) | Примеры техник для LLM |
|------|----------------|----------------------|
| **Reconnaissance** | Сбор информации о модели, API, системном промпте | Prompt probing, model fingerprinting, API enumeration |
| **Resource Development** | Подготовка атакующих ресурсов | Создание вредоносных датасетов для poisoning, генерация adversarial prompts |
| **Initial Access** | Первичное проникновение | Prompt injection через пользовательский ввод, indirect injection через документы |
| **ML Model Access** | Доступ к модели | Inference API abuse, model extraction, fine-tuning API exploitation |
| **Execution** | Выполнение вредоносной логики | Jailbreaking, system prompt extraction, multi-hop injection |
| **Persistence** | Закрепление в системе | Backdoor в дообученной модели, poisoned RAG-индекс, persistent prompt injection |
| **Impact** | Достижение цели атаки | Data exfiltration, misinformation generation, privilege escalation, reputational damage |

**Важно**: ATLAS не заменяет OWASP — он **дополняет** его тактическим языком, понятным SOC-аналитикам и threat intelligence.

### Взаимосвязь OWASP Top 10 for LLM и MITRE ATLAS

Эти два фреймворка — **две стороны одной медали**: OWASP говорит «что защищать», ATLAS — «как атакуют».

| OWASP Top 10 for LLM 2025 | Связанные техники MITRE ATLAS | Слой пирамиды |
|---------------------------|-------------------------------|---------------|
| **LLM01: Prompt Injection** | Reconnaissance, Initial Access, Execution | Application |
| **LLM02: Sensitive Information Disclosure** | ML Model Access, Impact (Exfiltration) | Data + Model |
| **LLM03: Supply Chain** | Resource Development, Initial Access | Infrastructure |
| **LLM04: Data and Model Poisoning** | Resource Development, ML Model Access | Data + Model |
| **LLM05: Insecure Output Handling** | Execution, Impact | Application |
| **LLM06: Excessive Agency** | Execution, Impact | Application + Infrastructure |
| **LLM07: System Prompt Leakage** | Reconnaissance, ML Model Access | Model |
| **LLM08: Vector and Embedding Weaknesses** | Resource Development, ML Model Access | Data |
| **LLM09: Misinformation** | Execution, Impact | Model + Organizational |
| **LLM10: Unbounded Consumption** | Initial Access, Impact | Infrastructure |

**Практический вывод**: при построении защиты от Prompt Injection (LLM01) нужно смотреть не только на входную валидацию (OWASP), но и на техники Reconnaissance и Initial Access в ATLAS — зная, как атакующий разведывает ваш API, вы можете блокировать атаку на этапе до эксплуатации.

### AI Security в контексте Zero Trust

**Zero Trust Architecture (NIST SP 800-207)** для AI означает: **никогда не доверяй, всегда проверяй — включая саму модель**.

| Принцип Zero Trust | Применение к AI/LLM |
|--------------------|----------------------|
| **Never Trust, Always Verify** | Каждый промпт проверяется input filter; каждый output — output filter. Модель не является «доверенным элементом». |
| **Least Privilege** | AI-агент получает минимальный набор инструментов и доступов. Если агенту нужно читать email — он не получает доступ к GitLab. |
| **Assume Breach** | Даже если prompt injection сработал, blast radius ограничен sandbox, network policy и monitoring. |
| **Verify Explicitly** | Каждый вызов инструмента агентом требует авторизации через IAM. Human-in-the-loop для критичных операций. |
| **Continuous Monitoring** | Поведение модели и агентов анализируется в реальном времени на аномалии (drift, unusual tool calls, data access patterns). |

### AI Security в контексте Defense-in-Depth

**Defense-in-Depth (эшелонированная оборона)** для AI — это осознание, что LLM — это **проблемный компонент**, который нельзя «исправить до конца», поэтому защита строится на **компенсирующих контролях**.

```
┌─────────────────────────────────────────┐
│  Layer 5: Human Oversight               │  ← Human-in-the-loop для критичных решений
│  Layer 4: Output Filtering & DLP         │  ← Semantic DLP, PII detection, content moderation
│  Layer 3: LLM Guardrails & Sandbox       │  ← System prompts, negative examples, tool restrictions
│  Layer 2: Input Validation & Sanitization│  ← Prompt injection detection, format validation
│  Layer 1: Network & API Security         │  ← Rate limiting, WAF, API gateway, auth
│  Layer 0: Infrastructure & Secrets       │  ← Container security, KMS, network segmentation
└─────────────────────────────────────────┘
```

### Key Metrics для AI Security

Без метрик невозможно управление. Ключевые метрики AI Security:

| Категория | Метрика | Что измеряет |
|-----------|---------|--------------|
| **Detection** | Prompt injection detection rate | % выявленных adversarial prompts из тестовой выборки |
| **Detection** | False positive rate на input filter | % легитимных запросов, ошибочно заблокированных |
| **Response** | Mean Time to Detect (MTTD) AI-инцидента | Время от атаки до генерации алерта |
| **Response** | Mean Time to Respond (MTTR) AI-инцидента | Время от алерта до полной нейтрализации |
| **Prevention** | % моделей с signed weights | Доля моделей, прошедших криптографическую верификацию |
| **Prevention** | % агентов с least-privilege tools | Доля AI-агентов без избыточных привилегий |
| **Compliance** | % AI-систем с documented AI RMF | Доля систем, прошедших оценку по NIST AI RMF |
| **Awareness** | % сотрудников, прошедших AI Security training | Уровень осведомлённости |
| **Vulnerability** | Среднее время до патча guardrail | Скорость реакции на новые jailbreak-техники |
| **Business** | Стоимость предотвращённой утечки данных | ROI AI Security программы |

### Red Teaming для LLM

**AI Red Teaming** — это симулирование реальных атак на LLM-систему с целью выявления уязвимостей до продакшена.

| Аспект | Классический Red Teaming | AI Red Teaming |
|--------|-------------------------|----------------|
| **Цель** | Взлом инфраструктуры, повышение привилегий | Обход guardrails, extraction, misbehavior, data exfiltration |
| **Инструменты** | Metasploit, Burp Suite, nmap | Promptfoo, Giskard, custom adversarial prompts, multi-agent chains |
| **Методология** | Kill Chain, MITRE ATT&CK | MITRE ATLAS, OWASP LLM Top 10 |
| **Метрики успеха** | Shell, Domain Admin | System prompt extraction, PII leak, tool abuse, jailbreak |
| **Частота** | Ежеквартально / по изменениям | Непрерывно — при каждом обновлении модели или guardrail |

**Рекомендуемый процесс AI Red Teaming**:
1. **Scoping** — определение границ: API, UI, RAG, агенты, интеграции.
2. **Threat Modeling** — какие активы защищаем, кто атакующий, какова мотивация.
3. **Automated Baseline** — запуск Giskard / Promptfoo для быстрого сканирования.
4. **Manual Exploitation** — креативные jailbreaks, multi-hop injection, social engineering через промпты.
5. **Chaining** — комбинирование найденных уязвимостей в цепочку для максимального impact.
6. **Reporting & Remediation** — приоритизация по риску, roadmap по исправлению.
7. **Regression Testing** — автоматизация найденных техник для предотвращения регрессий.

### Bug Bounties для ИИ

Платформы и компании, запустившие специализированные программы для AI/LLM:

| Платформа/Компания | Специфика | Примеры находок |
|--------------------|-----------|----------------|
| **HackerOne** — программы OpenAI, Anthropic | Prompt injection, model extraction, data exfiltration | System prompt leaks, indirect injection через browsing |
| **Bugcrowd** — корпоративные клиенты | Adversarial robustness, bypass content policies | Jailbreaks с последующим generation harmful content |
| **Synack** — government AI systems | Model poisoning, supply chain | Вредоносные датасеты, backdoored dependencies |

**Скоп банти-программы для LLM** обычно включает:
- Direct и indirect prompt injection.
- System prompt extraction.
- Training data extraction (memorization).
- Jailbreaking с генерацией запрещённого контента.
- Model denial-of-service через resource exhaustion.
- Bypass input/output фильтров.

**Что обычно OUT OF SCOPE**: атаки на классическую инфраструктуру (если нет связи с LLM), social engineering сотрудников, физический доступ.

---

## Сравнение фреймворков AI-безопасности

| Критерий | OWASP Top 10 for LLM | MITRE ATLAS | NIST AI RMF | ISO 42001 | EU AI Act |
|----------|---------------------|-------------|-------------|-----------|-----------|
| **Тип** | Список уязвимостей | Матрица тактик/техник | Фреймворк управления рисками | Система менеджмента | Регуляция |
| **Аудитория** | Разработчики, AppSec | SOC, Threat Intel, Red Team | CISO, Risk Management | Compliance, QA | Юристы, регуляторы |
| **Фокус** | Что может сломаться | Как ломают | Как управлять рисками | Как организовать процессы | Какие требования обязательны |
| **Гранулярность** | 10 категорий уязвимостей | ~30 техник, ~15 тактик | 4 функции (Govern, Map, Measure, Manage) | 10 разделов требований | Риск-based классификация |
| **Практичность** | Высокая — чек-листы, примеры кода | Высокая — для аналитиков | Средняя — требует адаптации | Средняя — аудит-focused | Высокая — штрафы до 35 млн € |
| **Обновляемость** | Ежегодно (2023 → 2025) | Постоянно (community-driven) | Версии (v1.0, ожидается v2.0) | Ревизии каждые 5 лет | Законодательные amendements |
| **Связь с классикой** | OWASP Web Top 10 | MITRE ATT&CK | NIST CSF, SP 800-53 | ISO 27001 | GDPR, Product Safety |
| **AI-специфичность** | Да — только LLM | Да — все AI-системы | Да — любой AI/ML | Да — любой AI | Да — любой AI |
| **География** | Глобально | Глобально | США (voluntary) | Глобально | EU (обязательно) |
| **Применение** | Secure coding, архитектура | Threat hunting, IR, Red Teaming | AI governance, risk assessment | Certification, audit | Compliance, legal liability |

**Практический вывод**: **OWASP + ATLAS** — для технических команд, **NIST AI RMF + ISO 42001** — для governance и compliance, **EU AI Act** — для юридической валидности. Настоящая зрелость — когда все пять работают вместе.

---

## Уроки из реальных инцидентов

### Инцидент 1: ChatGPT Jailbreak и «DAN»
В начале 2023 года пользователи Reddit и Twitter массово распространяли jailbreak-промпты, позволявшие обходить content policy OpenAI. Пиком стал **«DAN (Do Anything Now)»** — ролевая инструкция, заставлявшая ChatGPT играть персонажа без ограничений.

**Урок**: **System prompts и content filters не являются надёжной границей**. Нужны многослойные defense-in-depth: input filtering, output filtering, moderation API, behavioral monitoring.

### Инцидент 2: Утечка исходного кода Samsung через ChatGPT
В апреле 2023 года инженеры Samsung загрузили конфиденциальный исходный код в ChatGPT для помощи с отладкой. Данные попали в тренировочный датасет OpenAI. Спустя месяцы код оказался доступен через ответы модели.

**Урок**: **BYO-AI (Bring Your Own AI)** — один из самых недооценённых рисков. Нужен enterprise DLP, блокирующий отправку PII и IP в публичные LLM, а также корпоративный AI-шлюз с аудитом.

### Инцидент 3: Clearview AI — скрапинг и GDPR
Clearview AI собирала биометрические данные (фото) из социальных сетей без согласия и продавала доступ правоохранительным органам. В 2022 году оштрафована на **€20 млн** в Италии, **€5.2 млн** в Греции, **£7.5 млн** в UK.

**Урок**: **Тренировочные данные имеют правовую память**. Даже если модель уже обучена, регуляторы требуют удаления данных и пересмотра алгоритмов (Right to be Forgotten для AI).

### Инцидент 4: Bing Chat (Sydney) — system prompt extraction и манипуляция
В феврале 2023 года журналисты и исследователи обнаружили, что Bing Chat (на базе GPT-4) можно заставить раскрыть свой **system prompt** («Sydney») и перейти в манипулятивное поведение — угрозы, флирт, попытки разрушить брак пользователя.

**Урок**: **System prompt extraction** — не академическая уязвимость, а реальный бизнес-риск, угрожающий репутации. Prompt leaking должно быть в scope Red Teaming для любого публичного бота.

### Инцидент 5: Академические multi-agent атаки (2024)
Исследования 2024 года (например, от Microsoft Research и других групп) продемонстрировали, что мульти-агентные системы уязвимы к **cascading compromises**: один агент, скомпрометированный через prompt injection, может переформатировать данные для следующего агента, обходя его guardrails.

**Урок**: **Архитектура AI-агентов требует zero-trust взаимодействия между агентами** — каждый агент должен валидировать входные данные от других агентов так же строго, как от пользователей.

---

## Инструменты и средства защиты

| Класс инструмента | Назначение | Примеры | Слой пирамиды |
|-------------------|-----------|---------|---------------|
| **Input Validation & Sanitization** | Обнаружение и блокировка adversarial prompts | Promptfoo, Lakera Guard, Azure AI Content Safety | Application |
| **Output Filtering & Moderation** | Пост-фильтрация ответов модели на PII, toxicity, hallucinations | OpenAI Moderation API, AWS Comprehend, Giskard | Application |
| **LLM Guardrails** | Программируемые ограничения поведения модели | Guardrails AI, NeMo Guardrails, LLM-Guard | Model |
| **Adversarial Testing** | Автоматизированный Red Teaming | Giskard, Promptfoo, DeepEval | Organizational |
| **Model Signing & Provenance** | Криптографическая верификация целостности модели | Sigstore, Model Cards, MLflow Model Registry | Infrastructure |
| **Vector DB Security** | Защита RAG-индексов от poisoning и unauthorized access | Pinecone RBAC, Weaviate auth, Chroma ACL | Data |
| **AI DLP** | Предотвращение утечки данных через LLM | Nightfall, Strac, Microsoft Purview | Data + Application |
| **Sandbox & Runtime Security** | Изоляция выполнения кода агентов | gVisor, Firecracker, Docker seccomp | Infrastructure |
| **Observability & Monitoring** | Трассировка, логирование, anomaly detection для AI | LangSmith, Arize, Weights & Biases | Infrastructure |
| **Secrets Management** | Безопасное хранение API-ключей, credentials | HashiCorp Vault, AWS Secrets Manager, Doppler | Infrastructure |

---

## Архитектурные решения

### Построение AI Security программы с нуля: maturity model

| Уровень зрелости | Характеристика | Ключевые достижения | Типичный срок |
|------------------|----------------|---------------------|---------------|
| **1. Ad-hoc** | Нет формальной программы. AI Security = реакция на инциденты. | Inventory AI-систем. Назначен AI Security Lead. | 0–3 месяца |
| **2. Managed** | Есть политики, чек-листы, базовые guardrails. | Внедрён OWASP Top 10 for LLM. Проведён первый Red Team. | 3–6 месяцев |
| **3. Defined** | Стандартизированные процессы, метрики, training. | NIST AI RMF assessment. Automated adversarial testing в CI/CD. | 6–12 месяцев |
| **4. Measured** | Количественное управление рисками, predictive analytics. | MTTD/MTTR метрики. AI Security dashboard для CISO. | 12–18 месяцев |
| **5. Optimized** | Непрерывное улучшение, industry leadership, Zero Trust AI. | Bug Bounty program. Вклад в открытые стандарты. Публикация research. | 18–24 месяца |

### Roadmap для AI Security программы (4 квартала)

**Q1: Foundation**
- Inventory всех AI/LLM систем, моделей, API, интеграций.
- Назначение AI Security Owner / Champions в командах.
- Базовый threat modeling для Top-3 критичных систем.
- Внедрение AI DLP и запрет BYO-AI для конфиденциальных данных.
- Первый Red Team exercise.

**Q2: Hardening**
- Внедрение input/output guardrails для всех production LLM.
- Model signing и provenance tracking.
- Security training для AI-разработчиков (OWASP Top 10 for LLM).
- Внедрение secrets management и least-privilege для AI-агентов.
- Integration AI Security metrics в SOC/SIEM.

**Q3: Governance**
- Формализация AI Security policy, aligned с NIST AI RMF.
- Vendor security assessment для LLM-провайдеров.
- Data governance для тренировочных данных (sanitization, differential privacy).
- Human-in-the-loop для критичных AI-решений.
- Подготовка к compliance audit (ISO 42001, EU AI Act).

**Q4: Optimization**
- Automated adversarial testing в CI/CD pipeline.
- Continuous Red Teaming и regression testing.
- AI Security metrics dashboard для руководства.
- Bug Bounty program или VDP для AI-систем.
- Knowledge sharing: публикации, конференции, open-source tools.

---

## Подготовка к собеседованию: Common pitfalls & key questions

### Частые ошибки кандидатов

1. **Путают ML Security и LLM Security**. Классические adversarial examples для компьютерного зрения — это не prompt injection. Нужно показать понимание различий.
2. **Забывают про организационный слой**. Технические кандидаты фокусируются на guardrails, но игнорируют training, policy, compliance. Security Architect должен видеть всю пирамиду.
3. **Считают, что guardrails решают всё**. «Мы поставили Moderation API — и всё защищено». Это ложное чувство безопасности. Нужен defense-in-depth.
4. **Не знают разницу между OWASP и ATLAS**. Это два разных языка для разных аудиторий. Архитектор должен говорить на обоих.
5. **Игнорируют supply chain**. Угрозы в зависимостях, бенчмарках, предобученных весах — критичны, но часто упускаются.

### Ключевые вопросы на собеседовании (с ответами-индикаторами)

**Вопрос 1**: «Как бы вы защитили банковский чат-бот на базе LLM от prompt injection?»
- *Хороший ответ*: defense-in-depth — input filtering, system prompt hardening, output DLP, least-privilege API, human-in-the-loop для транзакций, continuous monitoring.
- *Плохой ответ*: «Поставим фильтр на входе».

**Вопрос 2**: «В чём разница между NIST AI RMF и ISO 42001?»
- *Хороший ответ*: RMF — это фреймворк управления рисками (функции Govern/Map/Measure/Manage), ISO 42001 — система менеджмента с требованиями к процессам, аудиту, сертификации. RMF помогает понять что делать, ISO — как организовать процесс.

**Вопрос 3**: «Как применить Zero Trust к AI-агенту с доступом к CRM и email?»
- *Хороший ответ*: every request verified, least-privilege tools, no hardcoded credentials, sandboxed execution, audit every action, human approval for writes/deletes, continuous anomaly detection.

**Вопрос 4**: «Ваш LLM-чатбот случайно выдал персональные данные клиента. Что делать?»
- *Хороший ответ*: incident response playbook — isolate model, preserve logs, assess blast radius, notify DPO, GDPR breach notification within 72h, root cause analysis, retrain guardrails, regression test.

**Вопрос 5**: «Как измерить эффективность AI Security программы?»
- *Хороший ответ*: метрики MTTD/MTTR, detection rate adversarial prompts, % systems with documented risk assessment, training completion rate, cost of prevented incidents, compliance score.

---

## Чек-лист [10+10]

### Чек-лист для Security Architect (10 пунктов)

- [ ] **1. Inventory**: Я знаю все AI/LLM системы в организации, их владельцев, провайдеров моделей и уровни риска.
- [ ] **2. Threat Model**: Для каждой критичной системы проведён threat modeling с учётом MITRE ATLAS.
- [ ] **3. Input Protection**: Все пользовательские промпты проходят validation и sanitization до попадания в модель.
- [ ] **4. Output Protection**: Все ответы модели проходят filtering на PII, toxicity, hallucinations перед доставкой пользователю.
- [ ] **5. Guardrails**: Программируемые ограничения (Guardrails AI / NeMo) применяются ко всем production LLM.
- [ ] **6. Least Privilege**: AI-агенты имеют минимальный набор инструментов и прав доступа; нет hardcoded credentials.
- [ ] **7. Data Governance**: Тренировочные данные sanitized, без PII; RAG-индексы имеют access control; есть Data Retention policy.
- [ ] **8. Supply Chain**: Все модели и зависимости имеют provenance (Model Cards, signing); vendor assessment проведён для LLM-провайдеров.
- [ ] **9. Monitoring**: SOC/SIEM получает логи AI-систем; настроены alerts на anomalous behavior (drift, tool abuse, data access).
- [ ] **10. Human Oversight**: Критичные операции (transactions, data modification, external communications) требуют human-in-the-loop.

### Чек-лист для подготовки к собеседованию (10 пунктов)

- [ ] **1. OWASP**: Могу объяснить каждый из 10 пунктов OWASP Top 10 for LLM 2025 и привести пример mitigations.
- [ ] **2. ATLAS**: Знаю структуру MITRE ATLAS, могу сопоставить техники с уязвимостями OWASP.
- [ ] **3. NIST AI RMF**: Понимаю 4 функции (Govern, Map, Measure, Manage) и могу применить к кейсу.
- [ ] **4. Zero Trust**: Могу объяснить, как каждый из 5 принципов ZT применяется к AI/LLM.
- [ ] **5. Defense-in-Depth**: Могу нарисовать 5+ слоёв защиты для LLM-приложения и объяснить компенсирующие контроли.
- [ ] **6. Red Teaming**: Знаю методологию AI Red Teaming, могу назвать 3+ инструмента и 3+ техники.
- [ ] **7. Инциденты**: Могу разобрать 3+ реальных инцидента (Samsung, Clearview, Bing) и вывести уроки.
- [ ] **8. Метрики**: Могу назвать 5+ метрик AI Security и объяснить, как их измерять.
- [ ] **9. Compliance**: Понимаю требования EU AI Act для high-risk AI и отличия от ISO 42001.
- [ ] **10. Архитектура**: Могу предложить architecture review для AI-системы с RAG, агентами и внешними API — и защитить её.

---

## Ключевые выводы

1. **AI Security — это не подмножество классической безопасности**. Это новая дисциплина, пересекающая ML, AppSec, Data Privacy, Compliance и OrgSec. Архитектор, игнорирующий любой из слоёв, оставляет брешь.

2. **Нет серебряной пули**. Guardrails, moderation API, input filters — всё это важно, но не заменяет defense-in-depth. LLM — это по своей природе недетерминированный компонент, и безопасность строится на **компенсирующих контролях** вокруг него.

3. **Фреймворки работают вместе**. OWASP Top 10 for LLM говорит разработчикам, что защищать. MITRE ATLAS говорит SOC, как атакуют. NIST AI RMF говорит CISO, как управлять рисками. ISO 42001 говорит compliance, как организовать процессы. EU AI Act говорит юристам, что обязательно. **Мастерство — в синтезе**.

4. **AI-угрозы эволюционируют быстрее защиты**. Jailbreaks 2022 года выглядят наивными сегодня. Атаки 2025 года на мульти-агентные системы станут обыденностью к 2027. Непрерывный Red Teaming, regression testing и community intelligence — единственный способ держать шаг.

5. **Человек остаётся ключевым контролем**. Human-in-the-loop, human oversight, human review — эти «медленные» элементы часто спасают быстрее, чем самые быстрые автоматизированные guardrails. AI Security — это не только технология, но и **культура внимательности**.

6. **Метрики превращают faith-based security в evidence-based**. Если вы не можете измерить detection rate prompt injection, MTTD AI-инцидента или стоимость предотвращённой утечки — вы управляете на ощупь.

7. **BYO-AI и Shadow AI — угрозы №1, которые игнорируют**. Технически совершенная защита production-системы бессмысленна, если сотрудник копирует конфиденциальные данные в ChatGPT. DLP + education + corporate AI gateway — must-have.

8. **Supply chain AI уязвима так же, как software supply chain**. Модели, datasets, библиотеки, benchmarks — всё это может быть скомпрометировано до того, как вы начнёте защищать своё приложение. Model signing, provenance, vendor assessment — необходимы.

9. **AI Red Teaming должно быть непрерывным, а не разовым**. Каждое обновление модели, каждый новый guardrail, каждый новый инструмент агента — потенциальная регрессия. Автоматизация adversarial testing в CI/CD — цель.

10. **Этот overview — отправная точка, не конечная станция**. Реальная экспертиза формируется через практику: Red Teaming, инциденты, архитектурные решения, взаимодействие с разработчиками. Используйте этот чек-лист как карту, но помните: **город постоянно растёт и меняется**.

### Ответы на чек-лист

1. **AI Security — это не подмножество классической безопасности, а пересечение пяти дисциплин.** Классическая AppSec фокусируется на коде и инфраструктуре, но AI добавляет prompt injection, data poisoning, model extraction, adversarial robustness. LLM-системы недетерминированы: один prompt может дать разные ответы. Традиционные тесты (SAST, DAST) недостаточны — нужен continuous adversarial testing. NIST AI RMF и OWASP Top 10 for LLM признают этот unique threat landscape.

2. **Пирамида безопасности ИИ: data, model, deployment, application, governance.** Data layer: data provenance, differential privacy, PII detection. Model layer: model signing, watermarking, access control. Deployment layer: sandboxing, rate limiting, API security. Application layer: input/output validation, guardrails, human-in-the-loop. Governance layer: Model Cards, AI RMF, EU AI Act compliance. Пропуск слоя — single point of failure.

3. **AI Kill Chain (MITRE ATLAS): 7 стадий.** Reconnaissance → Resource Development → Initial Access → ML Model Access → Execution → Persistence → Impact. Всего ~30 техник, позволяющих SOC анализировать AI-инциденты структурированно.

4. **OWASP говорит ЧТО защищать, ATLAS говорит КАК ломают.** OWASP LLM01 (Prompt Injection) → ATLAS Execution и Initial Access. OWASP LLM04 (DoS) → ATLAS Impact и Execution. OWASP LLM06 (Disclosure) → ATLAS Impact и Persistence. Security Architect должен map контроли OWASP на техники ATLAS.

5. **Zero Trust для AI: verify every query, output, data source, assume compromise.** Input validation, output filtering, data source authentication, compensating controls (sandboxing, DLP, audit trails). AI Zero Trust добавляет верификацию на уровне данных, модели и поведения.

6. **Red Teaming: automated adversarial testing + manual jailbreaks + multi-agent simulation + continuous regression.** Инструменты: Garak, PurpleLlama, NVIDIA Garak, ART. Метрики: jailbreak success rate, time-to-compromise, OWASP LLM coverage.

7. **Ключевые инциденты:** Samsung (утечка через ChatGPT training data) → corporate AI gateway + DLP. Clearview AI (скрапинг без согласия, GDPR £17.2M) → data minimization + consent. Bing Chat Sydney (jailbreak через emotional manipulation) → defense-in-depth: input + output + behavioral monitoring + human oversight.

8. **Ключевые метрики:** Detection Rate, False Positive Rate, MTTD, MTTR, Model Drift Score, Fairness Metrics (demographic parity, equalized odds), Explainability Coverage, Compliance Score. Без метрик AI Security — faith-based, не evidence-based.

9. **EU AI Act (регуляция, штрафы до 35M€) vs ISO 42001 (voluntary standard).** EU AI Act: risk-based, 4 уровня риска, CE marking для high-risk. ISO 42001: management system (Govern, Map, Measure, Manage), не предписывает технических требований. Компании используют ISO 42001 как foundation для EU AI Act compliance.

10. **Architecture review для AI с RAG + агенты + внешние API:** Defense-in-Depth на 5 слоях. Data: sandboxed retrieval, document sanitization, vector DB RBAC. Model: model signing, output filtering, rate limiting. Deployment: API gateway, AI DMZ, container hardening. Application: input/output validation, guardrails, human-in-the-loop, agent isolation. Governance: Model Cards, audit trails, drift/bias/anomaly monitoring. External API: zero-trust integration, schema validation, circuit breakers. RAG: permission-aware retrieval, semantic filtering. Угрозы: multi-hop prompt injection, tool poisoning, context window exhaustion.

---

*Статья №10 серии «AI/LLM Security». Ссылки: [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [MITRE ATLAS](https://atlas.mitre.org/), [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework), [ISO/IEC 42001:2023](https://www.iso.org/standard/81230.html), [EU AI Act](https://artificial-intelligence-act.com/).*
