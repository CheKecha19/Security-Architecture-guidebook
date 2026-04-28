# LLM Authorization and Governance: Управление доступом и аудит AI-систем

## Что такое governance для LLM?

Представьте, что в крупной компании появляется новый сотрудник — AI-ассистент. Он не подчиняется ни одному отделу, не имеет начальника, не следует никаким процедурам. Он просто берёт данные, которые ему дали, и отвечает всем, кто спрашивает. Хаос? Именно. Governance для LLM — это система управления AI-ассистентами: кто может их использовать, какие данные они могут видеть, какие действия выполнять, как контролировать их работу.

Governance (управление) LLM — это политики, процедуры и технические средства для контроля AI-систем. Включает: RBAC (кто может использовать модель), ACL (какие данные доступны), audit (что делает модель), compliance (соответствие законам), lifecycle (управление версиями моделей), risk management (оценка рисков).

Без governance LLM — это «чёрный ящик», который может делать что угодно с любыми данными. Governance — это правила, по которым AI работает. Без правил — анархия, риски, штрафы. С правилами — контролируемый, предсказуемый, аудируемый AI.

## Зачем это нужно?

**Сценарий 1: RBAC для HR-ассистента.** HR-ассистент Сбера имеет разные уровни доступа: рядовой сотрудник — может спросить о политике отпусков, но не может узнать зарплаты коллег; HR-специалист — может получить доступ к зарплатам и оценкам; руководитель — может получить аналитику по отделу. Без RBAC — любой может спросить любую информацию. RBAC — базовый элемент governance.

**Сценарий 2: Audit всех AI-действий.** Все запросы к HR-ассистенту логируются: user_id, запрос, ответ, время, использованные документы. При инциденте (утечка PII, неправильный совет) — audit trail позволяет восстановить, что произошло. Кто спросил, что ответила модель, какие документы были в контексте, сработали ли guardrails.

**Сценарий 3: Model lifecycle management.** Команда AI выпускает новую версию HR-ассистента v2.0. Новая модель прошла тестирование и red teaming. Старая v1.3 выводится из эксплуатации. Все запросы автоматически перенаправляются на v2.0. При проблемах — откат на v1.3. Lifecycle management — управление версиями моделей с контролем качества.

## Основные концепции

### RBAC для LLM

RBAC (Role-Based Access Control) — управление доступом на основе ролей. Для LLM RBAC определяет, какие действия может выполнять пользователь.

**Роли для LLM:**
- **User (потребитель).** Может отправлять запросы, получать ответы. Не может менять модель, данные, конфигурацию.
- **Power User.** Может настраивать контекст, загружать документы в личное пространство.
- **Admin.**
- Может управлять моделью (развёртывать, откатывать), настраивать guardrails, управлять доступом.
- **Auditor.** Может читать логи, но не может отправлять запросы.

**Scope-доступ:**
- Read-only (только запросы, изменение данных запрещено)
- Write (может загружать документы в RAG-базу)
- Admin (может менять конфигурацию)

**Пример RBAC для HR-ассистента:**
- Сотрудник: role=user, scope=public-documents, quota=100 req/day
- HR-специалист: role=power-user, scope=hr-documents, quota=500 req/day
- Руководитель: role=power-user, scope=department-analytics, quota=200 req/day
- DevSecOps: role=admin, scope=all, quota=unlimited
- Compliance: role=auditor, scope=logs, quota=read-only

### Policy as Code для LLM

Policy as Code — политики управления LLM, записанные в виде кода (YAML, JSON). Политики могут быть проверены, автоматизированы, версионированы.

**Пример политики (OPA/Rego):**
```yaml
# Политика доступа к документам
rule "hr-assistant-document-access" {
  if user.role == "employee"
  then allow access to documents with scope "public"
  
  if user.role == "hr-specialist"
  then allow access to documents with scope "hr" or "public"
  
  if user.department == user.document.department
  then allow access to department-specific documents
}
```

**Платформы Policy as Code:** Open Policy Agent (OPA), Kyverno (Kubernetes), HashiCorp Sentinel.

Policy as Code позволяет:
- Версионировать политики (Git)
- Тестировать политики (unit tests)
- Автоматизировать проверку (CI/CD)
- Audit — кто, когда изменил политику

### Audit Logging

Audit logging — запись всех действий с LLM для последующего анализа. Логи содержат:

**Обязательные поля:**
- timestamp — когда произошло действие
- user_id — кто выполнил действие
- action — что сделано (query, upload, config change)
- resource — к какому ресурсу доступ (модель, документ)
- result — успех/неудача
- metadata — IP, user agent, session_id

**Специфичные для LLM:**
- prompt (masked) — запрос пользователя (с PII-маскированием)
- response (masked) — ответ модели (с PII-маскированием)
- documents_used — какие документы были в RAG-контексте
- guardrails_triggered — какие guardrails сработали
- confidence_score — уверенность модели в ответе
- latency — время ответа

**Хранение логов:**
- WORM-хранилище (Write Once, Read Many) — защита от изменения
- Encryption at rest — шифрование
- RBAC — доступ только для auditors
- Retention — 1-3 года (по требованию комплаенс)
- Backup — резервное копирование

### Model Lifecycle Management

Model lifecycle — управление версиями моделей от разработки до вывода из эксплуатации.

**Этапы lifecycle:**
1. **Development** — обучение модели, тестирование
2. **Staging** — проверка на staging, red teaming, метрики
3. **Approval** — approval безопасности, комплаенс
4. **Production** — развёртывание для пользователей
5. **Monitoring** — мониторинг в production (латентность, качество ответов, guardrails)
6. **Update** — обновление модели (новая версия)
7. **Retirement** — вывод из эксплуатации (архивация для audit)

**Blue/Green deployment.** Две версии модели одновременно: Blue (текущая production), Green (новая). Трафик постепенно переключается на Green. При проблемах — мгновенный откат на Blue.

**Canary deployment.** Новая модель сначала получает 10% трафика. Если метрики стабильны — 50%, затем 100%. Canary снижает риск: если новая модель хуже — пострадают только 10% пользователей.

**Rollback plan.** При проблемах — автоматический откат на предыдущую версию. Время отката — минуты, а не дни.

### Risk Assessment для LLM

Risk assessment — оценка рисков, связанных с внедрением LLM. Документ, описывающий: какие модели используются, какие данные обрабатываются, какие риски существуют, какие меры защиты применяются.

**Структура risk assessment:**
- Asset — что защищаем (модели, данные, пользователи)
- Threat — что угрожает (prompt injection, data leakage, model theft)
- Vulnerability — почему уязвимы (LLM фундаментально не разделяет инструкции и данные)
- Impact — какой ущерб (утечка PII — штрафы, нарушение compliance)
- Likelihood — вероятность (prompt injection — высокая, model theft — средняя)
- Mitigation — защита (guardrails, DP, RBAC, audit)
- Residual risk — остаточный риск после mitigation

**Инструменты:** FAIR (Factor Analysis of Information Risk), NIST AI RMF (Risk Management Framework), MITRE ATLAS (Adversarial Threat Landscape for AI Systems).

## Сравнение ролей в LLM governance

| Роль | Права | Ответственность | Доступ |
|------|-------|-----------------|--------|
| User | Запросы | Не разглашать ответы | Public docs |
| Power User | Запросы + загрузка документов | Качество документов | Department docs |
| Admin | Управление моделью + guardrails | Безопасность модели | Все |
| Auditor | Только чтение логов | Анализ инцидентов | Логи |
| Compliance | Approval деплоя | Соответствие законам | Метаданные |

## Практические применения

**Сценарий 1: RBAC для HR-ассистента Сбера.**
Созданы роли: employee (только public docs), hr-specialist (hr docs), manager (свои department docs), admin (управление моделью). При запросе сотрудника: запрос → authentication → RBAC check → RAG с фильтром по scope → LLM → output guard → response. Сотрудник видит только то, что разрешено его ролью.

**Сценарий 2: Policy as Code для LLM.** Политики LLM записаны в OPA/Rego и хранятся в Git. Каждое изменение политики — pull request + code review + тесты. При деплое политики автоматически применяются к LLM-шлюзу (Envoy, Kong, Tyk). Политики версионируются: v1.0 (базовый RBAC), v1.1 (добавлен scope для department), v2.0 (добавлен compliance filter).

**Сценарий 3: Audit dashboard.**
Grafana дашборд для мониторинга LLM:
- Топ запросов (какие вопросы задают чаще всего)
- Guardrails triggers (сколько запросов заблокировано, какие guardrails)
- Latency (среднее время ответа, медленные запросы)
- Anomaly detection (аномальные паттерны запросов)
- User activity (активность по ролям, по отделам)
Дашборд — основа для еженедельного review безопасности LLM.

**Сценарий 4: Canary deployment новой модели.** Новая версия HR-ассистента v2.0 развёрнута для 10% пользователей. Мониторинг: метрики качества ответов (сравнение с v1.3), guardrails triggers (не увеличилось ли?), latency (не замедлилась ли?), user feedback (жалобы?). Через 24 часа — расширение до 50%. Через 48 часов — 100% или откат.

**Сценарий 5: Risk assessment для LLM.** Документ: AI-ассистент для HR (asset). Threat: prompt injection (high), data leakage (medium), DoS (low). Impact: утечка PII — high, неправильные советы — medium, простой — low. Mitigation: input guard, output guard, RBAC, rate limit, audit. Residual risk: prompt injection (medium — guardrails не 100%). Решение: внедрить с current mitigation, мониторинг, red teaming.

## Ограничения и нюансы

**RBAC — сложно для больших организаций.** При 50 отделах и 50 000 сотрудниках — управление ролями становится нетривиальной задачей. Нужна автоматизация: group-based access, ABAC (attribute-based), автоматическая синхронизация с LDAP/AD.

**Policy as Code — не для всех.** OPA/Rego — сложный язык. Разработчики политик — редкая специализация. Обучение команды policy as code — затраты времени и бюджета.

**Audit logging — дорого для LLM.** Каждый запрос к LLM генерирует много данных (prompt, response, контекст, метаданные). Хранение всех логов — терабайты в месяц. Нужна политика: краткосрочные логи (30 дней), archive (1 год), delete (после retention).

**Canary deployment — не защищает от всех проблем.** Prompt injection на новую модель будет работать и для 10%, и для 100% пользователей. Canary — защита от снижения качества, не от атак. Безопасность модели должна быть проверена до canary.

**Risk assessment — не заменяет реальную защиту.** Документ risk assessment — это анализ, не защита. После документа нужно внедрять mitigation. Risk assessment без action — бюрократия.

## Архитектурные решения

**Centralized API gateway для LLM.** Все запросы к LLM проходят через единый API gateway (Kong, Tyk, Envoy). Gateway реализует: authentication, RBAC, rate limiting, audit logging. Centralized gateway — единая точка контроля политик.

**Policy engine.** OPA/Kyverno как единый движок политик для всех LLM. Политики: RBAC, guardrails, compliance. Policy engine — «мозг» governance.

**Audit as a service.** Все логи LLM отправляются в центральную audit-систему (ELK, Splunk). Audit — не «запись всего подряд», а структурированные данные с поиском, alerting, dashboard.

**Human-in-the-loop.** Критические действия AI требуют подтверждения человека. AI предлагает, человек утверждает. Human-in-the-loop — последняя линия защиты от AI-ошибок.

## Чек-лист понимания

- [ ] Какие роли существуют в RBAC для LLM?
- [ ] Что такое Policy as Code и какие инструменты используются?
- [ ] Какие поля обязательны для audit logging LLM?
- [ ] Какие этапы model lifecycle management?
- [ ] Чем canary deployment отличается от blue/green?
- [ ] Что такое risk assessment для LLM?
- [ ] Как API gateway централизует управление LLM?
- [ ] Почему human-in-the-loop важен для критических AI-действий?
- [ ] Как OPA/Kyverno используются для политик LLM?
- [ ] Какие compliance требования влияют на governance?

### Ответы на чек-лист

1. **Какие роли существуют в RBAC для LLM?** User (потребитель — запросы), Power User (загрузка документов), Admin (управление моделью, guardrails), Auditor (чтение логов), Compliance (approval деплоя). Каждая роль имеет scope (доступ к документам), quota (лимит запросов), actions (какие операции разрешены). RBAC реализуется на API gateway перед LLM.

2. **Что такое Policy as Code и какие инструменты используются?** Policy as Code — политики управления LLM в виде кода (YAML/Rego). Инструменты: Open Policy Agent (OPA) — универсальный движок политик, Kyverno — для Kubernetes, HashiCorp Sentinel — для HashiCorp экосистемы. Policy as Code: хранится в Git, версионируется, тестируется. Изменение политики — pull request + review.

3. **Какие поля обязательны для audit logging LLM?** Обязательные поля: timestamp, user_id, action (query/upload/config), resource (модель/документ), prompt (masked), response (masked), documents_used, guardrails_triggered, result (success/fail). Метаданные: IP, user_agent, session_id, latency. Retention: 1-3 года. Хранилище: WORM, encrypted, RBAC.

4. **Какие этапы model lifecycle management?** (1) Development, (2) Staging (red teaming, тесты), (3) Approval (security, compliance), (4) Production, (5) Monitoring (латентность, качество, guardrails), (6) Update (новая версия), (7) Retirement (архивация). Каждый этап — gate with approval.

5. **Чем canary deployment отличается от blue/green?** Blue/Green — две версии (Blue = current, Green = new). Трафик мгновенно переключается на Green. Откат — обратно на Blue. Canary — постепенный rollout: 10% → 50% → 100%. Canary безопаснее (проблемы затрагивают только 10%), но медленнее (часы). Blue/Green быстрее, но рискованнее.

6. **Что такое risk assessment для LLM?** Risk assessment — документ, описывающий риски LLM. Компоненты: asset (модель, данные), threat (prompt injection, data leakage), vulnerability (LLM — не разделяет инструкции и данные), impact (утечка PII — штрафы), likelihood (prompt injection — high), mitigation (guardrails, RBAC, audit), residual risk (остаточный после защиты). Инструменты: FAIR, NIST AI RMF, MITRE ATLAS.

7. **Как API gateway централизует управление LLM?** API gateway — единая точка входа для всех запросов к LLM. Gateway реализует: authentication (JWT, OAuth), RBAC (role check), rate limiting (100 req/min), audit logging (все запросы), guardrails (input guard). Centralized gateway: единый контроль, единый audit, единые политики. Примеры: Kong, Tyk, Envoy + OPA.

8. **Почему human-in-the-loop важен для критических AI-действий?** Human-in-the-loop (HITL) — подтверждение человека для AI-действий. AI предлагает → человек утверждает. HITL защищает от: prompt injection (AI может предложить вредоносное действие), галлюцинаций (AI предлагает неверное решение), compliance (решения с PII требуют approval человека). HITL — последняя линия защиты.

9. **Как OPA/Kyverno используются для политик LLM?** OPA/Kyverno — policy engines, проверяющие запросы на соответствие политикам. Интеграция: API gateway → OPA → LLM. Политики OPA: user.role == «employee» → разрешить доступ к public docs, prompt содержит PII → блокировать. Kyverno — для Kubernetes: pod с LLM должен иметь sidecar с guardrails. OPA/Kyverno делают политики автоматическими, версионируемыми, тестируемыми.

10. **Какие compliance требования влияют на governance?** GDPR (Europe): DPIA, right to be forgotten, Article 22 (человеческий контроль). 152-ФЗ (РФ): согласие на обработку ПД, локализация данных, уведомление Роскомнадзора. CCPA (California): opt-out, disclosure. SOC 2 (в США): audit, RBAC, encryption. ISO 27001: risk assessment, policy, audit. PCI DSS (если AI обрабатывает платежи): шифрование, RBAC, audit. Compliance — обязательное требование для enterprise LLM.

---

_Статья создана на основе анализа материалов: OWASP Top 10 для LLM: Excessive Agency, Overreliance (habr.com/ru/companies/bastion/articles/960918), Безопасность LLM (codeby.net/threads/bezopasnost-llm-polnaya-karta-atak-na-yazykovyye-modeli), LLM Firewall (habr.com/ru/articles/981408), Guardrails для LLM (habr.com/ru/articles/1024028), AI Governance (habr.com/ru/companies/ru_mts/articles/841010)_
