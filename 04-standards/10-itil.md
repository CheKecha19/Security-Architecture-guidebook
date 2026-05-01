# ITIL 4 и ISO 20000: Управление ИТ-услугами

## Часть 1: Что такое ITIL?

Представь ресторан. В ресторане есть кухня, официанты, меню, заказы, доставка, жалобы. Всё должно работать слаженно: клиент заказал — официант передал — кухня приготовила — принесли — клиент доволен. Если клиент недоволен — менеджер решает проблему.

**ITIL** — это именно такой «ресторан» для ИТ. Не про кухню и поваров, а про то, как предоставлять ИТ-услуги. Как принять заявку, как решить проблему, как внедрить изменение, как управлять инцидентом.

ITIL расшифровывается как Information Technology Infrastructure Library. Разработан британским правительством (OGC — Office of Government Commerce). Текущая версия — ITIL 4 (2019).

**ISO 20000** — международный стандарт управления ИТ-услугами. Основан на ITIL, но формализован. Можно сертифицировать организацию.

## Часть 2: Зачем нужны ITIL и ISO 20000?

### Ситуация 1: Хаос в ИТ-поддержке

Сотрудник написал: «Не работает принтер». Заявка потерялась в почте. Через неделю напомнил. Оказалось — забыли. Принтер всё ещё не работает.

ITIL вводит процессы: заявка → классификация → назначение → решение → закрытие. Ничего не теряется. SLA: 4 часа на решение.

### Ситуация 2: Изменения ломают систему

Админ обновил сервер в пятницу вечером. Система упала. Никто не знал, что обновили. Не было процесса согласования.

ITIL: Change Management. RFC (Request for Change) → оценка → CAB (Change Advisory Board) approval → тест → внедрение в окно обслуживания. Контролируемо.

### Ситуация 3: Доказательство качества

Компания хочет показать клиентам: «Мы профессионально управляем ИТ». ISO 20000 — сертификат от аккредитованного органа.

### Ситуация 4: Интеграция с безопасностью

Инцидент безопасности (взлом) — это не просто «проблема», а **Security Incident**. ITIL определяет: как классифицировать, куда эскалировать, какие SLA.

## Часть 3: ITIL 4 — четыре измерения

ITIL 4 смотрит на ИТ-услуги через 4 измерения:

| Измерение | Описание | Пример | Влияние на безопасность |
|-----------|----------|--------|------------------------|
| **Организации и люди** | Культура, навыки, коммуникация | Команда поддержки, обучение | Security awareness, training |
| **Информация и технологии** | Информация, знания, сервисы | База знаний, CMDB, SIEM | Data classification, encryption |
| **Партнёры и поставщики** | Отношения с внешними | Аутсорсинг, облако, MSP | Third-party risk, contracts |
| **Потоки ценности** | Процессы, workflow | Incident management pipeline | Security incident handling |

## Часть 4: Service Value System (SVS)

```
                    [Входы: возможности, потребности]
                              |
                              v
                    [Guiding Principles] ←── [Governance]
                              |                  |
                              v                  |
                    [Service Value Chain] ──────┘
                              |
                    ┌─────────┼─────────┐
                    |         |         |
                    v         v         v
               [Practices] [Processes] [Roles]
                              |
                              v
                    [Выходы: ценность]
```

### Guiding Principles (Руководящие принципы)

| Принцип | Описание | Применение в ИБ |
|---------|----------|----------------|
| Focus on value | Фокус на ценности | ИБ-сервисы должны помогать бизнесу |
| Start where you are | Начинай с текущего | Не «с нуля», а улучшай |
| Progress iteratively | Прогресс итеративно | Incremental security improvements |
| Collaborate and promote visibility | Сотрудничай | Security != отдельный остров |
| Think and work holistically | Мысли системно | ИБ — часть всей системы |
| Keep it simple and practical | Простота | Не перегружай процессы |
| Optimize and automate | Оптимизируй и автоматизируй | SOAR, automated response |

### Service Value Chain (Цепочка создания ценности)

| Активность | Описание | ИБ-пример |
|------------|----------|-----------|
| Plan | Планирование | Security planning, risk assessment |
| Improve | Улучшение | Security improvements, patches |
| Engage | Вовлечение | Security communication, training |
| Design & Transition | Проектирование и переход | Secure design, change management |
| Obtain/Build | Получение/строительство | Security tools, hardening |
| Deliver & Support | Доставка и поддержка | Incident response, monitoring |

## Часть 5: Practices (Практики)

ITIL 4 определяет 34 практики:

### General Management Practices (14)

| Практика | Описание | ИБ-аспект |
|----------|----------|-----------|
| Architecture Management | Управление архитектурой | Security architecture |
| Continual Improvement | Непрерывное улучшение | Security metrics, improvement |
| Information Security Management | Управление ИБ | **Центральная практика** |
| Knowledge Management | Управление знаниями | Security runbook, playbooks |
| Measurement and Reporting | Измерение | Security KPI, SLI |
| Organizational Change Management | Управление изменениями | Security changes |
| Portfolio Management | Управление портфелем | Security projects |
| Project Management | Управление проектами | Security implementations |
| Relationship Management | Управление отношениями | Vendor security |
| Risk Management | Управление рисками | Security risk assessment |
| Service Financial Management | Финансовый менеджмент | Security budget |
| Strategy Management | Управление стратегией | Security strategy |
| Supplier Management | Управление поставщиками | Third-party security |
| Workforce and Talent Management | Управление персоналом | Security skills |

### Service Management Practices (17)

| Практика | Описание | ИБ-аспект |
|----------|----------|-----------|
| Availability Management | Управление доступностью | DoS protection |
| Capacity and Performance | Производительность | Resource exhaustion attacks |
| Change Control | Управление изменениями | Security review |
| Incident Management | Управление инцидентами | **Security incidents** |
| IT Asset Management | Управление активами | Asset inventory |
| Monitoring and Event Management | Мониторинг | SIEM, SOC |
| Problem Management | Управление проблемами | Root cause analysis |
| Release Management | Управление релизами | Secure deployment |
| Service Catalogue | Каталог услуг | Security services |
| Service Configuration | Конфигурация | CMDB, hardening |
| Service Continuity | Непрерывность | DR, backup |
| Service Design | Проектирование | Secure by design |
| Service Desk | Служба поддержки | Security hotline |
| Service Level Management | SLA | Security SLA |
| Service Request Management | Запросы | Access requests |
| Service Validation and Testing | Тестирование | Security testing |
| Deployment Management | Развёртывание | Secure deployment |

### Technical Management Practices (3)

| Практика | Описание | ИБ-аспект |
|----------|----------|-----------|
| Deployment Management | Развёртывание | Secure CI/CD |
| Infrastructure and Platform Management | Инфраструктура | Hardening, patching |
| Software Development and Management | Разработка | SDLC, DevSecOps |

## Часть 6: ISO 20000

### Структура ISO 20000-1:2018

| Раздел | Описание |
|--------|----------|
| 4.1 | Understanding the organization and its context | Контекст, включая ИБ |
| 4.2 | Understanding the needs and expectations | Требования, включая compliance |
| 5.1 | Leadership and commitment | Лидерство в ИТ-услугах |
| 6.1 | Actions to address risks and opportunities | Risk management |
| 6.2 | Service management objectives | Цели, включая ИБ-цели |
| 7.1 | Resources | Ресурсы для ИБ |
| 7.2 | Competence | Компетенции (security skills) |
| 7.5 | Documented information | Документация (политики, процедуры) |
| 8.2 | Determination of requirements | Требования к услугам |
| 8.3 | Design and development | Проектирование (secure by design) |
| 8.5.1 | Change management | Управление изменениями |
| 8.6.1 | Incident management | Управление инцидентами |
| 8.6.3 | Problem management | Управление проблемами |
| 8.7.1 | Service availability | Доступность (DoS protection) |
| 9.1 | Monitoring, measurement, analysis | Мониторинг (SIEM) |
| 9.2 | Internal audit | Внутренний аудит |
| 9.3 | Management review | Обзор руководством |
| 10.1 | Nonconformity and corrective action | Корректирующие действия |

### ISO 20000 и ISO 27001

| ISO 20000 | ISO 27001 | Пересечение |
|-----------|-----------|-------------|
| Управление ИТ-услугами | Управление ИБ | Risk management, Incident management |
| Service continuity | Business continuity | DR, backup |
| Change management | Change management | Security review |
| Supplier management | Supplier relationships | Third-party security |

**Можно интегрировать:** ISO 20000 + ISO 27001 = полное покрытие ИТ + ИБ.

## Часть 7: Security Incident Management в ITIL

### Процесс Incident Management (с ИБ-фокусом)

```
[Detection] → [Logging] → [Categorization] → [Prioritization] → [Initial Diagnosis]
                                                                      |
                    [Major Incident?] ──ДА──→ [Major Incident Procedure]
                          |
                         НЕТ
                          |
                    [Investigation & Diagnosis] → [Resolution] → [Recovery] → [Closure]
                                                                              |
                                                                       [Review]
```

### Классификация инцидентов

| Тип | Описание | Примеры | Приоритет |
|-----|----------|---------|-----------|
| Network incident | Проблема с сетью | Отказ маршрутизатора | P2 |
| Server incident | Проблема с сервером | Перегрузка CPU | P2 |
| Application incident | Проблема приложения | Ошибка 500 | P3 |
| **Security incident** | **Угроза безопасности** | **Взлом, malware, утечка** | **P1** |
| Service request | Запрос услуги | Новый доступ, пароль | P4 |

### SLA для Security Incidents

| Приоритет | Время реакции | Время решения |
|-----------|--------------|---------------|
| P1 — Critical | 15 минут | 4 часа |
| P2 — High | 30 минут | 8 часов |
| P3 — Medium | 2 часа | 24 часа |
| P4 — Low | 4 часа | 72 часа |

## Вывод

ITIL 4 + ISO 20000:
1. **Фокус** — создание ценности через ИТ-услуги
2. **4 измерения** — организация, информация, партнёры, потоки ценности
3. **34 практики** — процессы для управления ИТ
4. **SVS** — система создания ценности
5. **ISO 20000** — формализация и сертификация
6. **ИБ-интеграция** — Security Management как практика, Security Incidents как приоритет

ITIL — язык бизнеса для ИТ. ISO 20000 — доказательство качества.

---
_Статья создана на основе анализа материалов по стандартам безопасности_