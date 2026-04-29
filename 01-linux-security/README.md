# Linux Security

## Описание раздела

Linux — основа большинства серверных инфраструктур. Понимание механизмов безопасности Linux критично для:
- Настройки безопасных серверов
- Контейнерной безопасности (контейнеры используют namespaces, capabilities, seccomp)
- Compliance (CIS Benchmarks, NIST SP 800-53, ISO 27001)
- Troubleshooting инцидентов и forensic-анализа
- Подготовки к собеседованиям на позиции Security Architect / DevSecOps Engineer

## Ключевые темы

| # | Тема | Описание |
|---|------|----------|
| 01 | **Модель безопасности Linux** | DAC, MAC, capabilities, namespaces — фундамент |
| 02 | **AppArmor** | Profile-based MAC: профили безопасности и принудительный контроль доступа |
| 03 | **SELinux** | Label-based MAC: контексты безопасности, политики, MLS/MCS |
| 04 | **Namespaces и Seccomp** | Изоляция процессов и фильтрация системных вызовов |
| 05 | **POSIX ACL** | Расширенные списки контроля доступа |
| 06 | **Linux Capabilities** | Детализация привилегий root |
| 07 | **PAM** | Модульная аутентификация в Linux |
| 08 | **chroot** | Изоляция файловой системы и псевдоконтейнеризация |
| 09 | **Linux Audit** | Мониторинг системных событий: auditd, правила, forensic-анализ |
| 10 | **Обзор безопасности Linux** | Defense-in-Depth, стратегия защиты |

## Файлы

- [checklist.md](checklist.md) — 30 вопросов по всем темам раздела с ответами
- [links.md](links.md) — Использованные источники и материалы
- [summary.md](summary.md) — Краткая выжимка знаний из всех статей

## Стандарты и контроли

| Стандарт | Применимые контроли |
|----------|---------------------|
| NIST SP 800-53 | AC-3, AC-6, AU-2–AU-12, CM-7, SC-7, SC-39 |
| CIS Controls v8 | 3 (Data Protection), 4 (Secure Configuration), 6 (Access Control Management), 8 (Audit Log Management) |
| ISO 27001 Annex A | A.9.1, A.9.2, A.9.4, A.12.4, A.12.7, A.14.2 |
| CIS Benchmarks | Уровень 1 (Server), Уровень 2 (Workstation) |

---
