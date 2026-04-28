# Security Architecture Guidebook

> Комплексная база знаний по информационной безопасности для подготовки к собеседованиям на позицию **Security Architect**

## Структура репозитория

| # | Раздел | Статей | Статус |
|---|--------|--------|--------|
| 01 | [Linux Security](01-linux-security/) | 10 | ✅ Готово |
| 02 | [CI/CD Security](02-cicd-security/) | 10 | ✅ Готово |
| 03 | [Methodologies](03-methodologies/) | 14 | ✅ Готово |
| 04 | [Standards](04-standards/) | 10 | ✅ Готово |
| 05 | [Cloud Security](05-cloud-security/) | 10 | ✅ Готово |
| 06 | [Java Security](06-java-security/) | 10 | ✅ Готово |
| 07 | [Kafka Security](07-kafka-security/) | 10 | ✅ Готово |
| 08 | [PostgreSQL Security](08-postgresql-security/) | 10 | ✅ Готово |
| 09 | [OAuth/OIDC](09-oauth-oidc/) | 10 | ✅ Готово |
| 10 | [Docker Security](10-docker-security/) | 10 | ✅ Готово |
| 11 | [AI/LLM Security](11-ai-llm-security/) | 10 | ✅ Готово |

**Всего: 114 статей по 11 разделам**

## Формат статей

Все статьи следуют единому шаблону (PROMPT.md v2.2):
- **Аналогии** — сложные концепции объясняются через бытовые примеры
- **Без кода** — только описательные упоминания инструментов
- **Таблицы сравнения** — минимум 2 таблицы на статью
- **Чек-листы** — 10 ситуационных вопросов + 10 развёрнутых ответов
- **Ссылки на стандарты** — NIST, CIS, OWASP, ISO, GDPR, 152-ФЗ
- **5000+ токенов** — глубокий анализ компромиссов и edge cases

## Ключевые темы

### Технические
- **Linux Security:** Landlock, AppArmor, SELinux, Namespaces, Seccomp, POSIX ACL, Capabilities, PAM, chroot, PolicyKit
- **CI/CD Security:** SAST, DAST, SCA, Secrets Management, Supply Chain, Pipeline Security, DevSecOps, IaC, Container Security
- **Java Security:** Spring Security, OWASP Java, AuthN/AuthZ, Session Management, Input Validation, Secure Coding, Crypto, Dependencies
- **Kafka Security:** SSL/TLS, SASL, ACL, Quotas, Encryption, Monitoring
- **PostgreSQL Security:** RBAC, Row-Level Security, Encryption, Audit, Replication Security
- **Docker Security:** Namespaces, cgroups, Seccomp, AppArmor, Image Scanning, Registry Security, Orchestration
- **AI/LLM Security:** OWASP Top 10 for LLM, Prompt Injection, Training Poisoning, Supply Chain, RAG Security, GDPR/152-ФЗ, Governance, Enterprise Architecture

### Методологии и стандарты
- **Threat Modeling:** STRIDE, DREAD, PASTA, Attack Trees, MITRE ATT&CK, VAST, Trike, Diamond Model, NIST SP 800-154
- **Standards:** ISO 27001/27002, NIST CSF, CIS Controls, PCI-DSS, SOC 2, GDPR, FedRAMP, COBIT, ITIL
- **Cloud Security:** Shared Responsibility, IAM, VPC, Encryption, Monitoring, Containers, Serverless, DLP, Multi-cloud
- **Identity:** OAuth 2.0, OIDC, PKCE, JWT, JWS/JWE, SAML, LDAP, SCIM, WebAuthn

## Для кого это руководство

- **Security Architect** — подготовка к собеседованиям и систематизация знаний
- **Security Engineer** — углубление в архитектурные решения
- **DevSecOps** — понимание security в контексте CI/CD и облаков
- **Team Lead / CTO** — обзор ландшафта InfoSec для принятия решений

## Лицензия

MIT License — свободное использование с указанием авторства.

---

*Репозиторий создан при участии AI-ассистента. Все материалы прошли ревью и соответствуют актуальным стандартам безопасности.*
