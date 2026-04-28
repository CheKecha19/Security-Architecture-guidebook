# Multi-cloud и гибридное облако: безопасность

## Что такое multi-cloud?

Представь, что у тебя есть деньги в нескольких банках: один — для зарплаты, другой — для сбережений, третий — для инвестиций. Каждый банк — хороший. Но у каждого — свои правила, свои карты, своё приложение.

**Multi-cloud** — это именно такой подход. Компания использует несколько облачных провайдеров: AWS для compute, Azure для AI, GCP для analytics. Не кладёт все яйца в одну корзину.

**Гибридное облако** — это когда у тебя есть и свой сейф дома (on-premise), и деньги в банке (облако). Некоторые данные — дома, некоторые — в банке.

## Зачем нужна безопасность multi-cloud?

### Ситуация 1: Управление доступом

Сотрудник имеет доступ к AWS и Azure. Уволился. Нужно отключить везде. Но процессы разные. Консоли разные. Можно забыть.

### Ситуация 2: Консистентность политик

В AWS — одни правила фаервола. В Azure — другие. В GCP — третьи. Как не запутаться?

### Ситуация 3: Visibility

Где мои данные? В AWS S3? Azure Blob? GCP Storage? На-prem? Кто имеет доступ? Что защищено?

## Вызовы multi-cloud

### Identity

| Проблема | Решение |
|----------|---------|
| Разные IAM | SSO, SAML, OIDC |
| MFA везде | Centralized MFA |
| Управление жизненным циклом | Identity governance |

### Network

| Проблема | Решение |
|----------|---------|
| Разные VPC | VPN, Direct Connect, ExpressRoute |
| Маршрутизация | SD-WAN |
| Firewall | Centralized firewall |

### Data

| Проблема | Решение |
|----------|---------|
| Разные форматы | ETL, standardization |
| Шифрование | Customer-managed keys |
| Backup | Cross-cloud backup |

### Monitoring

| Проблема | Решение |
|----------|---------|
| Разные логи | Centralized SIEM |
| Разные alerts | Unified alerting |
| Correlation | Cross-cloud correlation |

## Архитектурные паттерны

### Cloud-agnostic security

| Уровень | Подход |
|---------|--------|
| IAM | Okta, Azure AD, Google Workspace |
| Network | SD-WAN, SASE |
| Data | Customer-managed keys |
| Monitoring | Splunk, Elastic, Datadog |
| Policy | Terraform, OPA |

### SASE (Secure Access Service Edge)

| Компонент | Описание |
|-----------|----------|
| SD-WAN | Software-defined WAN |
| CASB | Cloud Access Security Broker |
| SWG | Secure Web Gateway |
| ZTNA | Zero Trust Network Access |
| FWaaS | Firewall as a Service |

### CSPM для multi-cloud

| Инструмент | Облака |
|------------|--------|
| Prisma Cloud | AWS, Azure, GCP, on-prem |
| Wiz | AWS, Azure, GCP |
| Orca | AWS, Azure, GCP |
| Lacework | AWS, Azure, GCP, Kubernetes |

## Гибридное облако

### Сценарии

| Сценарий | Описание |
|----------|----------|
| Burst | Пиковые нагрузки в облако |
| Backup | On-prem + облачный backup |
| DR | Disaster recovery в облаке |
| Legacy | Старое on-prem, новое в облаке |
| Data residency | Чувствительные данные on-prem |

### Безопасность гибрида

| Компонент | Решение |
|-----------|---------|
| Соединение | VPN, Direct Connect |
| Identity | Federated identity |
| Monitoring | Единый SIEM |
| Policy | Consistent policies |
| Compliance | Unified compliance |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Unified IAM | Единая точка входа |
| Consistent policies | Одинаковые правила |
| Centralized logging | Все логи в одном месте |
| Cross-cloud backup | Backup в другое облако |
| Automation | Terraform, Ansible |
| Visibility | Inventory всего |
| Least privilege | Минимум прав везде |

## Чек-лист понимания

- [ ] Что такое multi-cloud?
- [ ] Что такое гибридное облако?
- [ ] Какие вызовы multi-cloud?
- [ ] Что такое SASE?
- [ ] Какие инструменты CSPM?
- [ ] Что такое CASB?
| [ ] Как соединить on-prem и облако?
- [ ] Что такое cloud-agnostic?
- [ ] Какие лучшие практики?
- [ ] Почему unified IAM важен?

### Ответы на чек-лист

1. **Multi-cloud** — использование нескольких облачных провайдеров.

2. **Гибридное облако** — комбинация on-premise и облака.

3. **Вызовы**: управление доступом, консистентность политик, visibility, identity, network, data, monitoring.

4. **SASE** — Secure Access Service Edge. Объединяет сеть и безопасность: SD-WAN, CASB, SWG, ZTNA, FWaaS.

5. **CSPM**: Prisma Cloud, Wiz, Orca, Lacework.

6. **CASB** — Cloud Access Security Broker. Посредник между пользователем и облаком. Контроль доступа, DLP, threat protection.

7. **Соединение**: VPN, Direct Connect (AWS), ExpressRoute (Azure), Cloud Interconnect (GCP).

8. **Cloud-agnostic** — не зависит от облака. Работает везде одинаково.

9. **Лучшие практики**: unified IAM, consistent policies, centralized logging, cross-cloud backup, automation, visibility, least privilege.

10. **Unified IAM важен**, потому что сотрудник может иметь доступ в несколько облак. Управление жизненным циклом критично.

---

_Статья создана на основе анализа материалов Habr по multi-cloud_
