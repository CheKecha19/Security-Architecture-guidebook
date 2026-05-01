# COBIT 2019: Управление ИТ и информационной безопасностью

## Часть 1: Что такое COBIT?

Представь большую компанию. В ней много отделов: ИТ, бухгалтерия, HR, продажи, производство. Каждый отдел использует технологии. Каждый хочет что-то своё. ИТ-отдел хочет новые серверы. Бухгалтерия — новую программу. Продажи — мобильное приложение.

Кто решает, что важнее? Кто следит, чтобы деньги на ИТ тратились правильно? Кто проверяет, что всё это работает на бизнес?

**COBIT** — это фреймворк, который помогает управлять ИТ как бизнес-активом. Не просто «у нас есть компьютеры», а «ИТ помогает бизнесу достигать целей».

COBIT расшифровывается как Control Objectives for Information and Related Technologies. Разработан ISACA (Information Systems Audit and Control Association). Текущая версия — COBIT 2019.

## Часть 2: Зачем нужен COBIT?

### Ситуация 1: ИТ и бизнес говорят на разных языках

Бизнес говорит: «Нам нужно увеличить продажи на 20%». ИТ отвечает: «Нам нужен новый сервер с 128 ядрами». Они не понимают друг друга.

COBIT переводит: «Новый сервер поможет обработать больше заказов → увеличит продажи → ROI 150% за год».

### Ситуация 2: Хаос в ИТ

Проекты проваливаются. Бюджеты превышаются на 300%. Системы падают каждую неделю. Никто не знает, кто за что отвечает.

COBIT вводит порядок: процессы, роли, метрики, зрелость.

### Ситуация 3: Аудит

Аудитор спрашивает: «Как вы управляете ИТ-рисками?» Без COBIT — расплывчатый ответ. С COBIT — конкретные процессы, контроли, доказательства.

### Ситуация 4: Цифровая трансформация

Компания переходит на облако, DevOps, микросервисы. COBIT помогает управлять изменениями без хаоса.

## Часть 3: Структура COBIT 2019

### Гovernance Framework (Рамки управления)

COBIT 2019 состоит из:

| Компонент | Описание | Количество |
|-----------|----------|------------|
| Domains (Области) | Группы процессов | 5 |
| Governance Objectives (Цели управления) | Чего достичь | 5 |
| Management Objectives (Цели управления) | Как достичь | 32 |
| Components (Компоненты) | Элементы системы управления | 7 |
| Design Factors (Факторы дизайна) | Контекст организации | 11 |

### 5 Domains (Областей)

| Domain | Описание | Процессы |
|--------|----------|----------|
| EDM — Evaluate, Direct and Monitor | Оценивать, направлять, мониторить | 5 |
| APO — Align, Plan and Organize | Согласовывать, планировать, организовывать | 14 |
| BAI — Build, Acquire and Implement | Строить, приобретать, внедрять | 10 |
| DSS — Deliver, Service and Support | Доставлять, обслуживать, поддерживать | 6 |
| MEA — Monitor, Evaluate and Assess | Мониторить, оценивать, анализировать | 6 |

### 7 Components (Компонентов системы управления)

| Component | Описание | Примеры |
|-----------|----------|---------|
| Processes | Процессы | Incident management, Change management |
| Organizational Structures | Структуры | Роли, комитеты, RACI |
| Information Flows | Потоки информации | Отчёты, KPI, dashboards |
| Culture and Behavior | Культура и поведение | Ценности, awareness |
| Skills and Competencies | Навыки | Обучение, сертификации |
| Policies and Procedures | Политики | ИБ-политика, доступ |
| Information and Data | Информация и данные | CMDB, asset inventory |

## Часть 4: Governance vs Management

COBIT различает **Governance** (управление на уровне совета директоров) и **Management** (операционное управление):

| | Governance (EDM) | Management (APO, BAI, DSS, MEA) |
|---|------------------|--------------------------------|
| **Кто** | Совет директоров, C-level | CIO, CTO, менеджеры |
| **Что** | Цели, ценности, риски | Реализация, операции |
| **Вопросы** | «Правильные ли цели?» | «Как достичь целей?» |
| **Процессы** | EDM01-EDM05 | APO01-APO14, BAI01-BAI10, DSS01-DSS06, MEA01-MEA06 |

### Governance Objectives

| ID | Цель | Описание |
|----|------|----------|
| EDM01 | Ensured Governance Framework | Управление фреймворком |
| EDM02 | Ensured Value Delivery | Доставка ценности |
| EDM03 | Ensured Risk Optimization | Оптимизация рисков |
| EDM04 | Ensured Resource Optimization | Оптимизация ресурсов |
| EDM05 | Ensured Stakeholder Engagement | Вовлечение stakeholders |

### Management Objectives (примеры)

| ID | Цель | Область |
|----|------|---------|
| APO01 | Managed I&T Management Framework | Управление фреймворком |
| APO12 | Managed Risk | Управление рисками |
| APO13 | Managed Security | Управление безопасностью |
| APO14 | Managed Data | Управление данными |
| BAI01 | Managed Programs | Управление программами |
| BAI06 | Managed Changes | Управление изменениями |
| DSS01 | Managed Operations | Управление операциями |
| DSS05 | Managed Security Services | Управление ИБ-сервисами |
| MEA01 | Managed Performance | Управление performance |

## Часть 5: COBIT и информационная безопасность

### APO13 — Managed Security

| Практика | Описание | Примеры |
|----------|----------|---------|
| APO13.01 | Establish and maintain security governance | ИБ-политика, комитет |
| APO13.02 | Define and maintain a security risk profile | Профиль рисков |
| APO13.03 | Establish and maintain security architecture | Архитектура безопасности |
| APO13.04 | Ensure security compliance | Комплаенс |
| APO13.05 | Ensure security monitoring | Мониторинг |

### DSS05 — Managed Security Services

| Практика | Описание |
|----------|----------|
| DSS05.01 | Protect against malware | Antivirus, EDR |
| DSS05.02 | Manage network security | Firewall, segmentation |
| DSS05.03 | Manage endpoint security | EDR, hardening |
| DSS05.04 | Manage identity and access | IAM, MFA, RBAC |
| DSS05.05 | Manage data security | Шифрование, DLP |
| DSS05.06 | Ensure physical security | Дата-центры, доступ |

### Capability Levels (Уровни зрелости)

| Уровень | Описание | Что это значит |
|---------|----------|----------------|
| 0 — Incomplete | Не выполняется | Процесса нет |
| 1 — Performed | Выполняется | Ad-hoc, непредсказуемо |
| 2 — Managed | Управляется | Планируется, отслеживается |
| 3 — Established | Установлено | Стандартизировано |
| 4 — Predictable | Предсказуемо | Измеряется, контролируется |
| 5 — Optimizing | Оптимизируется | Непрерывное улучшение |

**Цель для ИБ:** минимум уровень 3 (Established) для критичных процессов.

## Часть 6: Внедрение COBIT

### Шаги внедрения

```
1. Understand enterprise context
   └── Факторы: размер, отрасль, регуляторы, risk appetite

2. Determine governance system scope
   └── Какие процессы, какие системы, какие данные

3. Refine governance system design
   └── Выбор компонентов, настройка процессов

4. Implement governance system
   └── Политики, процедуры, инструменты, обучение

5. Evaluate and direct governance system
   └── Аудит, review, непрерывное улучшение
```

### COBIT vs другие фреймворки

| Фреймворк | Фокус | Как использовать с COBIT |
|-----------|-------|--------------------------|
| ISO 27001 | ИБ-менеджмент | APO13 + DSS05 маппятся на ISO 27001 |
| ITIL | ИТ-услуги | DSS маппится на ITIL service management |
| NIST CSF | Киберриски | EDM03 + APO12 + MEA |
| TOGAF | Enterprise architecture | APO04 + BAI03 |
| CMMI | Разработка ПО | BAI процессы |

### Сертификации ISACA

| Сертификация | Для кого | Что подтверждает |
|--------------|----------|-----------------|
| CISA | Аудиторы | Знание аудита ИТ |
| CISM | ИБ-менеджеры | Управление ИБ |
| CGEIT | Governance | ИТ-управление |
| CRISC | Риск-менеджеры | Управление рисками |
| COBIT Foundation | Все | Базовое знание COBIT |

## Вывод

COBIT 2019:
1. **Фреймворк** — не стандарт, а система управления
2. **5 областей** — EDM, APO, BAI, DSS, MEA
3. **32 цели управления** — конкретные процессы
4. **7 компонентов** — процессы, структуры, культура, навыки
5. **6 уровней зрелости** — от Incomplete до Optimizing
6. **ИБ-фокус** — APO13 + DSS05
7. **Интеграция** — ISO 27001, ITIL, NIST CSF

COBIT — мост между бизнесом и ИТ. Помогает говорить на одном языке.

---
_Статья создана на основе анализа материалов по стандартам безопасности_