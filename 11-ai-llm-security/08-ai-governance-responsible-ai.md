# Управление ИИ и Responsible AI: Governance-фреймворки, EU AI Act, ISO 42001, модельные карточки, MLOps Security и архитектура контроля для генеративных моделей

## Что такое AI Governance?

Представьте огромную атомную электростанцию, построенную в центре города. Городской совет не может просто доверить её операторам — нужны обязательные проверки, сертификация персонала, аварийные протоколы и постоянный мониторинг. Если что-то пойдёт не так, ответственность лежит не только на операторах, но и на тех, кто утвердил запуск станции.

**AI Governance** — это совокупность процессов, политик, стандартов и архитектурных решений, обеспечивающих **контролируемое, прозрачное и подотчётное** использование искусственного интелекта. **Responsible AI** расширяет эту концепцию этическими принципами: справедливость, прозрачность, подотчётность, надёжность и уважение к приватности. Принцип прост: технологическая мощь требует соответствующего уровня governance — без него AI превращается из инструмента в угрозу.

## Эволюция и мотивация

История AI Governance прошла три стадии:

**Ad-hoc ML (2010–2018):** Data Scientists обучали модели на локальных ноутбуках, деплоили вручную, мониторили через логи. Governance отсутствовала — модели были простыми, а масштаб ограниченным.

**MLOps эра (2018–2022):** Переход к CI/CD для ML, Model Registry, automated retraining. Фокус на скорости и reproducibility, но безопасность оставалась вторичной. Появились первые инциденты bias и утечек данных.

**AI Governance эра (2022–2025):** Запуск ChatGPT и корпоративное внедрение LLM изменили правила игры. Регуляторы отреагировали:
- **EU AI Act (2024):** Первый комплексный закон о регулировании AI, risk-based подход
- **NYC Local Law 144 (2023):** Обязательный аудит hiring algorithms
- **ISO 42001 (2023):** Первый международный стандарт management systems для AI
- **NIST AI RMF (2023):** Фреймворк управления рисками AI для организаций
- **Китай:** Строгие правила для алгоритмic рекомендаций и deepfakes

Почему governance критичен сейчас: модели принимают решения, влияющие на жизни людей (hiring, lending, healthcare, criminal justice), а их поведение непредсказуемо и труднообъяснимо.

## Зачем это нужно?

**High-risk AI в медицине.** AI-система диагностики рака должна пройти клинические испытания, получить сертификацию FDA/EMA, иметь explainable decision-making и human-in-the-loop. Без governance — юридическая ответственность за ошибочный диагноз не определена.

**Hiring и HR.** Алгоритмы скрининга резюме могут воспроизводить исторические bias (например, против женщин в tech). NYC Local Law 144 требует ежегодного аудита bias. Governance обеспечивает fairness metrics и appeal process.

**Кредитный скоринг.** AI-модели оценки кредитоспособности влияют на экономические возможности. EU AI Act классифицирует это как high-risk. Требуется: explainability, право на объяснение, human oversight.

**Правоохранительные органы.** Predictive policing и facial recognition требуют строгого governance: accuracy thresholds, demographic parity, audit trails. COMPAS показал, что без governance — дискриминация в масштабе.

**Автономные системы.** Беспилотные автомобили, дроны, роботы-хирурги. Без governance framework — невозможно определить ответственность при аварии.

## Основные концепции

### AI Governance Framework

**AI Governance Framework** — это структурированный набор политик, процессов и ролей для управления жизненным циклом AI-систем. Два ключевых фреймворка:

**NIST AI Risk Management Framework (AI RMF):** Четыре функции:
- **Govern:** Установление политик, ролей, accountability
- **Map:** Идентификация контекста использования AI, stakeholders, рисков
- **Measure:** Количественная и качественная оценка рисков, metrics, testing
- **Manage:** Реагирование на риски, monitoring, continuous improvement

**ISO 42001:** Management system для AI, совместимый с ISO 27001. Фокус на:
- Политиках и objectives для AI
- Risk assessment и treatment
- Internal audit и management review
- Continuous improvement

### Model Cards и Transparency

**Model Card** (популяризировано Google, 2019) — стандартизированный документ, описывающий:
- Model purpose, intended use, limitations
- Training data sources, preprocessing, demographics
- Performance metrics (overall и per-subgroup)
- Ethical considerations, known biases, caveats
- Environmental impact (compute, carbon footprint)

Model Cards делают модель **прозрачной и auditable**. Это не просто документация — это контракт между разработчиком и пользователем.

### Model Risk Management (MRM)

**MRM** — адаптация традиционного управления рисками (ORM) для AI:
- **Model Inventory:** Реестр всех моделей с metadata (owner, purpose, risk level)
- **Validation:** Независимая валидация model performance, robustness, fairness
- **Monitoring:** Continuous monitoring model drift, performance degradation, bias emergence
- **Remediation:** Процедуры decommissioning, rollback, retraining
- **Documentation:** Audit trails для всех решений

MRM требует **трех линий обороны**: business owners (1st line), risk managers (2nd line), internal audit (3rd line).

### MLOps Security

**MLOps Security** — интеграция security в CI/CD для ML:
- **Secure training pipelines:** Валидация данных, sandboxed execution, secret management
- **Model registry security:** Access control, versioning, provenance tracking
- **Deployment security:** Container hardening, network segmentation, API security
- **Monitoring:** Drift detection, anomaly detection, adversarial input monitoring

MLOps Security закрывает gap между традиционным DevSecOps и спецификой ML (data poisoning, model extraction, adversarial examples).

### Auditability и Explainability (XAI)

**Explainability** — способность объяснить решение модели. Три уровня:
- **Global explainability:** Как модель принимает решения в целом (SHAP, LIME, feature importance)
- **Local explainability:** Почему конкретное решение (counterfactual explanations)
- **Model-agnostic vs model-specific:** SHAP работает для любой модели, attention weights — только для transformers

**Auditability** — способность воспроизвести и проверить решение:
- Version control для данных, кода, моделей
- Immutable audit trails
- Reproducible experiments
- Decision logging

### EU AI Act

**EU AI Act (2024)** — первый в мире комплексный закон о регулировании AI. Risk-based подход:

| Risk Level | Описание | Примеры | Требования |
|------------|----------|---------|------------|
| **Unacceptable** | Запрещено | Social scoring, real-time biometric ID в публичных местах, subliminal manipulation | Полный запрет |
| **High-risk** | Строгий контроль | Medical devices, hiring, credit scoring, education, law enforcement | CE marking, risk management, data governance, transparency, human oversight, accuracy, robustness, cybersecurity |
| **Limited risk** | Прозрачность | Chatbots, emotion recognition | Disclosure obligations |
| **Minimal risk** | Рекомендации | AI-enabled video games, spam filters | Voluntary codes |

Штрафы: до 35 млн € или 7% global turnover.

### Model Registry и Versioning

**Model Registry** — централизованное хранилище моделей с:
- Version control (data + code + model artifacts)
- Metadata (hyperparameters, metrics, lineage)
- Stage management (development → staging → production)
- Access control и approval workflows

Versioning для AI сложнее, чем для кода: нужно отслеживать **data version, code version, model weights version** как единое целое.

### Human-in-the-loop

**Human-in-the-loop (HITL)** — обязательное человеческое участие в критических решениях:
- **Human-over-the-loop:** Человек может отменить решение AI
- **Human-in-the-loop:** Человек участвует в процессе принятия решения
- **Human-on-the-loop:** Человек мониторит и может вмешаться

EU AI Act требует HITL для high-risk систем. Это не просто "кнопка утверждения" — это meaningful human control с пониманием контекста и последствий.

## Сравнение подходов к governance

| Критерий | Manual Review | Automated Governance | Hybrid (рекомендуется) |
|----------|---------------|----------------------|------------------------|
| **Скорость** | Низкая (bottleneck) | Высокая | Средняя (automated + spot checks) |
| **Масштабируемость** | Плохая | Отличная | Хорошая |
| **Гибкость** | Высокая (human judgment) | Низкая (rigid rules) | Высокая |
| **Стоимость** | Высокая (персонал) | Средняя (инструменты) | Средняя |
| **Покрытие** | Частичное (sample-based) | Полное | Полное + targeted deep dives |
| **Explainability** | Естественная | Требует tooling | Комбинированная |
| **Compliance** | Трудно доказать | Легко доказать | Легко доказать |

**Практический вывод:** Pure automated governance не работает для edge cases и новых угроз. Pure manual не масштабируется. Hybrid — automated monitoring + human review для exceptions и high-risk decisions.

## Уроки из реальных инцидентов

### COMPAS (2016)
Алгоритм оценки рецидива в Wisconsin показал **racial bias**: ложноположительная rate для Black defendants была почти вдвое выше, чем для White. Корневая причина: обучающие данные отражали историческую дискриминацию в судебной системе. Урок: **bias в данных воспроизводит bias в реальности**. Требуется fairness auditing до deployment.

### Amazon Rekognition (2018)
Система facial recognition ошибочно идентифицировала членов Конгресса как преступников. Корневая причина: низкая accuracy для non-White лиц и женщин. Amazon предоставила Rekognition правоохранительным органам без adequate governance. Урок: **accuracy disparities across demographics — критичный риск**, особенно для high-stakes applications.

### Apple Card (2019)
Алгоритм кредитного лимита Apple Card давал значительно более низкие лимиты женщинам с аналогичным financial profile. Алгоритм — black box, Apple не могла объяснить решения. Урок: **lack of explainability → inability to detect bias → reputational and legal damage**. Transparency requirements (например, право на объяснение по GDPR) становятся обязательными.

### Google Duplex (2018)
AI-ассистент, имитирующий человеческий голос для звонков, вызвал этический скандал: люди не знали, что разговаривают с машиной. Google добавила disclosure, но инцидент показал: **disclosure obligations — минимальный bar для responsible AI**. EU AI Act теперь требует disclosure для chatbots.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| **Model Cards** | Прозрачность модели | Post-training, pre-deployment | ISO 42001 (документация), EU AI Act (transparency) |
| **Bias Detection (Fairlearn, Aequitas)** | Обнаружение demographic bias | Training, validation | NIST AI RMF (Measure), EU AI Act (fairness) |
| **Explainability (SHAP, LIME, Captum)** | Объяснение решений модели | Validation, runtime | EU AI Act (explainability), NIST AI RMF |
| **Model Registry (MLflow, Weights & Biases)** | Version control, lineage, approval | Entire lifecycle | ISO 42001 (management system) |
| **Drift Detection (Evidently, Fiddler)** | Мониторинг data/model drift | Production | NIST AI RMF (Monitor), MRM |
| **Policy as Code (Open Policy Agent)** | Automated governance enforcement | Deployment | ISO 42001 (controls) |
| **Audit Trails (Databricks Unity Catalog)** | Immutable logging | Entire lifecycle | EU AI Act (accountability), ISO 42001 |
| **Human-in-the-loop Platforms (Amazon A2I)** | Human review interfaces | Runtime, exceptions | EU AI Act (human oversight) |

## Архитектурные решения

### Policy as Code для AI
Политики governance (например, "high-risk модели требуют fairness audit") кодируются в виде executable rules. **Open Policy Agent (OPA)** или **OPA for ML** автоматически проверяют compliance перед deployment. Преимущество: policies versioned, tested, enforced automatically.

### Model Registry как Source of Truth
Централизованный Model Registry с:
- Immutable artifact storage
- Role-based access control (data scientists, reviewers, auditors)
- Approval workflows (automated checks → human approval → deployment)
- Integration с CI/CD pipelines

### Drift Monitoring Pipeline
Continuous monitoring:
- **Data drift:** Распределение входных данных изменилось (concept drift)
- **Model drift:** Performance degradation over time
- **Bias drift:** Fairness metrics изменились для подгрупп
- **Anomaly detection:** Adversarial inputs, out-of-distribution samples

### Audit Trails и Lineage
**End-to-end lineage:** от raw data → cleaned data → training run → model version → deployment → prediction log. **Immutable audit trails** в tamper-proof storage (blockchain или WORM — Write Once Read Many).

### Human-in-the-loop Integration
- **Exception-based review:** Automated governance flags edge cases для human review
- **Random sampling:** X% decisions reviewed for quality assurance
- **Escalation paths:** Automated alerts → team lead → ethics committee

## Подготовка к собеседованию: Common pitfalls & key questions

### Типичные ошибки кандидатов

**Ошибка 1: Путать AI Governance с традиционным IT Governance.**
IT Governance фокусируется на availability, confidentiality, integrity систем. AI Governance добавляет: fairness, explainability, robustness, accountability за decisions. Кандидат, не выделяющий эти дополнительные измерения, показывает поверхностное понимание.

**Ошибка 2: Считать, что explainability = interpretability.**
Interpretability — intrinsic property модели (например, decision tree). Explainability — post-hoc объяснение любой модели (SHAP, LIME). Deep learning модели не interpretable, но могут быть explainable. Путаница показывает непонимание fundamental difference.

**Ошибка 3: Думать, что EU AI Act касается только EU companies.**
EU AI Act применяется к любому AI-системе, используемой в EU, независимо от места разработки. American компания, деплоящая hiring AI в EU, обязана comply. Кандидат, не понимающий extraterritorial applicability, демонстрирует пробел в compliance knowledge.

### Ключевые вопросы интервьюера

**Вопрос 1:** "Как бы вы построили AI Governance программу с нуля для финансовой организации с 50+ ML-моделями?"

*Ожидаемый ответ:* Начал бы с inventory всех моделей (risk-based classification). High-risk модели (credit scoring, fraud detection) — priority для governance. Внедрил бы Model Registry с approval workflows. Установил бы continuous monitoring drift и bias. Обеспечил бы explainability для customer-facing decisions (GDPR/CCPA right to explanation). Регулярный audit (quarterly) с independent validation team. Integration с existing risk management framework (ORM, BCM).

**Вопрос 2:** "В чём разница между bias mitigation pre-processing, in-processing и post-processing? Приведите примеры."

*Ожидаемый ответ:* **Pre-processing** — изменение данных до обучения (resampling, reweighting, synthetic data generation). **In-processing** — изменение алгоритма обучения (fairness constraints в loss function, adversarial debiasing). **Post-processing** — корректировка predictions после обучения (threshold optimization per group, calibrated equalized odds). Pre-processing работает, когда проблема в данных. In-processing — когда нужно изменить сам learning process. Post-processing — когда модель уже обучена и нельзя переобучить.

**Вопрос 3:** "Какие метрики fairness вы бы использовали для hiring AI, и почему?"

*Ожидаемый ответ:* **Demographic parity** (одинаковая rate positive decisions across groups) — важна, но может mask qualified candidates. **Equalized odds** (одинаковая TPR и FPR across groups) — более сильная fairness guarantee, но сложнее достичь. **Predictive parity** (одинаковая precision across groups). Для hiring предпочёл бы **equalized odds** с monitoring **disparate impact ratio** (80% rule — EEOC). Но fairness — не одна метрика, а trade-off: optimizing for fairness может снизить overall accuracy. Нужен business alignment на acceptable trade-off.

## Чек-лист понимания

- [ ] Почему AI Governance нельзя свести к традиционному IT Governance, и какие три дополнительных измерения оно добавляет?
- [ ] Как EU AI Act классифицирует AI-системы по уровням риска, и какие требования предъявляются к high-risk системам?
- [ ] В чём разница между interpretability и explainability, и почему deep learning модели требуют post-hoc объяснений?
- [ ] Какие три линии обороны используются в Model Risk Management, и какова роль каждой?
- [ ] Почему hybrid governance (automated + manual) предпочтительнее pure automated или pure manual approaches?
- [ ] Как historical bias в обучающих данных привёл к дискриминации в COMPAS, и какие методы могли бы это предотвратить?
- [ ] В чём разница между human-in-the-loop, human-over-the-loop и human-on-the-loop, и когда применяется каждый?
- [ ] Как Model Cards повышают transparency и какие восемь элементов они должны содержать?
- [ ] Почему end-to-end lineage (data → model → prediction) критична для auditability AI-систем?
- [ ] Как extraterritorial applicability EU AI Act влияет на non-EU компании, использующие AI в Европе?

### Ответы на чек-лист

1. **AI Governance добавляет fairness, explainability и accountability за decisions.** IT Governance фокусируется на CIA triad (confidentiality, integrity, availability) для систем. AI Governance расширяет это: fairness — отсутствие дискриминации по защищённым признакам; explainability — способность объяснить конкретное решение; accountability — определение ответственных лиц за AI-driven decisions. Эти измерения критичны, потому что AI принимает автономные решения, влияющие на права и свободы людей. NIST AI RMF явно выделяет trustworthy AI characteristics: valid and reliable, safe, secure and resilient, accountable and transparent, explainable and interpretable, privacy-enhanced, fair with harmful bias managed.

2. **EU AI Act использует четырёхуровневую классификацию рисков.** Unacceptable risk (запрещено): social scoring государством, real-time biometric identification в публичных местах, subliminal techniques causing harm. High-risk (строгий контроль): AI в критических инфраструктурах, образовании, труде, правоохране, миграции, justice. Требования: risk management system, data governance, technical documentation, record-keeping, transparency, human oversight, accuracy, robustness, cybersecurity. Limited risk (прозрачность): chatbots, emotion recognition — disclosure obligations. Minimal risk (волонтёрские коды): spam filters, AI в играх. High-risk системы требуют CE marking, conformity assessment, и регистрации в EU database до deployment. Штрафы до 35 млн € или 7% global turnover.

3. **Interpretability — intrinsic property модели, explainability — post-hoc анализ.** Interpretable модели (linear regression, decision trees) позволяют прямо прочитать decision logic. Deep learning модели (transformers, CNN) — black boxes: мы не можем прочитать weights и понять логику. Explainability (SHAP, LIME, attention visualization) создаёт approximation explanation после принятия решения. Это не тождество: explainable модель может давать misleading explanations, если explanation method не подходит для архитектуры. SHAP работает model-agnostically, attention weights — только для transformer-based models и дают limited insight.

4. **Три линии обороны в MRM: Business Owners (1st line), Risk Managers (2nd line), Internal Audit (3rd line).** 1st line — data scientists и ML engineers, ответственные за development, validation, deployment. Они реализуют controls и monitoring. 2nd line — независимые risk managers, проводящие review, setting risk appetite, validating models. Они не разрабатывают модели, но утверждают их для production. 3rd line — internal audit, проверяющая эффективность governance framework целиком. Эта структура обеспечивает checks and balances: разработчики не проверяют сами себя, risk managers не разрабатывают, audit независим от обеих линий.

5. **Hybrid governance сочетает масштабируемость automated с judgment human reviewers.** Pure automated governance обрабатывает 100% decisions, но не справляется с edge cases, novel attacks, ethical dilemmas. Pure manual не масштабируется на тысячи decisions в час и создаёт bottlenecks. Hybrid: automated checks покрывают routine decisions (90%+), flagged exceptions направляются human reviewers. Human reviewers также проводят random sampling для quality assurance. Это обеспечивает полное покрытие без потери гибкости. Пример: credit scoring AI автоматически одобряет/отклоняет 95% заявок, edge cases (borderline scores, atypical profiles) — human loan officer.

6. **COMPAS bias возник из historical bias в обучающих данных.** Судебные записи отражали системную дискриминацию: Black defendants получали более высокие sentences за аналогичные преступления. Алгоритм обучался на этих данных и воспроизводил bias, предсказывая higher recidivism risk для Black defendants. Методы предотвращения: pre-processing (reweighting training examples, removing protected attributes), in-processing (fairness constraints в loss function — например, equalized odds), post-processing (threshold optimization per group), diverse training data (oversampling underrepresented groups), human review (mandatory review для flagged cases). Но ключевой урок: **fairness metrics должны измеряться не только overall, но и per-subgroup**. Aggregate accuracy маскирует disparate impact.

7. **Human-in-the-loop (HITL): человек участвует в процессе принятия решения.** Human-over-the-loop: человек может отменить AI decision, но не участвует в процессе (supervisory control). Human-on-the-loop: человек мониторит процесс и может вмешаться при anomalies (monitoring). EU AI Act требует meaningful human oversight для high-risk систем — это не просто "кнопка утверждения", а understanding context, implications, ability to override. HITL применяется, когда решение критично и не может быть полностью автоматизировано (medical diagnosis, criminal sentencing). Human-over-the-loop — когда AI делает routine decisions, но человек может вмешаться (fraud detection alerts). Human-on-the-loop — continuous monitoring (autonomous vehicles safety driver).

8. **Model Cards содержат восемь ключевых элементов:** (1) Model details (owner, version, type, architecture); (2) Intended use (primary purpose, users, out-of-scope uses); (3) Factors (relevant demographic groups, environmental conditions); (4) Metrics (performance metrics, decision thresholds); (5) Evaluation data (datasets, preprocessing, splits); (6) Training data (sources, size, demographics, known limitations); (7) Ethical considerations (bias risks, fairness analysis, mitigation); (8) Caveats and recommendations (known failure modes, appropriate use cases, maintenance requirements). Model Cards повышают transparency, enabling informed decision-making, facilitating audit, supporting regulatory compliance. Они — не просто документация, а accountability mechanism: публично доступные Model Cards позволяют third-party scrutiny.

9. **End-to-end lineage критична для auditability, потому что AI decisions — это emergent property всего pipeline.** Ответственность за ошибочное решение может лежать на: data collectors (biased data), data scientists (problematic architecture), DevOps (deployment configuration), operators (drift monitoring failure). Без lineage невозможно определить, кто и что привело к ошибке. Lineage включает: data provenance (источники, преобразования, версии), code version (training script, preprocessing, inference), model artifacts (weights, hyperparameters, metrics), deployment configuration (API endpoints, thresholds, fallback logic), prediction logs (inputs, outputs, confidence scores, explanations). Immutable lineage обеспечивает: root cause analysis, regulatory compliance, reproducibility, continuous improvement.

10. **EU AI Act имеет extraterritorial applicability:** применяется к любому AI-системе, используемой на территории EU, независимо от места разработки или места компании. American SaaS-платформа, предоставляющая hiring AI европейским клиентам, обязана comply. Chinese производитель autonomous vehicles, продаваемых в EU — обязан. Это создаёт global compliance pressure: компании предпочитают строить unified governance framework, соответствующий highest common denominator (EU AI Act), вместо maintaining separate regional systems. Для non-EU компаний это означает: (1) понимание risk classification своих AI-систем в EU context; (2) внедрение CE marking process для high-risk систем; (3) назначение authorized representative в EU; (4) integration EU requirements в global governance framework. Игнорирование → блокировка access к EU market или штрафы до 35 млн €.

## Ключевые выводы для собеседования

- **AI Governance = IT Governance + fairness + explainability + accountability.** Без этих трёх дополнительных измерений governance неполноценно.
- **EU AI Act — первый comprehensive AI regulation, использующий risk-based approach.** Понимание классификации рисков и требований к high-risk системам обязательно для Security Architect.
- **Hybrid governance (automated + manual) — единственный масштабируемый и гибкий подход.** Pure automated не справляется с edge cases, pure manual — не масштабируется.
- **End-to-end lineage и Model Cards — foundation для auditability и transparency.** Без них невозможно compliance и continuous improvement.
- **Fairness — не одна метрика, а многомерный trade-off.** Demographic parity, equalized odds, predictive parity — разные fairness definitions могут конфликтовать. Выбор зависит от business context и regulatory requirements.

---
_Статья создана на основе анализа материалов Habr, официальной документации, отчётов CERT/ENISA и международных стандартов безопасности (NIST, CIS, OWASP, ISO)._