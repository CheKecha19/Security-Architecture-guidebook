# Enterprise AI Security Architecture: Защита AI-систем в корпоративной среде

## Что такое enterprise AI security?

Представьте, что крупная компания — это город. В городе есть жилые кварталы (отделы), правительство (руководство), банки (финансы), больницы (HR), полиция (безопасность). В город приезжает новый житель — AI-ассистент. Он умён, полезен, но может случайно выдать чужие секреты, выполнить опасные команды или просто занять слишком много ресурсов. Enterprise AI security — это «правила города» для AI: в какие здания он может заходить, какие документы читать, какие действия выполнять, как его контролируют.

Enterprise AI security — это архитектура защиты AI-систем в корпоративной среде. Включает: защиту данных (PII, секреты), защиту моделей (IP, weights), защиту инфраструктуры (GPU, API), защиту пользователей (от вредоносных ответов), комплаенс (GDPR, 152-ФЗ), мониторинг (атаки, аномалии).

В отличие от «просто запустить LLM», enterprise требует: многопользовательской изоляции, интеграции с существующей security-инфраструктурой (SIEM, IAM, DLP), соответствия корпоративным политикам, аудита всего и вся. Enterprise AI security — это не продукт, а архитектура: как AI вписывается в существующую инфраструктуру безопасности.

## Зачем это нужно?

**Сценарий 1: Интеграция с SIEM.** Сбер уже использует Splunk для сбора логов безопасности. AI-ассистент должен отправлять логи в Splunk: input guard triggers, output guard triggers, prompt injection attempts, data leakage events. Security team может увидеть: «Кто-то пытается jailbreak HR-ассистента из IP 10.0.0.55 — заблокировать». AI-безопасность не должна быть отдельным островом — она должна вписываться в существующий security stack.

**Сценарий 2: DLP для LLM.** Корпоративная DLP-система (Data Loss Prevention) должна контролировать ответы AI: не отправляет ли модель конфиденциальные данные пользователям? DLP проверяет: ответ содержит PII → блокировка. Ответ содержит «коммерческая тайна» → блокировка. Ответ содержит внутренние ссылки → блокировка. DLP для LLM — интеграция output guard с корпоративной DLP.

**Сценарий 3: IAM интеграция.** Сбер использует Active Directory для управления пользователями. AI-ассистент должен: аутентифицироваться через AD (SSO), использовать роли AD (user → employee, hr, admin), синхронизировать группы доступа. IAM интеграция — AI не создаёт свою систему пользователей, а использует существующую.

## Основные концепции

### Defense in Depth для LLM

Многослойная защита AI-системы. Каждый слой защищает от определённых угроз.

**Layer 1: Network.** Защита сети: LLM API доступен только через VPN, IP whitelist, WAF (для веб-интерфейсов), DDoS protection. Сетевой уровень — первый барьер.

**Layer 2: IAM/Auth.** Аутентификация и авторизация: SSO (через AD/LDAP), JWT tokens, MFA для администраторов, RBAC. Только авторизованные пользователи могут получить доступ к LLM.

**Layer 3: API Gateway.** Centralized gateway: rate limiting, quota, audit logging, IP blocking, user agent validation. Gateway — единая точка входа со всеми проверками.

**Layer 4: Input Guard.** Проверка запросов: prompt injection detection, длина запроса, вредоносный контент, PII в запросе. Input guard — фильтр на входе.

**Layer 5: RAG Security.** Безопасность retrieval: RBAC на документы, confidence score, source verification, context size limit. RAG — защита документов от утечки.

**Layer 6: LLM.** Модель: fine-tuning на безопасность, adversarial training, DP обучение, подпись модели. Модель — core, который должен быть безопасным.

**Layer 7: Output Guard.** Проверка ответов: PII маскирование, content safety, code scanner, fact check. Output guard — последняя линия перед пользователем.

**Layer 8: DLP.** Data Loss Prevention: обнаружение утечки конфиденциальных данных, блокировка ответов с коммерческой тайной. DLP — интеграция с корпоративной DLP.

**Layer 9: SIEM/Monitoring.** Логирование и мониторинг: все запросы в SIEM, anomaly detection, alerting, dashboard. SIEM — обнаружение атак.

**Layer 10: Human Review.** Periodic review: red teaming, audit логов, review guardrails, update policies. Human review — контроль качества защиты.

### Zero Trust для AI

Zero Trust — «никому не доверяй, всегда проверяй» — для AI-систем.

**Verify explicitly.** Каждый запрос проверяется, независимо от источника. Даже запрос от администратора (с AD) проходит input guard, RBAC, rate limiting. Нет «доверенного источника» — каждый запрос верифицируется.

**Least privilege.** Минимальные права для AI-агента. Модель имеет доступ только к тем данным, которые нужны для её задачи. Не к лишним. Human-in-the-loop для критических действий. AI не может делать то, что не разрешено явно.

**Assume breach.** Считаем, что LLM уже скомпрометирована (через prompt injection). Все ответы фильтруются (output guard). Все запросы логируются. Модель не доверяет контексту из RAG. Output guard — последняя линия.

### AI Security Stack: Компоненты

Enterprise AI Security Stack — набор инструментов для защиты LLM.

**Input/Output Guards:**
- NeMo Guardrails (NVIDIA) — open-source guardrails
- Guardrails AI — ML-based guardrails
- LLM Guard (Protect AI) — security scanner for LLM
- Rebuff — prompt injection detector

**LLM Firewall:**
- TitanML — enterprise LLM firewall
- WhyLabs — AI observability platform
- Arthur AI — LLM monitoring
- AIShield — LLM security testing

**Model Registry:**
- MLflow — open-source ML lifecycle
- Hugging Face Hub — model registry (с ограничениями для enterprise)
- Seldon Core — model deployment
- BentoML — model serving

**Monitoring:**
- ELK Stack — логи
- Prometheus/Grafana — метрики
- Falco — runtime security (может мониторить LLM контейнеры)
- Datadog — observability (платный)

**IAM:**
- Keycloak — open-source IAM
- Auth0 — cloud IAM
- Azure AD / AWS IAM — для облачных LLM

### Data Classification: Классификация данных

Не все данные одинаково чувствительны. Классификация данных — разделение данных на категории по уровню доступа и риска.

**Уровни классификации:**
- **Public** — открытая информация (политика отпусков, бонусная программа). Доступна всем сотрудникам.
- **Internal** — внутренняя информация (справочник должностей). Доступна всем сотрудникам, но не внешним.
- **Confidential** — конфиденциальная информация (зарплаты, оценки). Доступна по роли.
- **Restricted** — ограниченный доступ (планы реорганизации, данные о слияниях). Доступна по отдельному approval.

**Как классификация влияет на LLM:**
- Public — модель может отвечать без ограничений
- Internal — нужна аутентификация, но без RBAC
- Confidential — RBAC, output guard, audit
- Restricted — Human-in-the-loop + approval

### Incident Response для LLM

План реагирования на инциденты с AI-системами.

**Шаги incident response:**
1. **Detection.** Обнаружение: SIEM alert, guardrails trigger, user report, red team finding.
2. **Containment.** Изоляция: временное отключение LLM, блокировка IP, откат модели.
3. **Investigation.** Логи: кто, когда, какой запрос, какой ответ, какие guardrails сработали.
4. **Analysis.** Почему произошло: недостаток guardrails, новый тип атаки, ошибка конфигурации.
5. **Remediation.** Исправление: update guardrails, retrain model, patch infrastructure.
6. **Recovery.** Восстановление: включение LLM, проверка, мониторинг.
7. **Post-mortem.** Анализ: что пошло не так, что улучшить, как предотвратить повторение.

**Специфичные для LLM аспекты IR:**
- Инцидент может быть не единичным, а массовым (один вредоносный документ в RAG — скомпрометированы ответы для тысяч пользователей)
- Инцидент может быть распределённым во времени (backdoor, активированный через 3 месяца после обучения)
- Инцидент может быть неявным (утечка PII без прямого запроса — через memorization)

## Сравнение enterprise AI security vs простого развёртывания

| Аспект | Простое развёртывание | Enterprise |
|--------|-----------------------|------------|
| Auth | API key | SSO + MFA |
| RBAC | Нет | Role + scope + quota |
| Guardrails | Optional | Mandatory (input + output) |
| Audit | Minimal | Full (SIEM integration) |
| RAG | Open access | RBAC + filtering |
| DLP | Нет | Integration |
| Incident Response | Manual | Automated playbook |
| Red teaming | No | Quarterly |
| Compliance | No | GDPR, 152-ФЗ, SOC 2 |

## Практические применения

**Сценарий 1: Enterprise AI stack в Сбере.**
Компоненты: API Gateway (Kong) → Input Guard (NeMo Guardrails) → RAG (Vector DB + RBAC) → LLM (корпоративная модель + guardrails) → Output Guard (Presidio PII) → DLP (интеграция с Symantec DLP) → SIEM (Splunk). Все компоненты — managed, с SLA. Интеграция с AD (SSO) и HashiCorp Vault (secrets).

**Сценарий 2: Incident Response playbook для LLM.** Playbook: SIEM получает alert (prompt injection detected). Автоматически блокируется IP пользователя. Создаётся инцидент в ServiceNow. Дежурный security engineer проверяет логи: был ли успешный injection? Если да — временное отключение LLM, расследование, обновление guardrails. Если false positive — разблокировка, обновление правил. Время реакции — 5 минут.

**Сценарий 3: Red teaming программа.** Ежеквартальный red team: команда security атакует LLM. Prompt injection (100+ техник), jailbreaking, data extraction, membership inference. Результаты: найденные уязвимости, рекомендации по усилению, обновление guardrails. Red team — обязательный процесс для enterprise LLM. Результаты — в backlog security team.

**Сценарий 4: DLP интеграция.**
Output guard LLM отправляет «подозрительные» ответы в DLP-систему. DLP проверяет: содержит ли ответ «коммерческая тайна», «конфиденциально», внутренние ссылки на документы, PII (номера паспортов, зарплаты). Если DLP блокирует — ответ не отправляется, создаётся security event. Интеграция output guard + DLP — двойная фильтрация.

**Сценарий 5: Data classification policy for LLM.** Все документы, загружаемые в RAG-базу, должны иметь метку классификации (public, internal, confidential, restricted). Политика: documents without classification — rejected. Documents with «restricted» — require human approval for RAG access. Автоматическая классификация (ML-модель) + ручная верификация.

## Ограничения и нюансы

**Enterprise AI stack — дорого.** NeMo Guardrails, Kong, Splunk, DLP, SIEM — лицензии на всё это стоят миллионы рублей в год. Не каждая компания может себе позволить полный stack. Минимальный stack: API Gateway (open-source) + Guardrails (open-source) + Audit (ELK open-source). Но minimal — не enterprise.

**Zero Trust для LLM — сложно.** Assume breach означает: каждый ответ проверяется, каждый запрос логируется, каждая операция — с подтверждением. Это добавляет latency, сложность, стоимость. Баланс между безопасностью и user experience — архитектурное решение.

**Incident Response для LLM — новая область.** Нет стандартных playbook для «LLM скомпрометирована через prompt injection». Команды безопасности учатся на ходу. Нужно: создавать playbook, тренировать команду, проводить tabletop exercises.

**Red teaming — специализированная экспертиза.** Не каждый security engineer умеет jailbreak LLM или проводить membership inference. Red teaming требует: знания LLM архитектуры, prompt engineering, ML понимания. Обучение red team — инвестиции.

## Архитектурные решения

**Security as Code.** Все политики безопасности LLM — в коде. Guardrails rules, OPA policies, RBAC конфигурация, DLP правила — в Git. Изменения — pull request + review. Security as Code — версионируемость, тестируемость, аудируемость.

**Integration, not replacement.** LLM безопасность — не замена существующим инструментам. DLP, SIEM, IAM уже есть в enterprise. LLM должна интегрироваться с ними, а не создавать свои. AI security — часть security stack, не отдельный остров.

**Continuous improvement.** Безопасность LLM — не проект с датой завершения. Это непрерывный процесс: новые атаки → новые guardrails → новый red team → обновление политик. Security — марафон.

**Layered defense.** Ни один слой не защищает полностью. Input guard блокирует 80% prompt injection. LLM-as-judge — ещё 10%. Human review — ещё 5%. Остаётся 5% риска — это residual. Enterprise принимает residual или усиливает защиту.

## Чек-лист понимания

- [ ] Какие 10 слоёв Defense in Depth для LLM?
- [ ] Как Zero Trust применяется к AI-системам?
- [ ] Какие компоненты enterprise AI security stack?
- [ ] Как data classification влияет на доступ к LLM?
- [ ] Какие шаги incident response для LLM?
- [ ] Как DLP интегрируется с LLM?
- [ ] Почему enterprise AI security дороже простого развёртывания?
- [ ] Чем red teaming для LLM отличается от традиционного?
- [ ] Почему security as code — best practice?
- [ ] Что такое residual risk для LLM?

### Ответы на чек-лист

1. **Какие 10 слоёв Defense in Depth для LLM?** (1) Network — WAF, IP whitelist, VPN. (2) IAM — SSO, MFA, AD. (3) API Gateway — rate limit, audit. (4) Input Guard — prompt injection detection. (5) RAG Security — RBAC, confidence score. (6) LLM — fine-tuning, adversarial training. (7) Output Guard — PII mask, content safety. (8) DLP — Data Loss Prevention. (9) SIEM — monitoring, alerting. (10) Human Review — red team, audit. Каждый слой — отдельная линия защиты.

2. **Как Zero Trust применяется к AI-системам?** Verify explicitly: каждый запрос проверяется (auth, RBAC, input guard), независимо от источника. Least privilege: минимальные права для AI (только чтение, только нужные документы). Assume breach: считаем, что LLM скомпрометирована, все ответы фильтруются (output guard). Zero Trust для AI — «никогда не доверяй, всегда проверяй».

3. **Какие компоненты enterprise AI security stack?** Input/Output Guards (NeMo, Guardrails AI), LLM Firewall (TitanML, WhyLabs), Model Registry (MLflow), IAM (Keycloak, AD), API Gateway (Kong, Tyk), Monitoring (ELK, Prometheus), DLP (Symantec, McAfee), SIEM (Splunk, ELK), Secrets Management (HashiCorp Vault). Все компоненты интегрируются через API и стандартные протоколы.

4. **Как data classification влияет на доступ к LLM?** Public — модель отвечает без ограничений. Internal — аутентификация, но без RBAC. Confidential — RBAC + output guard + audit. Restricted — Human-in-the-loop + approval. Классификация данных — основа для RBAC и DLP. Если документ не имеет классификации — он не может быть использован RAG.

5. **Какие шаги incident response для LLM?** (1) Detection — SIEM alert, guardrails trigger. (2) Containment — отключение LLM, блокировка IP. (3) Investigation — логи, кто/что/когда. (4) Analysis — почему произошло, какой вектор атаки. (5) Remediation — update guardrails, patch. (6) Recovery — включение LLM, мониторинг. (7) Post-mortem — что улучшить. Все шаги — автоматизированы (playbook) для быстрой реакции.

6. **Как DLP интегрируется с LLM?** Output guard LLM отправляет «подозрительные» ответы в корпоративную DLP. DLP проверяет: PII, коммерческая тайна, конфиденциальные паттерны. Если DLP блокирует — ответ не отправляется пользователю. Интеграция: стандартный API (REST), syslog, SIEM. DLP для LLM — та же DLP, что для email и файлов, адаптированная под AI-ответы.

7. **Почему enterprise AI security дороже простого развёртывания?** Enterprise требует: инфраструктуру (API Gateway, SIEM, DLP), лицензии (коммерческий SIEM и DLP — миллионы), экспертизу (security engineers, red team), compliance (аудит, документация), мониторинг (24/7). Простое развёртывание — API key + LLM. Enterprise — десятки компонентов. Разница в стоимости — 10-100x.

8. **Чем red teaming для LLM отличается от традиционного?** Традиционный red team: find vulnerabilities in code, network, infrastructure. LLM red team: find prompt injection, jailbreaking, data extraction, membership inference. LLM red team должен знать: LLM архитектуру, prompt engineering, ML. Традиционные инструменты (Nmap, Burp Suite) — не работают для LLM. Нужны специализированные: Garak (LLM vulnerability scanner), Counterfit (AI security testing).

9. **Почему security as code — best practice?** Security as Code — политики безопасности в Git. Преимущества: версионирование (история изменений), code review (качество политик), testing (unit tests для OPA/Rego), automation (CI/CD — изменение политики → деплой), audit (кто, когда, что изменил). Security as Code — стандарт для enterprise.

10. **Что такое residual risk для LLM?** Residual risk — остаточный риск после применения всех мер защиты. Пример: prompt injection — guardrails блокируют 90% атак. Остаётся 10% атак, которые guardrails не обнаруживают. Residual risk = 10%. Enterprise принимает этот риск (уровень acceptable risk) или усиливает защиту (LLM-as-judge, human review). Residual risk документируется в risk assessment.

---

_Статья создана на основе анализа материалов: OWASP Top 10 для LLM (habr.com/ru/companies/bastion/articles/960918), LLM Firewall (habr.com/ru/articles/981408), Guardrails для LLM (habr.com/ru/articles/1024028), AI Governance (habr.com/ru/companies/ru_mts/articles/841010), SecAI защита (codeby.net/threads/secai-zashchita-ml-i-llm-produktov-ot-prompt-injection-i-data-poisoning)_
