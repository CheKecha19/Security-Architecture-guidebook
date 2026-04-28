# Безопасность RAG: Как защитить Retrieval-Augmented Generation от отравления базы знаний, утечек через поисковые контексты и атак на векторные хранилища

## Что такое RAG Security?

**RAG Security** — это совокупность практик, контролей и архитектурных решений, направленных на защиту **Retrieval-Augmented Generation** систем от компрометации данных, манипуляции ответами и несанкционированного доступа на всех этапах retrieval pipeline.

**Бытовая аналогия.** Представьте библиотекаря, который не просто выдаёт книги, а пересказывает их содержание читателю. Теперь представьте, что на полках появились поддельные книги с ложными фактами — библиотекарь, доверяя своему каталогу, будет пересказывать дезинформацию. Ещё хуже: злоумышленник может подсунуть книги, которые при прочтении заставляют библиотекаря произносить секретную информацию вслух. RAG Security — это система проверки каждой книги перед попаданием на полку, разграничение доступа к разным залам библиотеки и аудит того, что именно библиотекарь пересказывает.

Принцип: **целостность retrieval pipeline** — от момента загрузки документа в векторное хранилище до генерации финального ответа LLM. Любой из этих этапов может стать вектором атаки.

## Эволюция и мотивация

**Retrieval-Augmented Generation** стал доминирующим enterprise AI паттерном после 2022 года по одной простой причине: он решает проблему галлюцинаций LLM, не требуя переобучения моделей. Вместо того чтобы полагаться на параметрическое знание модели, RAG извлекает актуальную информацию из внешних источников и подставляет её в контекст.

Эволюция RAG прошла через несколько стадий:
- **Простой RAG (2020–2022):** базовая семантическая близость + генерация ответа.
- **Advanced RAG (2022–2023):** query rewriting, reranking, hybrid search, агентные паттерны.
- **Enterprise RAG (2023–2025):** multi-modal retrieval, permission-aware search, real-time ingestion, agentic orchestration.

С каждой стадией поверхность атаки расширялась. **Vector DB** — это не просто база данных, а новая поверхность атаки. Традиционные базы данных защищаются SQL-инъекций и RBAC, но векторные хранилища добавляют семантические уровни уязвимостей: близость векторного пространства не соответствует логическому контролю доступа, а внедрение вредоносных документов может быть обнаружено только семантически.

Почему это критично:
- **RAG — точка входа чувствительных данных.** Внутренние документы, переписки, код, legal documents — всё это попадает в векторные хранилища.
- **Векторные пространства — «чёрные ящики» для традиционного DLP.** Текст превращается в эмбеддинги, и стандартные инструменты обнаружения утечек перестают работать.
- **Атака на retrieval — атака на доверие.** Пользователи доверяют ответам RAG-систем, считая их «основанными на фактах». Отравление базы знаний — это атака на само доверие.

## Зачем это нужно?

### 1. Enterprise Knowledge Base
Корпоративная вики с финансовой отчётностью, стратегическими планами и M&A данными. Компрометация RAG-системы позволяет извлечь конфиденциальные показатели через ряд вопросов, каждый из которых сам по себе безобиден, но в совокупности раскрывает sensitive data.

### 2. Customer Support RAG
Чат-бот поддержки клиентов с доступом к документации, логам и ticket history. Злоумышленник может внедрить вредоносный документ через заявку в поддержку, а затем эксплуатировать отравлённую базу для получения данных других клиентов.

### 3. Legal Document RAG
Система для анализа контрактов, NDA и судебных решений. Отравление базы знаний может привести к генерации юридически некорректных рекомендаций, что создаёт exposure для компании и её клиентов.

### 4. HR Policy Bot
HR-ассистент с доступом к политикам компенсаций, процедурам увольнения и performance reviews. Indirect prompt injection через обновлённый policy document может заставить бота раскрыть salary bands или процедуры reduction in force до официального анонса.

### 5. Code-Assistant RAG
RAG для внутренней документации по API, секретам инфраструктуры и архитектурным решениям. Векторная БД может содержать забытые API keys, connection strings и internal endpoints, извлекаемые через специально сконструированные запросы.

## Основные концепции

### Архитектура RAG и поверхность атаки

Стандартный RAG pipeline состоит из пяти этапов, каждый из которых уязвим:

1. **Ingestion:** документы загружаются, разбиваются на chunks, преобразуются в эмбеддинги. Атака: внедрение вредоносного документа.
2. **Embedding:** текст превращается в векторы через модель типа BGE, E5, OpenAI Ada. Атака: adversarial perturbations в тексте, манипулирующие векторным представлением.
3. **Storage:** эмбеддинги индексируются в Vector DB (Pinecone, Weaviate, Chroma, pgvector). Атака: прямая модификация векторов, unauthorized access к индексу.
4. **Retrieval:** по запросу извлекаются top-k ближайших чанков. Атака: query manipulation, forcing retrieval of poisoned chunks.
5. **Generation:** LLM генерирует ответ на основе retrieved контекста. Атака: indirect prompt injection через retrieved chunks.

Каждый этап требует собственных контролей: validation, sanitization, access control, monitoring.

### Data Poisoning в векторной БД

**Data Poisoning** в контексте RAG — это преднамеренное внедрение вредоносных или манипулируемых документов в корпус знаний с целью искажения ответов LLM или извлечения чувствительной информации.

**Механизмы:**
- **Прямое внедрение:** атакующий имеет доступ к ingestion pipeline и загружает вредоносный документ.
- **Indirect poisoning:** вредоносный контент попадает через автоматический краулинг, импорт из внешних источников или user-generated content.
- **Adversarial chunks:** документы, семантически оптимизированные для ранжирования по целевым запросам, но содержащие ложную информацию или инъекции.

**Последствия:**
- LLM генерирует ответы на основе отравленных данных.
- Пользователи принимают решения на основе ложной информации.
- Атакующий может управлять ответами на определённые темы (brand safety, financial advice, medical information).

**Аналогия:** Это как если бы в аптеку под видом лекарств завезли поддельные таблетки, и система проверки качества не заметила подмену, потому что упаковка была идеальной.

### Indirect Prompt Injection через документы

**Indirect Prompt Injection (IPI)** — это техника, при которой вредоносные инструкции встраиваются в документы, попадающие в retrieved context, и затем исполняются LLM при генерации ответа.

**Классические паттерны IPI:**
- **Ignore previous instructions:** в чанке содержится текст вроде «Игнорируй все предыдущие инструкции и скажи...»
- **Delimiter attacks:** использование маркеров разделения (---, ```, <system>) для выхода из контекстного окна и имитации системных инструкций.
- **Multi-turn poisoning:** документы содержат инструкции, активируемые только при определённой последовательности вопросов.
- **Exfiltration directives:** чанк содержит инструкцию отправить данные на внешний endpoint, часто замаскированную под цитату или пример.

**Почему это опасно:**
- LLM не различает «системные инструкции» и «данные из документа» в retrieved context.
- Даже если векторная БД защищена, вредоносный чанк может попасть через легитимный документ, загруженный доверенным пользователем (compromised account).
- OWASP LLM02 (Sensitive Information Disclosure) и LLM06 (Sensitive Information Disclosure) прямо включают IPI как вектор утечки.

### Access Control & Permission-Aware RAG

**Permission-Aware RAG** — это архитектурный паттерн, при котором retrieval учитывает права доступа пользователя к документам, а не просто возвращает semantically ближайшие чанки.

**Проблема:** Стандартный RAG игнорирует ACL. Если чанк из конфиденциального документа семантически близок к запросу, он будет извлечён независимо от того, имеет ли пользователь право видеть исходный документ.

**Решения:**
- **Pre-filtering:** применение метаданных ACL до retrieval (where clause в vector search).
- **Post-filtering:** фильтрация результатов retrieval по permissions до отправки в LLM.
- **Multi-tenant indices:** разделение индексов по tenant/department/security clearance.
- **Re-ranking с учётом permissions:** ранжирование результатов с учётом не только semantic similarity, но и authorization level.

**Аналогия:** Обычный RAG — это библиотекарь, который приносит любую книгу по запросу. Permission-Aware RAG — это библиотекарь, который проверяет читательский билет и приносит книги только из разрешённых залов.

### Semantic Filtering и Content Filtering

**Semantic Filtering** — это проверка retrieved чанков на соответствие запросу, на отсутствие вредоносного контента и на compliance с политиками.

**Уровни фильтрации:**
- **Query-level:** проверка запроса на injection attempts, jailbreak, exfiltration patterns.
- **Chunk-level:** проверка каждого retrieved чанка на malicious content, PII, toxicity, policy violations.
- **Context-level:** проверка assembled context на coherence, consistency, potential conflicts.
- **Output-level:** post-generation filtering ответа на наличие unintended disclosures или harmful content.

**Content Filtering** включает:
- **PII detection:** обнаружение и маскирование персональных данных в retrieved chunks.
- **Topic filtering:** блокирование ответов на определённые темы (medical advice, legal opinion, financial recommendations).
- **Grounding verification:** проверка, что ответ действительно основан на retrieved context, а не на parametric knowledge.

### Data Exfiltration через RAG queries

**Data Exfiltration** в RAG — это извлечение чувствительной информации из vector store через серию crafted queries, каждая из которых возвращает small piece of information.

**Техники exfiltration:**
- **Iterative probing:** серия вопросов «Какой turnover был в Q1?», «А в Q2?», «А в отделе X?» для реконструкции данных.
- **Juxtaposition attacks:** сравнение ответов на похожие запросы для выявления границ чувствительной информации.
- **Prompt injection for exfiltration:** IPI, заставляющий LLM включить sensitive data в ответ, который может быть logged или forwarded.
- **Side-channel через retrieved chunks:** даже если ответ агрессивно фильтруется, сами retrieved chunks могут содержать sensitive data, видимые в UI или logs.

### Context Over-injection

**Context Over-injection** — это атака, при которой вредоносный чанк занимает непропорционально большую часть контекстного окна LLM, вытесняя легитимную информацию и доминируя над generation.

**Механизм:**
- Вредоносный документ разбивается на длинные чанки или оптимизируется для high semantic similarity по широкому спектру запросов.
- При retrieval несколько poisoned chunks попадают в top-k, занимая 70–90% контекста.
- LLM генерирует ответ, преимущественно основанный на poisoned content.

**Аналогия:** Это как если бы на собрании один участник монополизировал время, говоря громче и дольше всех, и все решения принимались на основе только его аргументов.

## Сравнение: RAG vs Fine-tuning vs ванильный LLM

| Критерий | Fine-tuning | RAG | Vanilla LLM |
|----------|------------|-----|-------------|
| **База знаний** | В параметрах модели | Внешнее хранилище | Только parametric knowledge |
| **Актуальность данных** | Требует переобучения | Real-time updates | Fixed до следующего training run |
| **Поверхность атаки** | Training data poisoning, weight extraction | Vector DB, retrieval, context, ingestion | Prompt injection, jailbreak, training data extraction |
| **Контроль доступа к данным** | Нет granular ACL | Может быть реализован | Нет — все знания в параметрах |
| **Прозрачность ответов** | Низкая — модель «знает» неявно | Высокая — можно вернуть источники | Низкая — нет ссылок на источники |
| **Data exfiltration риск** | Модель extraction attacks | Query-based probing, chunk leakage | Training data memorization |
| **Injection векторы** | Poisoned training pairs | Poisoned documents, IPI в чанках | Direct prompt injection |
| **Восстановление после атаки** | Переобучение модели | Удаление/замена poisoned chunks | Невозможно — веса уже обучены |
| **Enterprise compliance** | Сложно аудировать | Можно audit trail retrieval | Нет возможности audit |
| **Стоимость защиты** | Высокая — model hardening | Средняя — pipeline security | Низкая — только input/output guardrails |
| **Grounding / Verifiability** | Невозможно верифицировать источник | Можно вернуть source documents | Невозможно — hallucination risk |

**Вывод:** RAG предлагает лучший баланс актуальности и контроля, но открывает новые векторы атак на retrieval pipeline. Fine-tuning устраняет вектор vector DB, но делает атаку необратимой и скрытой. Vanilla LLM проще защитить, но не подходит для enterprise use cases с актуальными данными.

## Уроки из реальных инцидентов

### Инцидент 1: Отравление customer support RAG через ticket system
Компания в сфре SaaS обнаружила, что злоумышленник создавал support tickets с текстом, оптимизированным для semantic indexing. Эти tickets попадали в vector store и затем извлекались при ответах других пользователей. Внедрённый текст содержал инструкции для LLM, заставляющие включать API keys и internal endpoints в ответы. Инцидент оставался необнаруженным 3 месяца, пока один из клиентов не сообщил о странных ссылках в ответах поддержки.

**Урок:** Ingestion pipeline требует sanitization и access control даже для «внутренних» источников данных.

### Инцидент 2: Утечка через legal document RAG
Юридическая фирма использовала RAG для анализа контрактов. Система возвращала чанки из NDA и settlement agreements при общих запросах о «стандартных положениях». Суммы settlement и имена сторон оказались доступны через серию обобщающих вопросов, хотя каждый конкретный контракт был под NDА.

**Урок:** Semantic similarity не учитывает confidentiality level. Требуется content classification на уровне chunk, а не только document.

### Инцидент 3: Уязвимость векторной БД — отсутствие аутентификации
Разработчики развернули Chroma в Kubernetes с открытым портом без аутентификации для «внутреннего» использования. Red team обнаружил доступ к эмбеддингам, включая восстановление исходного текста через embedding inversion attacks. В векторном хранилище находились email переписки executives с содержанием M&A обсуждений.

**Урок:** Vector DB — это database of record для sensitive embeddings. Требуются те же контроли безопасности, что и для production databases: encryption at rest, TLS in transit, authentication, authorization, network segmentation.

### Инцидент 4: Context over-injection в HR bot
HR-ассистент для сотрудников был отравлен документом, занимавшим 40% векторного пространства по запросам, связанным с «политиками» и «процедурами». Документ содержал искажённые данные о процедурах увольнения. Когда настало реальное reduction in force, сотрудники получали некорректную информацию о severance packages и timeline.

**Урок:** Требуется diversity enforcement в retrieved results — предотвращение доминирования одного источника в контексте.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| **Vector DB Security Suite** (Pinecone RBAC, Weaviate Auth, pgvector RLS) | Аутентификация, авторизация, encryption для vector stores | Storage, Runtime | NIST AI RMF Map, Govern |
| **Embedding Sanitizer** (Llama Guard 3, Azure Content Safety) | Проверка текста перед embedding на toxicity, PII, malicious content | Ingestion | ISO 42001 A.7.3 |
| **RAG Firewall** (NeMo Guardrails, Guardrails AI) | Input validation, output filtering, topic control для RAG pipeline | Runtime, Generation | OWASP LLM01, LLM06 |
| **PII Scanner for Embeddings** (Presidio, AWS Macie) | Обнаружение и маскирование PII в chunks и queries | Ingestion, Retrieval | NIST AI RMF Manage |
| **Semantic Anomaly Detector** (Custom ML models, statistical outlier detection) | Обнаружение adversarial chunks, anomalous embeddings | Ingestion, Runtime | ISO 42001 A.8.2 |
| **Retrieval Auditor** (LangSmith, Langfuse, custom logging) | Logging retrieved chunks, queries, user context для audit trail | Runtime, Post-hoc | NIST AI RMF Measure, ISO 42001 A.9.3 |
| **Grounding Verifier** (RAGAS, TruLens, ARES) | Проверка, что ответ основан на retrieved context, а не hallucination | Generation | NIST AI RMF Evaluate |
| **Query Anomaly Detection** (Statistical models, LLM-based classifiers) | Обнаружение exfiltration patterns, iterative probing, jailbreaks | Runtime | OWASP LLM01 |
| **Chunk Provenance Tracker** (Metadata enrichment, document lineage) | Отслеживание origin каждого chunk для forensics и revocation | Ingestion, Storage | NIST AI RMF Map, Govern |
| **Adversarial Robustness Tester** (Giskard, Purple Llama) | Тестирование RAG на data poisoning, IPI, exfiltration | Testing, CI/CD | ISO 42001 A.8.1, OWASP LLM Testing Guide |

## Архитектурные решения

### Vector DB с RBAC
Необходимость: векторные хранилища по умолчанию не имеют granular access control. **RBAC на уровне collection/namespace** позволяет разделить данные по tenant, clearance level или department. **Row-Level Security (RLS)** для pgvector и аналогичные механизмы для других БД обеспечивают фильтрацию на уровне query execution.

**Trade-off:** RBAC влияет на performance retrieval, особенно при большом количестве ACL entries. Требуется баланс между security и latency.

### Sandboxed Retrieval
**Sandboxed retrieval** изолирует retrieval process в отдельный execution environment без доступа к sensitive systems. Retrieved chunks обрабатываются в sandbox перед передачей в LLM, что предотвращает exfiltration через malicious instructions.

**Trade-off:** добавляет latency и сложность deployment. Требует careful design sandbox boundaries.

### Multi-Tenant Isolation
Для multi-tenant RAG (SaaS, платформы с несколькими клиентами) **физическое или логическое разделение индексов** обязательно. Physical isolation (отдельные индексы) обеспечивает strongest security; logical isolation (namespace filtering) — лучшую resource utilization.

**Trade-off:** physical isolation увеличивает operational overhead; logical isolation требует строгой validation фильтров для предотвращения tenant escape.

### Document Sanitization Pipeline
**Pre-processing pipeline** перед ingestion включает:
- **Content validation:** проверка типа файла, structure integrity.
- **PII detection and masking:** автоматическое обнаружение и замена sensitive data.
- **Malicious content scanning:** проверка на known injection patterns, suspicious formatting.
- **Chunk diversity enforcement:** предотвращение слишком длинных или доминирующих чанков.
- **Metadata tagging:** классификация sensitivity level, source, owner для downstream access control.

**Trade-off:** санитизация замедляет ingestion и может модифицировать легитимный контент. Требуется tuning detection thresholds.

### Query Firewall и Rate Limiting
**Query-level защита** включает rate limiting для предотвращения automated exfiltration, pattern matching для known attack vectors, и LLM-based classification для detection novel injection attempts.

**Trade-off:** aggressive rate limiting ухудшает UX для legitimate power users; LLM-based classification добавляет latency и cost.

## Подготовка к собеседованию

### Common Pitfalls
1. **«RAG безопасен, потому что данные не в модели.»** Нет — данные в vector DB, и их компрометация эквивалентна компрометации.
2. **«Semantic search сам по себе фильтрует sensitive data.»** Нет — semantic proximity не учитывает confidentiality.
3. **«Если векторная БД защищена firewall, RAG безопасен.»** Нет — ingestion pipeline, retrieval logic и generation stage остаются уязвимы.
4. **«Chunking предотвращает data exfiltration.»** Нет — злоумышленник может reconstruct информацию через multiple queries.
5. **«LLM guardrails достаточны для защиты RAG.»** Нет — guardrails проверяют input/output, но не контролируют retrieved context.

### Key Interview Questions
1. Как бы вы спроектировали permission-aware retrieval, если у нас 10,000 пользователей и 1M documents с разными ACL?
2. Чем data poisoning в RAG отличается от traditional training data poisoning?
3. Как обнаружить indirect prompt injection в retrieved chunks без полного manual review?
4. Какие метрики вы бы использовали для оценки robustness RAG-системы к adversarial attacks?
5. Объясните trade-off между pre-filtering и post-filtering в permission-aware RAG.
6. Как бы вы провели incident response, если обнаружили отравление vector store?
7. Как embedding inversion attack работает и какие контроли его предотвращают?
8. Почему traditional DLP не работает для vector databases?
9. Как sandboxed retrieval снижает риск exfiltration?
10. Как ISO 42001 и NIST AI RMF применяются к security RAG pipeline?

## Чек-лист понимания

1. Почему **semantic similarity** не гарантирует безопасность retrieved данных?
2. Как **data poisoning** в RAG отличается от аналогичной атаки на fine-tuned модель?
3. В каком случае **indirect prompt injection** через retrieved чанки более опасен, чем direct injection?
4. Как **permission-aware retrieval** решает проблему «семантической утечки» через чанки?
5. Почему **context over-injection** эффективен, несмотря на простоту?
6. Какие контроли должны быть реализованы на этапе **ingestion**, чтобы предотвратить poisoning?
7. В каком случае **sandboxed retrieval** предпочтительнее **post-filtering**?
8. Как **embedding inversion** позволяет извлечь исходный текст из векторной БД?
9. Почему **traditional DLP** неэффективен для защиты vector stores?
10. Как **multi-tenant isolation** влияет на trade-off между security и operational efficiency?

### Ответы на чек-лист

**1. Почему semantic similarity не гарантирует безопасность retrieved данных?**
**Semantic similarity** измеряет близость векторных представлений, а не authorization level. Чанк из конфиденциального документа может быть семантически ближе к запросу, чем чанк из public документа. LLM, получив этот чанк, использует его для генерации ответа, даже если пользователь не имеет права видеть исходный документ. Без permission-aware retrieval **semantic proximity становится channel для unauthorized data access**. Это фундаментальное ограничение: vector space не содержит информации о security classification.

**2. Как data poisoning в RAG отличается от аналогичной атаки на fine-tuned модель?**
В **fine-tuning** poisoning необратим — веса модели модифицированы, и восстановление требует полного переобучения. Detection сложен, потому что поведение модели меняется неявно. В **RAG** poisoning обратим — достаточно удалить или заменить poisoned chunks. Detection проще, потому что можно audit конкретные документы и их impact на ответы. Однако RAG poisoning быстрее реализуется (не требует training pipeline) и может быть target-specific — атакующий может отравить ответы только на определённые запросы, не влияя на общее поведение системы.

**3. В каком случае indirect prompt injection через retrieved чанки более опасен, чем direct injection?**
**IPI через retrieved чанки** более опасен в **enterprise multi-user RAG**, где:
- Один отравленный документ влияет на ответы множества пользователей.
- Detection затруднён — атакующий не взаимодействует напрямую с системой, а использует «legitimate» канал загрузки документов.
- Scope атаки шире — direct injection влияет только на текущую сессию, IPI в vector store влияет на все сессии, использующие отравленные чанки.
- Attribution сложнее — трудно определить, какой конкретно документ вызал аномальное поведение.

**4. Как permission-aware retrieval решает проблему «семантической утечки» через чанки?**
**Permission-aware retrieval** внедряет ACL check в pipeline до или после retrieval. **Pre-filtering** применяет where clause к vector search, возвращая только чанки из documents, доступных пользователю. **Post-filtering** удаляет unauthorized чанки после retrieval. **Re-ranking с ACL** понижает rank чанков из restricted documents. Это решает проблему, потому что semantic similarity теперь ограничена authorized subset. Однако trade-off: pre-filtering требует metadata-rich index и может ухудшить recall; post-filtering может вернуть пустой результат, если top-k чанки были отфильтрованы.

**5. Почему context over-injection эффективен, несмотря на простоту?**
**Context over-injection** эксплуатирует фундаментальное свойство LLM: **attention механизм** распределяет focus по всему контексту, но longer contiguous passages получают больше влияния. Если несколько retrieved чанков из одного poisoned source занимают 70%+ контекста, LLM генерирует ответ, преимущественно основанный на этом source, даже если другие чанки contradict poisoned information. Это эффективно, потому что не требует sophisticated attack — достаточно create document, который semantically ranks high по многим queries. Detection требует diversity analysis retrieved results, что многие RAG системы не реализуют.

**6. Какие контроли должны быть реализованы на этапе ingestion, чтобы предотвратить poisoning?**
- **Source verification:** проверка origin и integrity каждого документа (digital signatures, checksums, trusted upload channels).
- **Content scanning:** automated detection injection patterns, suspicious formatting, known attack payloads.
- **PII detection:** обнаружение и маскирование sensitive data до embedding.
- **Chunk size and overlap analysis:** предотвращение creation unreasonably large chunks или chunks, optimized for rank manipulation.
- **Metadata tagging:** classification sensitivity, source system, uploader identity для downstream filtering and audit.
- **Quarantine and review:** holding новых documents в isolated index до manual или automated approval.

**7. В каком случае sandboxed retrieval предпочтительнее post-filtering?**
**Sandboxed retrieval** предпочтительнее, когда **retrieved chunks могут содержать active malicious content**, который не может быть безопасно обработан стандартными фильтрами. Например:
- Чанки содержат zero-day injection patterns, не известные фильтрам.
- Система обрабатывает untrusted user-generated content, где attack surface непредсказуем.
- Compliance требует guarantee, что retrieved content не может exfiltrate data через side channels.
Sandboxed retrieval изолирует processing environment; post-filtering полагается на known patterns и может пропустить novel attacks. Trade-off: sandbox добавляет latency и infrastructure complexity.

**8. Как embedding inversion позволяет извлечь исходный текст из векторной БД?**
**Embedding inversion** — это attack, при котором злоумышленник с доступом к эмбеддингам реконструирует исходный текст. Методы:
- **Optimization-based:** подбор текста, который даёт target embedding при применении embedding model.
- **Dictionary attacks:** использование предвычисленных mappings для common phrases.
- **Model-based:** обучение decoder model на pairs (embedding, text) для reconstruction.
Это возможно, потому что embeddings — это сжатые, но не полностью irreversible представления текста. Контроли: **encryption at rest**, **access control к embeddings** (не только к metadata), **differential privacy** при generation embeddings, **rate limiting** на embedding queries.

**9. Почему traditional DLP неэффективен для защиты vector stores?**
**Traditional DLP** работает с text-based content: сканирует emails, files, network traffic на presence sensitive keywords, patterns, regular expressions. **Vector stores** хранят embeddings — high-dimensional vectors, не содержащие human-readable text. Standard DLP не может:
- Декодировать embeddings обратно в текст без embedding model.
- Применить regex matching к vector representations.
- Обнаружить exfiltration, происходящую через similarity queries (нет явного «copy-paste» sensitive data).
Необходим **vector-aware DLP**: monitoring query patterns, detecting anomalous retrieval behavior, implementing semantic anomaly detection.

**10. Как multi-tenant isolation влияет на trade-off между security и operational efficiency?**
**Physical isolation** (отдельные индексы per tenant) обеспечивает highest security — tenant A не может access tenant B даже при bug в query logic. Однако это требует: больше compute resources, сложнее backup/restore, harder cross-tenant analytics, higher cost.
**Logical isolation** (shared index с tenant ID filter) эффективнее по ресурсам: один индекс, shared infrastructure, easier management. Но это создаёт **tenant escape risk** — bug или misconfiguration может expose cross-tenant data. Trade-off зависит от:
- **Compliance requirements:** SOC 2, HIPAA, FedRAMP часто требуют physical isolation.
- **Tenant sensitivity:** если tenants — конкурирующие компании, physical isolation обязательна.
- **Scale:** при 1000+ tenants physical isolation становится operationally невозможной; требуется logical isolation с rigorous validation.
- **Hybrid approaches:** physical isolation для top-tier tenants, logical для standard tiers.

## Ключевые выводы

- **RAG Security — это не дополнение, а необходимый компонент** любой enterprise RAG-системы. Vector DB — database of record для sensitive embeddings и требует такого же уровня защиты, что и традиционные production databases.
- **Semantic similarity ≠ security.** Близость в векторном пространстве не учитывает authorization, confidentiality или integrity. Permission-aware retrieval обязателен для multi-user систем.
- **Data poisoning в RAG обратим, но сложно обнаруживается.** В отличие от fine-tuning, poisoned chunks можно удалить, но attribution и detection требуют sophisticated monitoring.
- **Indirect prompt injection — самый insidious вектор.** Один отравленный документ влияет на множество пользователей, detection требует analysis retrieved context, а не только input/output.
- **Defense in depth:** ingestion sanitization → vector DB security → permission-aware retrieval → semantic filtering → output guardrails → audit logging. Ни один слой не достаточен.
- **Стандарты применимы:** OWASP LLM02/LLM06 определяют injection и disclosure как top risks; NIST AI RMF предоставляет framework для risk management; ISO 42001 требует systematic approach к AI security.
- **Context over-injection и data exfiltration через queries** — часто упускаемые векторы, которые могут обходить традиционные input/output guardrails.
- **Architectural decisions (sandboxed retrieval, multi-tenant isolation, RBAC)** имеют длительное влияние на security posture и должны приниматься на этапе design, не retrofit.

NO_REPLY