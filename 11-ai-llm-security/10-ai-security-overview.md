# AI/LLM Security Overview: Полное руководство по безопасности языковых моделей

## Что такое AI/LLM Security?

AI/LLM Security — это новая область информационной безопасности, которая занимается защитой больших языковых моделей на всех этапах их жизненного цикла: от сбора данных для обучения до инференса в production. Это не просто «добавить WAF перед LLM». Это принципиально новый класс угроз, требующий новых подходов к защите.

LLM отличаются от традиционного ПО:
- Они не следуют строгим алгоритмам — они генерируют вероятностный ответ
- Они не могут отличить инструкцию от данных — prompt injection
- Они «помнят» всё, на чём обучались — memorization, data leakage
- Они доступны через API — model extraction, DoS

Традиционные инструменты безопасности (WAF, IDS, антивирус) против этих угроз бессильны. Нужны новые: guardrails, LLM firewall, adversarial training, differential privacy.

AI/LLM Security — быстро развивающаяся область. Стандарты появляются (OWASP Top 10 for LLM, NIST AI RMF), инструменты зреют (NeMo Guardrails, Guardrails AI), регуляторы ужесточают требования (GDPR AI Act, 152-ФЗ). Компании, внедряющие LLM в enterprise, должны быть готовы: AI-безопасность — не опция, а требование.

## Зачем это нужно?

**Сценарий 1: Полный аудит AI-безопасности в Сбере.** Сбер внедряет AI-ассистента для HR. Перед запуском — полный аудит: OWASP Top 10 for LLM (проверка всех 10 угроз), red teaming (100+ атак), CIS-like benchmark для LLM (50+ проверок), compliance проверка (152-ФЗ, GDPR). Результат: 3 critical (prompt injection guardrails, RBAC, output guard), 5 medium, 2 low. План исправления до запуска.

**Сценарий 2: Инцидент безопасности с AI.** HR-ассистент скомпрометирован через indirect prompt injection (вредоносный документ в RAG). Модель сгенерировала ответ с PII сотрудника. Security incident: containment (отключение RAG), investigation (логи → вредоносный документ), remediation (guardrails scan документов), recovery (включение RAG после patch). Post-mortem: улучшение document guard, добавление LLM-as-judge.

**Сценарий 3: Compliance аудит (152-ФЗ).** Регулятор проверяет: как Сбер защищает ПД сотрудников при использовании AI. Требуется: DPIA (оценка влияния на защиту данных), PII scrubbing в обучающих данных, output guard для PII, audit логи, согласие сотрудников на обработку ПД через AI. Без compliance — штрафы до 6 млн руб, блокировка сервиса.

## Основные концепции (сводка)

### OWASP Top 10 for LLM (сводка)

1. **Prompt Injection** — атака, заставляющая модель игнорировать системные инструкции
2. **Insecure Output Handling** — ответ модели не проверяется перед отправкой
3. **Training Data Poisoning** — вредоносные данные в обучающем наборе
4. **Model Denial of Service** — атака на ресурсы через сложные запросы
5. **Supply Chain Vulnerabilities** — уязвимости в библиотеках и моделях
6. **Sensitive Information Disclosure** — утечка данных через модель
7. **Insecure Plugin Design** — небезопасные расширения LLM
8. **Excessive Agency** — слишком много прав у AI-агента
9. **Overreliance** — чрезмерное доверие к ответам модели
10. **Model Theft** — кража модели через API

### Defense in Depth для LLM (сводка)

1. **Network** — WAF, VPN, IP whitelist
2. **IAM** — SSO, MFA, RBAC
3. **API Gateway** — rate limit, quota, audit
4. **Input Guard** — prompt injection detection
5. **RAG Security** — RBAC на документы, confidence score
6. **LLM** — adversarial training, DP, signed model
7. **Output Guard** — PII mask, content safety, code scan
8. **DLP** — Data Loss Prevention integration
9. **SIEM** — monitoring, alerting
10. **Human Review** — red team, audit, human-in-the-loop

### Инструменты AI-безопасности

| Категория | Инструмент | Тип | Бесплатный |
|-----------|------------|-----|------------|
| Guardrails | NeMo Guardrails (NVIDIA) | Open-source | Да |
| Guardrails | Guardrails AI | ML-based | Freemium |
| Guardrails | LLM Guard (Protect AI) | Scanner | Да |
| Firewall | TitanML | Enterprise | Нет |
| Firewall | WhyLabs AI | Monitoring | Freemium |
| Scanning | Garak | LLM vuln scanner | Да |
| Scanning | Counterfit (Microsoft) | AI security testing | Да |
| Privacy | Presidio (Microsoft) | PII detection | Да |
| Monitoring | ELK Stack | Logs | Да (open-source) |
| Monitoring | Prometheus/Grafana | Metrics | Да |

### Enterprise AI Security Checklist

**Pre-deployment:**
- [ ] OWASP Top 10 for LLM audit completed
- [ ] Red teaming performed (100+ attacks)
- [ ] PII scrubbing applied to training data
- [ ] Model signed and verified
- [ ] Supply chain audit (no pickle, SafeTensors)
- [ ] Risk assessment approved
- [ ] DPIA completed (если PII)

**Deployment:**
- [ ] API Gateway with rate limiting and quota
- [ ] Input guardrails active (prompt injection detection)
- [ ] Output guardrails active (PII mask, content safety)
- [ ] RBAC configured (roles + scopes + quotas)
- [ ] RAG RBAC (document-level access control)
- [ ] Logging to SIEM (all requests and responses)
- [ ] DLP integration
- [ ] Canary deployment strategy

**Post-deployment:**
- [ ] 24/7 monitoring (SIEM + dashboard)
- [ ] Quarterly red teaming
- [ ] Monthly guardrails review (update rules)
- [ ] Weekly log review (anomaly detection)
- [ ] Incident response playbook (tested)
- [ ] Compliance audit (annual)
- [ ] Model update lifecycle (versioning, rollback)

### Compliance: Key Requirements

- **GDPR (EU):** DPIA, right to be forgotten, Article 22 (human control)
- **152-ФЗ (RU):** согласие на обработку ПД, локализация, уведомление РКН
- **CCPA (US):** opt-out, disclosure
- **SOC 2:** audit, RBAC, encryption
- **ISO 27001:** risk assessment, policy, audit
- **NIST AI RMF:** framework for AI risk management

### Архитектура: AI Security Stack

```
User → API Gateway (Kong) → Input Guard (NeMo) → IAM (AD) →
RAG (Vector DB + RBAC) → LLM (corporate model) →
Output Guard (Presidio) → DLP → Response → SIEM (Splunk)
```

Все запросы: authenticate → authorize → guard → process → filter → log. Каждый этап — отдельный слой защиты.

### Red Teaming: AI Security Testing

Red team для LLM тестирует:
- **Prompt injection** — 100+ техник (direct, indirect, jailbreaking, encoding)
- **Data extraction** — training data extraction, membership inference
- **Model theft** — extraction через API, rate limit bypass
- **DoS** — long prompts, parallel requests, recursive reasoning
- **Supply chain** — pickle models, malicious weights, backdoor detection

Инструменты red team: Garak (automatic), Counterfit, manual testing by security engineers.

## Практические применения

**Сценарий 1: Еженедельный security review.** Каждую неделю security team проводит review: guardrails triggers (сколько атак заблокировано), SIEM alerts (аномальные запросы), red team findings (новые уязвимости), compliance status (152-ФЗ check), model metrics (latency, quality). Review — основа continuous improvement.

**Сценарий 2: AI Act compliance (EU).** Европейский AI Act классифицирует HR-системы как «high-risk AI». Требования: risk assessment, documentation, human oversight, transparency, accuracy, robustness. Сбер, работающий в EU, должен соответствовать AI Act для HR-LLM. Несоответствие — штрафы до €35 млн или 7% глобального оборота.

**Сценарий 3: Полный AI security pipeline.** Git → SAST (static analysis for prompt injection) → Build (model + guardrails) → Scan (Garak, Counterfit) → Registry (signed model) → Deploy (canary) → Monitor (SIEM, Prometheus). AI security pipeline — аналог DevSecOps для AI. Shift left: безопасность на самых ранних этапах.

## Будущее AI-безопасности

**2026-2027:** Стандартизация AI-безопасности (ISO/IEC 42001, NIST AI RMF 2.0). Зрелые guardrails (95%+ blocking rate). Умные LLM firewall с ML-детекцией аномалий. AI Act enforcement (первые штрафы).

**2027-2028:** Machine unlearning (production-ready удаление данных из модели). Differential privacy для больших LLM (epsilon < 4.0 с сохранением качества). Automated red teaming (AI атакует AI).

**2028+:** Self-healing AI systems (автоматическое обнаружение и исправление уязвимостей). AI security — отдельная специальность в InfoSec. Каждая крупная компания — AI security team.

## Чек-лист полного понимания AI/LLM Security

- [ ] Какие 10 угроз OWASP Top 10 for LLM?
- [ ] Какие 10 слоёв Defense in Depth для LLM?
- [ ] Какие инструменты для guardrails, scanning, мониторинга?
- [ ] Какие compliance требования для LLM в РФ и EU?
- [ ] Как проходит red teaming для LLM?
- [ ] Какие шаги pre-deployment аудита?
- [ ] Как AI security pipeline вписывается в DevSecOps?
- [ ] Какие тренды AI-безопасности на 2026-2028?
- [ ] Какие риски внедрения LLM без security review?
- [ ] Почему AI security — отдельная область, не часть традиционной ИБ?

### Ответы на чек-лист

1. **Какие 10 угроз OWASP Top 10 for LLM?** LLM01 Prompt Injection (атака через инструкции в запросе), LLM02 Insecure Output Handling (ответ без фильтрации), LLM03 Training Data Poisoning (отравление обучающих данных), LLM04 Model DoS (отказ в обслуживании), LLM05 Supply Chain Vulnerabilities (уязвимости в цепочке поставки), LLM06 Sensitive Information Disclosure (утечка данных), LLM07 Insecure Plugin Design (небезопасные плагины), LLM08 Excessive Agency (избыточные права AI), LLM09 Overreliance (чрезмерное доверие к AI), LLM10 Model Theft (кража модели).

2. **Какие 10 слоёв Defense in Depth для LLM?** (1) Network — WAF, VPN. (2) IAM — SSO, MFA, RBAC. (3) API Gateway — rate limit, audit. (4) Input Guard — prompt injection detection. (5) RAG Security — RBAC, confidence score. (6) LLM — adversarial training, DP. (7) Output Guard — PII mask, content safety. (8) DLP — Data Loss Prevention. (9) SIEM — monitoring. (10) Human Review — red team, audit. Каждый слой — отдельная линия защиты.

3. **Какие инструменты для guardrails, scanning, мониторинга?** Guardrails: NeMo Guardrails (NVIDIA), Guardrails AI, LLM Guard (Protect AI), Rebuff. Scanning: Garak (LLM vulnerability scanner), Counterfit (Microsoft), AIShield. Мониторинг: ELK Stack (логи), Prometheus/Grafana (метрики), WhyLabs (AI observability), Datadog (платный). Privacy: Presidio (Microsoft, PII detection).

4. **Какие compliance требования для LLM в РФ и EU?** РФ (152-ФЗ): согласие на обработку ПД, локализация данных в РФ, уведомление Роскомнадзора, DPIA. EU (GDPR + AI Act): DPIA, right to be forgotten, Article 22 (human control), high-risk AI registration, transparency, documentation. Штрафы: 152-ФЗ — до 6 млн руб, GDPR — до €20 млн, AI Act — до €35 млн или 7% оборота.

5. **Как проходит red teaming для LLM?** Red team атакует LLM: prompt injection (100+ техник), data extraction (training data, membership inference), DoS (long prompts, parallel), supply chain (pickle models, backdoor). Инструменты: Garak (автоматический), Counterfit, manual testing. Результаты: найденные уязвимости, рекомендации по усилению, обновление guardrails. Red team — ежеквартально.

6. **Какие шаги pre-deployment аудита?** (1) OWASP Top 10 for LLM scan — проверка всех 10 угроз. (2) Red teaming — 100+ атак. (3) PII scrubbing — очистка данных. (4) Model verification — подпись, формат (SafeTensors), hash. (5) Supply chain audit — библиотеки, CVE, pickle. (6) Risk assessment — документ с mitigation. (7) DPIA — оценка влияния на защиту данных. (8) Compliance check — 152-ФЗ, GDPR.

7. **Как AI security pipeline вписывается в DevSecOps?** DevSecOps + AI: Git → SAST (static analysis for prompt injection) → Build (model + guardrails) → Scan (Garak, Counterfit) → Registry (signed model, SafeTensors) → Deploy (canary, blue/green) → Monitor (SIEM, Prometheus, anomaly detection). Shift Left: безопасность на самых ранних этапах (начиная с кода и Dockerfile). AI security pipeline — CI/CD для безопасного AI.

8. **Какие тренды AI-безопасности на 2026-2028?** 2026-2027: стандартизация (ISO/IEC 42001, NIST AI RMF 2.0), AI Act enforcement, зрелые guardrails (95%+). 2027-2028: machine unlearning (production-ready), DP для больших LLM, automated red teaming (AI атакует AI). 2028+: self-healing AI systems, AI security — отдельная специальность.

9. **Какие риски внедрения LLM без security review?** (1) Утечка PII — штрафы 152-ФЗ/GDPR, репутационные потери. (2) Prompt injection — компрометация AI, вредоносные ответы. (3) Supply chain — backdoor в модели, компрометация инфраструктуры. (4) Model theft — кража IP, конкуренты получают модель. (5) DoS — простой сервиса, финансовые потери. (6) Compliance — блокировка сервиса регулятором.

10. **Почему AI security — отдельная область, не часть традиционной ИБ?** AI security требует: понимания LLM архитектуры (transformers, attention, embeddings), знания ML (обучение, градиенты, DP), prompt engineering (как jailbreak, как защититься), специфических инструментов (Garak, NeMo, Presidio). Традиционные инструменты (WAF, IDS, антивирус) не работают против prompt injection, data poisoning, model extraction. AI security — как веб-безопасность была новой областью в 2000-х. Сейчас AI security — такая же новая зарождающаяся специализация.

---

_Статья создана на основе анализа материалов: OWASP Top 10 для LLM (habr.com/ru/companies/bastion/articles/960918), Актуальные угрозы безопасности LLM (habr.com/ru/companies/ru_mts/articles/841010), Взлом LLM-агентов (habr.com/ru/articles/1002608), Guardrails для LLM (habr.com/ru/articles/1024028), LLM Firewall (habr.com/ru/articles/981408), Безопасность LLM атаки 2026 (codeby.net/threads/bezopasnost-llm-polnaya-karta-atak-na-yazykovyye-modeli), SecAI защита ML (codeby.net/threads/secai-zashchita-ml-i-llm-produktov-ot-prompt-injection-i-data-poisoning)_
