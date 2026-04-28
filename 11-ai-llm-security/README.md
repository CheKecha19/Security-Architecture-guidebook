# AI/LLM Security — Безопасность языковых моделей

## Содержание раздела

| # | Статья | Тема |
|---|--------|------|
| 01 | [AI/LLM Security Overview](01-ai-llm-overview.md) | OWASP Top 10 для LLM, основные угрозы, Defense in Depth |
| 02 | [Prompt Injection](02-prompt-injection.md) | Direct/indirect injection, jailbreaking, adversarial suffix, guardrails |
| 03 | [Training Security](03-training-security.md) | Data poisoning, backdoors, triggers, differential privacy |
| 04 | [Supply Chain Security](04-supply-chain-security.md) | Pickle risks, SafeTensors, Hugging Face safety, model provenance |
| 05 | [Inference Security](05-inference-security.md) | Model extraction, DoS, membership inference, API guardrails |
| 06 | [RAG System Security](06-rag-security.md) | Indirect injection, retrieval poisoning, RBAC, context overflow |
| 07 | [Data Privacy](07-data-privacy.md) | PII leakage, training data extraction, GDPR/152-ФЗ, federated learning |
| 08 | [Authorization & Governance](08-governance.md) | RBAC, Policy as Code, audit logging, model lifecycle, risk assessment |
| 09 | [Enterprise Architecture](09-enterprise-architecture.md) | 10-layer defense, Zero Trust, DLP, SIEM integration, incident response |
| 10 | [AI Security Overview](10-ai-security-overview.md) | Полное руководство, чек-лист, инструменты, архитектура, тренды |

## Ключевые темы

- **Угрозы**: prompt injection, data poisoning, model theft, supply chain, DoS, data leakage
- **Защита**: guardrails (NeMo, Guardrails AI), LLM Firewall (TitanML), adversarial training, DP
- **Supply Chain**: SafeTensors vs Pickle, Hugging Face risks, model provenance, SBOM для AI
- **Приватность**: PII scrubbing, differential privacy, federated learning, membership inference
- **Governance**: RBAC, Policy as Code (OPA), audit logging, model lifecycle, risk assessment
- **Enterprise**: Defense in Depth, Zero Trust, SIEM/DLP integration, incident response
- **Compliance**: GDPR, 152-ФЗ, CCPA, AI Act, NIST AI RMF, SOC 2
- **Инструменты**: Garak, Counterfit, Presidio, NeMo Guardrails, WhyLabs

## Целевая аудитория

- Security Architects (интеграция AI в существующий security stack)
- ML Engineers (безопасное развёртывание моделей)
- Compliance Officers (GDPR, 152-ФЗ, AI Act)
- Red Teamers (AI-атаки и защита)
- InfoSec специалисты, переходящие в AI Security

## Уровень

Intermediate — Advanced. Предполагается знание основ ИБ и базовое понимание LLM.
