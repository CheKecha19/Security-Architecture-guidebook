# Методологии threat modeling и оценки рисков

## Описание раздела

Методологии — это инструменты мышления для анализа угроз и оценки рисков. Они помогают системно подходить к безопасности, не упускать важные аспекты и принимать обоснованные решения. Раздел покрывает все основные методологии: от классических (STRIDE, DREAD) до современных (MITRE ATT&CK, PASTA) и специализированных (LINDDUN для приватности, Diamond Model для threat intelligence).

## Ключевые темы

| # | Тема | Описание |
|---|------|----------|
| 01 | **STRIDE** | Классификация угроз Microsoft: Spoofing, Tampering, Repudiation, Information Disclosure, DoS, Elevation |
| 02 | **DREAD** | Количественная оценка рисков: Damage, Reproducibility, Exploitability, Affected Users, Discoverability |
| 03 | **PASTA** | Risk-centric подход: 7 этапов от бизнес-анализа до стоимости митигации |
| 04 | **Cyber Kill Chain** | 7 этапов целевой атаки: от разведки до достижения цели |
| 05 | **Attack Trees** | Иерархические деревья атак с AND/OR узлами и количественной оценкой |
| 06 | **MITRE ATT&CK** | Матрица тактик и техник — стандарт описания adversary-поведения |
| 07 | **OCTAVE** | Операционно-ориентированный подход: профили активов, уязвимости, стратегия защиты |
| 08 | **CVSS** | Система оценки серьёзности уязвимостей: 0-10, базовая/временная/контекстная метрики |
| 09 | **LINDDUN** | Privacy-focused методология: 7 категорий угроз приватности (GDPR/152-ФЗ) |
| 10 | **OWASP Threat Modeling** | 4 шага: scope, DFD, идентификация угроз, митигация |
| 11 | **VAST** | Visual, Agile, Simple Threat modeling — для agile-команд и DevOps |
| 12 | **Trike** | Risk-based подход: акторы, активы, действия, quantitative risk scoring |
| 13 | **Diamond Model** | 4 вершины анализа: adversary, infrastructure, capability, victim |
| 14 | **NIST SP 800-154** | Data-Centric Threat Modeling для федеральных систем США |

## Файлы

- [checklist.md](checklist.md) — 30 вопросов по всем темам раздела с ответами
- [links.md](links.md) — Использованные источники и материалы
- [summary.md](summary.md) — Краткая выжимка знаний (2500-3000 токенов)

## Стандарты и контроли

| Стандарт | Применимые разделы |
|----------|-------------------|
| NIST SP 800-30 | Risk Assessment Framework |
| NIST SP 800-53 | SA-11, SA-15, RA-3 |
| NIST SP 800-154 | Data-Centric Threat Modeling |
| ISO 27005 | Information Security Risk Management |
| OWASP Threat Modeling | Application Threat Modeling Guide |
| CIS Controls v8 | Control 1 (Inventory), Control 18 (Penetration Testing) |

---
