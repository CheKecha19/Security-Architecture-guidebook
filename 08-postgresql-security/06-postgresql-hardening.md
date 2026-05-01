# Hardening PostgreSQL: усиление защиты по умолчанию

## Что такое харденинг?

Представь замок. Не просто дверь с засовом, а многослойная защита: ров с водой, подъёмный мост, внешняя стена, башни, внутренний двор, гарнизон. Каждый уровень задерживает врага.

**Харденинг PostgreSQL** — это процесс усиления защиты базы данных: отключение ненужного, включение необходимого, минимизация поверхности атаки. Не «установил и забыл», а «установил и усилил».

## Эволюция и мотивация

PostgreSQL из коробки настроена на максимальную совместимость и минимальные препятствия. Это означает: trust для локальных подключений, слушать только localhost, пароли — md5 или trust, superuser без ограничений.

Но после установки администратор часто меняет: `listen_addresses = '*'` (чтобы приложения подключались), добавляет `host all all 0.0.0.0/0 md5` (чтобы было проще), не меняет пароль `postgres`.

Результат: база данных открыта миру. Брутфорс паролей, SQL Injection, privilege escalation — всё возможно.

**CIS Benchmarks** (Center for Internet Security) — стандарт харденинга. Для PostgreSQL — ~100 пунктов проверки. Версии: PostgreSQL 12 CIS Benchmark, 13, 14.

## Зачем это нужно?

### Сценарий 1: Стандартная установка

После `initdb` PostgreSQL слушает на `localhost`. Но администратор меняет на `*` — «чтобы было удобно». pg_hba.conf остаётся с `local all all trust`. Теперь любой хост в сети может подключиться без пароля.

### Сценарий 2: Лишние расширения

Установлены `adminpack`, `file_fdw`, `dblink` — «на всякий случай». Не используются. Но `adminpack` позволяет читать файлы сервера. `dblink` — подключаться к другим базам. Поверхность атаки увеличена.

### Сценарий 3: Слабые пароли

Пользователь `postgres` с паролем `postgres`. Первое, что проверяет злоумышленник. Brute-force через `psql` — и доступ получен.

## Основные концепции

### CIS Benchmarks

**CIS PostgreSQL Benchmark** — чек-лист из ~100 пунктов:

| Категория | Пункты |
|-----------|--------|
| Установка | Доверенные репозитории, проверка контрольных сумм |
| Сеть | Ограничение listen_addresses, SSL |
| Аутентификация | scram-sha-256, запрет trust |
| Авторизация | Минимум привилегий, аудит ролей |
| Аудит | pgaudit, логи |
| Конфигурация | Отключение опасных функций |
| Обновление | Последняя стабильная версия |
| Бэкап | Регулярно + тестирование |

### Сетевая безопасность

**listen_addresses:**
```
# Неправильно:
listen_addresses = '*'

# Правильно:
listen_addresses = 'localhost,10.0.0.5'
```

**Порт и firewall:**
```bash
# Стандартный порт — известен
port = 5432

# Firewall: только нужные IP
iptables -A INPUT -p tcp --dport 5432 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 5432 -j DROP
```

### Аутентификация и авторизация

**Пароли:**
```sql
-- Метод шифрования
SHOW password_encryption;  -- Должно быть: scram-sha-256

-- Сменить пароль superuser
ALTER USER postgres WITH PASSWORD 'new-strong-password';

-- Отключить стандартный логин
ALTER USER postgres WITH NOLOGIN;
CREATE ROLE admin WITH LOGIN PASSWORD 'strong' SUPERUSER;
```

**Роли и права:**
```sql
-- Отозвать PUBLIC
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- Создать роли для приложений
CREATE ROLE app_read NOLOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
```

### Конфигурация

**postgresql.conf — безопасный шаблон:**
```
# Сеть
listen_addresses = 'localhost'
ssl = on
ssl_min_protocol_version = 'TLSv1.2'

# Аутентификация
password_encryption = 'scram-sha-256'

# Аудит
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p]: user=%u,db=%d,client=%h '

# Лимиты
max_connections = 100

# Расширения
shared_preload_libraries = 'pgaudit'

# Таймауты
statement_timeout = '30s'
idle_in_transaction_session_timeout = '60s'
```

**pg_hba.conf:**
```
local   all             postgres                                peer
local   all             all                                     scram-sha-256
hostssl all             all             10.0.0.0/24             scram-sha-256
host    all             all             0.0.0.0/0               reject
```

### Отключение опасного

```sql
-- Удалить неиспользуемые расширения
DROP EXTENSION IF EXISTS adminpack;
DROP EXTENSION IF EXISTS file_fdw;

-- Отключить загрузку произвольных библиотек
ALTER SYSTEM SET local_preload_libraries = '';
```

### Обновление

| Версия | Статус |
|--------|--------|
| 9.x-11 | EOL, обновить срочно |
| 12-14 | Поддержка |
| 15-16 | Рекомендуется |

**Процесс обновления:**
1. Прочитать release notes
2. Сделать бэкап
3. Протестировать на staging
4. Обновить (pg_upgrade или логическая репликация)
5. Проверить

## Сравнение харденинг-фреймворков

| Фреймворк | Фокус | PostgreSQL |
|-----------|-------|------------|
| **CIS Benchmarks** | Технические настройки | Да (CIS PostgreSQL) |
| **STIG (DISA)** | DoD systems | Да (PostgreSQL STIG) |
| **NIST 800-53** | Федеральные системы | Маппится на CIS |
| **PCI DSS** | Данные карт | Маппится на CIS |
| **ISO 27001** | Управление ИБ | Маппится на CIS |

## Уроки из реальных инцидентов

### Инцидент 1: Уязвимость через неиспользуемое расширение (2019)

Расширение `plpythonu` было установлено «на всякий случай». Злоумышленник использовал SQL Injection для создания функции:
```sql
CREATE FUNCTION evil() RETURNS void AS $$
  import os
  os.system('rm -rf /')
$$ LANGUAGE plpythonu;
```

**Хронология:** plpythonu — untrusted language. Может выполнять произвольный код. Функция создана через SQL Injection, выполнена.

**Ущерб:** Удаление данных. Восстановление из backup.

**Как харденинг изменил бы исход:** Удаление неиспользуемых расширений. Использование `plpython3u` (trusted) вместо `plpythonu`. Отсутствие права CREATE FUNCTION для приложения.

### Инцидент 2: Утечка через `pg_read_file` (2020)

Администратор оставил `adminpack` установленным. Злоумышленник использовал `pg_read_file('/etc/passwd')` — чтение системных файлов.

**Хронология:** `adminpack` содержит `pg_read_file`, `pg_logfile_rotate`. Злоумышленник читал конфигурацию, пароли, ключи.

**Ущерб:** Компрометация всей системы.

**Как харденинг изменил бы исход:** Удаление `adminpack`. Ограничение `pg_read_file` только superuser. Мониторинг вызовов `pg_read_file`.

## Инструменты и средства защиты

| Класс инструмента | Назначение | Позиция в жизненном цикле | Стандарты |
|-------------------|-----------|--------------------------|-----------|
| **CIS Benchmarks** | Харденинг-чеклист | Регулярно | CIS |
| **STIG** | DoD харденинг | Регулярно | DISA |
| **pg_secure** | Автоматический аудит | Регулярно | NIST |
| **Ansible / Terraform** | IaC для конфигурации | CI/CD | NIST CM-3 |
| **Lynis** | Сканер безопасности ОС | Регулярно | CIS |

## Архитектурные решения

### Чек-лист харденинга

| Проверка | Команда | Ожидаемый результат |
|----------|---------|---------------------|
| listen_addresses | `SHOW listen_addresses;` | Не `*` |
| password_encryption | `SHOW password_encryption;` | `scram-sha-256` |
| ssl | `SHOW ssl;` | `on` |
| log_connections | `SHOW log_connections;` | `on` |
| max_connections | `SHOW max_connections;` | <= 200 |
| shared_preload_libraries | `SHOW shared_preload_libraries;` | Содержит `pgaudit` |
| pg_hba trust | `grep trust pg_hba.conf` | Нет для удалённых |
| Unused extensions | `SELECT * FROM pg_extension;` | Минимум |
| Superusers | `SELECT rolname FROM pg_roles WHERE rolsuper;` | Минимум |
| File permissions | `ls -la $PGDATA/pg_hba.conf` | 600 |

### Автоматизация харденинга

**Ansible playbook:**
```yaml
- name: PostgreSQL Hardening
  hosts: db_servers
  tasks:
    - name: Set listen_addresses
      lineinfile:
        path: /etc/postgresql/15/main/postgresql.conf
        regexp: '^listen_addresses'
        line: "listen_addresses = 'localhost,10.0.0.5'"

    - name: Enable SSL
      lineinfile:
        path: /etc/postgresql/15/main/postgresql.conf
        regexp: '^ssl ='
        line: "ssl = on"

    - name: Set password_encryption
      lineinfile:
        path: /etc/postgresql/15/main/postgresql.conf
        regexp: '^password_encryption'
        line: "password_encryption = 'scram-sha-256'"
```

## Подготовка к собеседованию

### Три типичные ошибки кандидатов

**Ошибка 1:** «PostgreSQL безопасна по умолчанию».

По умолчанию PostgreSQL: trust для локальных, md5 для удалённых (устарел), слушает localhost. Но администраторы часто меняют на более «удобные» настройки. Безопасность требует явной конфигурации.

**Ошибка 2:** «Харденинг — одноразовая задача».

Харденинг — процесс. Новые версии, новые уязвимости, новые требования. Регулярный аудит конфигурации — ежеквартально.

**Ошибка 3:** «CIS Benchmarks — только для enterprise».

CIS подходит для любого размера. Это best practices, не over-engineering. Даже маленький проект выигрывает от basic hardening.

### Три ситуационных вопроса

**Вопрос 1:** «Как автоматизировать харденинг PostgreSQL в Kubernetes?»

Ответ: **ConfigMap/Secret** для postgresql.conf. **Init container** для применения hardening. **Pod Security Policy** — запрет privileged. **Network Policy** — доступ только из app namespace. **Vault sidecar** для secrets. **Ansible operator** для управления конфигурацией.

**Вопрос 2:** «Как проверить, что hardening не сломал приложение?»

Ответ: **Staging environment** с теми же настройками. **Automated tests** — функциональные + security. **Slow rollout** — Canary deployment. **Monitoring** — метрики ошибок, latency. **Rollback plan** — быстрый откат при проблемах.

**Вопрос 3:** «Как соотнести CIS Benchmarks с PCI DSS?»

Ответ: CIS — технические настройки. PCI DSS — бизнес-требования. Маппинг: CIS 3.1 (password policy) → PCI 8.2.3. CIS 4.1 (SSL) → PCI 4.1. CIS 6.1 (audit) → PCI 10.2. Использовать CIS как implementation guide для PCI.

## Чек-лист понимания

- [ ] Что такое **харденинг**?
- [ ] Что такое **CIS Benchmarks**?
- [ ] Какие **параметры** ограничить в postgresql.conf?
- [ ] Как настроить **pg_hba.conf** безопасно?
- [ ] Какие **расширения** удалить?
- [ ] Как **обновлять** PostgreSQL?
- [ ] Как **аудитировать** конфигурацию?
- [ ] Что такое **STIG**?
- [ ] Как **автоматизировать** харденинг?
- [ ] Как **проверить** hardening?

### Ответы на чек-лист

1. **Харденинг** — усиление защиты: отключение лишнего, включение нужного, минимизация поверхности атаки.

2. **CIS Benchmarks** — стандартизированный чек-лист настроек безопасности. Для PostgreSQL — ~100 пунктов.

3. **Параметры:** listen_addresses, ssl, password_encryption, log_connections, max_connections, shared_preload_libraries.

4. **pg_hba.conf:** peer/scram для локальных, hostssl/scram для удалённых, reject для всего остального.

5. **Расширения:** adminpack, file_fdw, dblink, plpythonu — если не используются.

6. **Обновление:** release notes → backup → staging → pg_upgrade → проверка.

7. **Аудит:** регулярный запуск CIS scanner, ревью конфигурации, мониторинг изменений.

8. **STIG** — Security Technical Implementation Guides. DoD стандарт, аналог CIS.

9. **Автоматизация:** Ansible, Terraform, Chef, Puppet. Infrastructure as Code.

10. **Проверка:** staging environment, automated tests, canary deployment, monitoring.

## Ключевые выводы для собеседования

- **PostgreSQL не безопасна по умолчанию** — требует hardening
- **CIS Benchmarks — roadmap** — от «установил» до «защищён»
- **Процесс, не проект** — регулярный аудит, обновление
- **Automation** — Ansible/Terraform для конфигурации
- **Staging first** — never harden production directly

---

_Статья создана на основе анализа материалов по PostgreSQL_
