# Docker Security Overview: Всесторонняя безопасность контейнеров

## Что такое Docker Security?

Docker Security — это не один инструмент, не одна политика, не один механизм. Это система — совокупность мер, которые защищают контейнеры на всех этапах их жизненного цикла: от написания Dockerfile до запуска в production, от хранения образов до мониторинга работающих контейнеров.

Представьте, что вы — начальник службы безопасности целого квартала. Вы не можете просто поставить один замок на воротах и считать себя в безопасности. Вам нужно: проверить всех жильцов (сканирование образов), выдать им пропуска (аутентификация), настроить камеры (мониторинг), обучить охрану (политики безопасности), регулярно проверять замки (CIS Benchmark), обновлять системы (патчи). Docker Security — это те же задачи, но в мире контейнеров.

Docker Security охватывает цепочку поставки (от коммита до деплоя), изоляцию контейнеров (защита от выхода), управление секретами, сетевое взаимодействие, хранение образов и мониторинг. Каждый слой защиты дополняет предыдущий. Если один слой пробит — другие должны остановить атаку.

## Зачем это нужно?

**Сценарий 1: Полный аудит безопасности Docker.** В Сбере проводят annual security review Docker-инфраструктуры. Аудиторы проверяют: CIS Benchmark (0 critical failures), сканирование образов (нет критических CVE), runtime-защиту (Falco включён, правила актуальны), управление секретами (Vault, не переменные окружения), сетевые политики (microsegmentation), логирование (ELK включён, PII masked). Если какой-то пункт не выполнен — создаётся remediation план.

**Сценарий 2: Инцидент безопасности в production.** Один из микросервисов HR-платформы скомпрометирован (RCE). Без Docker Security: злоумышленник выполняет shell в контейнере, монтирует диск хоста, крадёт базу данных. С Docker Security: drop all capabilities блокирует mount, seccomp блокирует ptrace, read-only rootfs не позволяет устанавливать инструменты, Falco отправляет alert, контейнер изолируется. Ущерб — только данные внутри скомпрометированного контейнера.

**Сценарий 3: Compliance аудит (152-ФЗ, PCI DSS).** Регулятор проверяет, как компания защищает персональные данные сотрудников в контейнерах. Требуется: шифрование данных (TLS между микросервисами), аудит доступа (кто запускал контейнеры?), ограничение доступа (RBAC в registry), защита от утечек (DLP, runtime-мониторинг). Docker Security предоставляет все необходимые инструменты для compliance.

## Основные концепции

### Security Pyramid: Слои защиты

Docker Security строится как пирамида: каждый слой опирается на предыдущий и защищает от определённых угроз.

**Layer 1: Host Security.** Самая основа — защита хоста, на котором работают контейнеры. Свежее ядро (live patching), CIS Benchmark, аудит (auditd), разделы для Docker. Если хост скомпрометирован — все контейнеры скомпрометированы.

**Layer 2: Daemon Security.** Защита Docker daemon. TLS для remote API, rootless mode, ограничение доступа к socket, правильная конфигурация daemon. Если daemon скомпрометирован — злоумышленник управляет всеми контейнерами.

**Layer 3: Image Security.** Проверка образов перед запуском. Базовые образы (Alpine, Distroless), сканирование на CVE (Trivy, Snyk), подпись (Cosign), SBOM. Образ — «ингредиент». Если ингредиент испорчен — блюдо будет плохим.

**Layer 4: Container Runtime.** Ограничения на работающий контейнер. Capabilities (drop all), seccomp (блокировка syscall'ов), AppArmor (файлы и сеть), read-only rootfs, user namespace. Если злоумышленник проник в контейнер — runtime ограничивает ущерб.

**Layer 5: Network Security.** Сетевая изоляция. Микросегментация, шифрование трафика (mTLS), network policies, правильная публикация портов. Если контейнер скомпрометирован — злоумышленник не может перемещаться по сети (lateral movement).

**Layer 6: Registry Security.** Хранение и распространение образов. Аутентификация (LDAP/OIDC), авторизация (RBAC), сканирование, подпись, репликация. Registry — библиотека образов. Если библиотека не защищена — кто угодно может подложить отравленную книгу.

**Layer 7: Orchestration Security.** Управление контейнерами. Docker Swarm (mTLS, locked swarm), Kubernetes (RBAC, Pod Security, Service Mesh), secrets management. Оркестратор — центральный мозг. Если мозг захвачен — всё тело под контролем.

**Layer 8: Monitoring and Alerting.** Обнаружение атак. Falco (runtime security), Prometheus (метрики), ELK (логи), Docker Events (аудит). Без мониторинга вы не узнаете об атаке, пока не станет слишком поздно.

### Shift Left: Безопасность на ранних этапах

Shift Left — это принцип, по которому безопасность применяется на самых ранних этапах разработки, а не только в production. Чем раньше найдена проблема — тем дешевле её исправить.

| Этап | Действие | Инструмент |
|------|----------|------------|
| Код | SAST-анализ, поиск секретов | Semgrep, GitLeaks |
| Dockerfile | Lint, проверка best practices | Hadolint |
| Build | Сканирование образа | Trivy, Snyk |
| Registry | Сканирование + подпись | Harbor + Cosign |
| Deploy | Admission control | OPA/Kyverno |
| Runtime | Falco, мониторинг | Falco, Prometheus |

Shift Left — это как проверка продуктов на входе в магазин, а не после того, как покупатель уже съел испорченный продукт. Проверять образ при загрузке в registry дешевле, чем разворачивать его и ловить атаку в production.

### Defense in Depth: Многослойная защита

Defense in Depth — это принцип, по которому ни один механизм защиты не должен быть единственным. Если один механизм пробит — другие продолжают защищать.

Пример Defense in Depth для Docker:
1. Образ не содержит уязвимостей (сканирование)
2. Образ подписан (Cosign)
3. Контейнер не имеет лишних прав (drop capabilities)
4. Seccomp блокирует опасные syscall'ы
5. AppArmor ограничивает файлы
6. Read-only rootfs
7. Network policy изолирует контейнер
8. Falco мониторит аномалии
9. SIEM анализирует события

Атакующему нужно пробить все 9 слоёв. Пробить один слой — недостаточно.

### Zero Trust для контейнеров

Zero Trust в мире контейнеров означает: «Никакой внутренний трафик не является доверенным по умолчанию». Каждый запрос между микросервисами должен быть аутентифицирован и авторизован.

Принципы Zero Trust для Docker:
- **Verify explicitly** — каждый запрос проверяется (mTLS, JWT)
- **Least privilege** — минимальные права для каждого контейнера
- **Assume breach** — считаем, что контейнер уже скомпрометирован, и строим защиту исходя из этого

Zero Trust реализуется через:
- Service Mesh (Istio, Linkerd) — mTLS между всеми микросервисами
- Network Policies — точные правила доступа между контейнерами
- API Gateway — единая точка входа с аутентификацией
- Falco — мониторинг, даже если Zero Trust не сработал

## Сравнение уровней зрелости безопасности Docker

| Уровень | Host | Daemon | Images | Runtime | Network | Registry | Monitoring |
|---------|------|--------|--------|---------|---------|----------|------------|
| Level 1 (Base) | Docker установлен | Daemon default | Docker Hub images | Default | Default bridge | No auth | No monitoring |
| Level 2 (Hardened) | Updates, CIS | TLS, icc: false | Alpine, scanning | Non-root, cap drop | User-defined bridge | Harbor, RBAC | Docker events |
| Level 3 (Enterprise) | Live patching, audit | Rootless, userns | Distroless, SBOM | Seccomp, AppArmor, RO | Overlay, mTLS | Harbor + Cosign + Scan | Falco, ELK, SIEM |
| Level 4 (Zero Trust) | Immutable host | TLS + mTLS | Signed, attested | Falco + OPA | Service mesh | SLSA attestation | AI-based anomaly |

## Практические применения

**Сценарий 1: Enterprise Docker Security Policy.** В Сбере утверждена «Политика Docker Security»:
- Все образы — только из корпоративного Harbor (блокировка других registry)
- Любой образ с критической CVE — заблокирован для деплоя
- Контейнеры — non-root, cap-drop=ALL, seccomp default, read-only rootfs
- Привилегированные контейнеры — запрещены
- Docker socket не монтируется в контейнеры
- Falco обязателен на всех хостах
- Еженедельный CIS Benchmark check
- Нарушение политики — P0 инцидент с расследованием

**Сценарий 2: CI/CD Security Pipeline.** GitLab CI pipeline включает:
1. Hadolint — проверка Dockerfile на best practices
2. Trivy — сканирование образа на CVE и секреты
3. Cosign — подпись образа
4. Harbor — сканирование и блокировка при CVE
5. Kubernetes — проверка подписи через OPA (admission control)
6. Falco — runtime-мониторинг
7. Prometheus — метрики
Pipeline падает на любом этапе при обнаружении критической проблемы.

**Сценарий 3: Incident Response для Docker.** При обнаружении аномалии (Falco alert):
1. SIEM получает alert (severity: CRITICAL)
2. Автоматическая изоляция контейнера (сетевая изоляция, snapshot)
3. Дежурный аналитик проверяет в течение 5 минут
4. Если реальная атака — создаётся инцидент
5. Контейнер сохраняется для судебной экспертизы (forensics)
6. Хост проверяется на компрометацию
7. При компрометации — хост пересоздаётся из образа
8. Post-mortem: как злоумышленник проник? как предотвратить?

**Сценарий 4: Compliance Dashboard.** В Grafana создан дашборд «Docker Security Compliance»:
- CIS Benchmark score для каждого хоста (цель: >95%)
- Количество образов с CVE в registry
- Количество privileged контейнеров (цель: 0)
- Количество контейнеров с Docker socket mount (цель: 0)
- Falco alerts за последние 24 часа
- Docker events (подозрительные действия)
Дашборд проверяется на weekly security review.

## Ограничения и нюансы

**Docker Security — это процесс, не продукт.** Нельзя установить один инструмент и считать Docker-инфраструктуру защищённой. Нужна постоянная работа: обновление политик, обучение команды, аудит, реагирование на инциденты. Безопасность — это марафон, а не спринт.

**Human factor — слабейшее звено.** Разработчик запускает контейнер с `--privileged», чтобы «быстро проверить». DevOps открывает Docker API без TLS, «чтобы CI работал». Стажёр коммитит пароль в Dockerfile. Технические меры (admission control, policy as code) должны защищать от человеческих ошибок.

**Zero Trust дороже.** Service mesh, mTLS, OPA — это сложная инфраструктура. Не каждый проект может позволить себе десятки K8s контроллеров. Выбирайте уровень защиты, соответствующий критичности данных и бюджету.

**Compliance — не равен безопасности.** Можно пройти PCI DSS аудит (настроить логирование, RBAC, шифрование) — и всё равно быть скомпрометированным. Compliance — минимальный порог. Security — непрерывный процесс улучшения сверх compliance.

## Чек-лист полной безопасности Docker

- [ ] Какие 8 слоёв безопасности Docker?
- [ ] Что такое Shift Left в контексте Docker?
- [ ] Как Defense in Depth применяется к Docker?
- [ ] Что такое Zero Trust для контейнеров?
- [ ] Как измерить зрелость Docker Security?
- [ ] Какие этапы CI/CD pipeline для безопасности?
- [ ] Почему человеческий фактор — главная угроза?
- [ ] Как Incident Response работает для Docker?
- [ ] Чем compliance отличается от security?
- [ ] Какие политики Docker Security нужны enterprise?

### Ответы на чек-лист

1. **Какие 8 слоёв безопасности Docker?** (1) Host Security — защита хоста, ядра, CIS Benchmark. (2) Daemon Security — защита Docker daemon, TLS, socket. (3) Image Security — сканирование образов, подпись, base images. (4) Container Runtime — capabilities, seccomp, AppArmor, read-only rootfs. (5) Network Security — микросегментация, mTLS, network policies. (6) Registry Security — auth, RBAC, scan, replication. (7) Orchestration Security — Swarm/K8s safety, secrets, RBAC. (8) Monitoring — Falco, Prometheus, ELK, audit. Каждый слой опирается на предыдущий.

2. **Что такое Shift Left в контексте Docker?** Shift Left — перенос контроля безопасности на ранние этапы. Вместо обнаружения атаки в production — проверка безопасности в CI/CD. SAST ищет уязвимости в коде, Hadolint проверяет Dockerfile, Trivy сканирует образ при сборке, Harbor блокирует образ с CVE до деплоя. Чем левее сдвинут контроль — тем дешевле исправление и меньше риск.

3. **Как Defense in Depth применяется к Docker?** Defense in Depth — многослойная защита. Каждый слой защищает от определённого типа угроз. Layer 1 (Host) защищает от хостовых уязвимостей. Layer 4 (Runtime) защищает от Container Escape. Layer 5 (Network) защищает от Lateral Movement. Если Layer 4 не сработал — Layer 5 продолжает защищать. Атакующему нужно пробить все слои. Ни один слой не должен быть единственной линией защиты.

4. **Что такое Zero Trust для контейнеров?** Zero Trust в Docker — принцип «никому не доверяй, проверяй всех». Каждый запрос между микросервисами аутентифицируется (mTLS). Каждый контейнер имеет минимальные права. Каждое сетевое соединение авторизуется (NetworkPolicy). Даже если злоумышленник проник в один контейнер — он не может свободно общаться с другими контейнерами. Zero Trust исходит из предположения, что компрометация уже произошла.

5. **Как измерить зрелость Docker Security?** Четыре уровня: Level 1 (Base) — Docker установлен, контейнеры запускаются, никакой безопасности. Level 2 (Hardened) — CIS Benchmark, сканирование образов, non-root, bridge сети. Level 3 (Enterprise) — Falco, ELK, SIEM, подпись образов, RBAC в registry, live patching. Level 4 (Zero Trust) — immutable hosts, mTLS, service mesh, SLSA attestation, AI-аномалии. Большинство компаний находятся на Level 1-2. Сбер/Крупные банки — Level 3. Level 4 — единицы.

6. **Какие этапы CI/CD pipeline для безопасности?** (1) SAST — статический анализ кода. (2) GitLeaks — поиск секретов в коде. (3) Hadolint — проверка Dockerfile. (4) Trivy — сканирование образа на CVE. (5) Cosign — подпись образа. (6) Harbor — сканирование и блокировка CVE. (7) OPA/Kyverno — admission control в K8s. (8) Falco — runtime-мониторинг. (9) Prometheus/ELK — метрики и логи. Pipeline падает на любом этапе при проблеме.

7. **Почему человеческий фактор — главная угроза?** Технические меры (seccomp, AppArmor, CIS) можно обойти, если разработчик запускает `--privileged» или DevOps открывает API без TLS. Люди — самое слабое звено. Решения: Policy as Code (OPA), admission control (Kyverno), обучение команды (security champion), автоматические блокировки (CIS сканер блокирует деплой при нарушениях), code review Dockerfile’ов.

8. **Как Incident Response работает для Docker?** Incident Response для Docker: (1) Обнаружение — Falco alert в SIEM. (2) Изоляция — контейнер изолируется сетевой политикой. (3) Forensics — делается snapshot контейнера (docker commit, docker export). (4) Анализ — логи (ELK), метрики (Prometheus), события (Docker events). (5) Эрадикация — хост проверяется, скомпрометированный хост пересоздаётся. (6) Восстановление — чистый контейнер из подписанного образа. (7) Post-mortem — как предотвратить повторение.

9. **Чем compliance отличается от security?** Compliance — соответствие минимальным требованиям регулятора (152-ФЗ, PCI DSS, SOC 2). Security — защита от реальных угроз, часто сверх требований. Можно пройти compliance (настроить логи, шифрование, RBAC) и быть скомпрометированным (zero-day уязвимость в Docker). Security — это compliance + защита от неизвестных угроз. Для реальной безопасности нужно идти дальше чек-листов compliance.

10. **Какие политики Docker Security нужны enterprise?** (1) Image policy — только из корпоративного registry, signed, scanned. (2) Container policy — non-root, cap-drop=ALL, read-only rootfs, no privileged. (3) Network policy — микросегментация, mTLS, default deny. (4) Secrets policy — Vault или Docker Secrets, не в образах. (5) Access policy — RBAC для registry и K8s, TLS для API. (6) Patch policy — обновление Docker Engine в 48 часов, live patching ядра. (7) Audit policy — Falco, ELK, weekly CIS Benchmark. (8) Incident Response policy — для Docker-инцидентов.

---

_Статья создана на основе анализа материалов Habr: Docker Security — полный обзор (habr.com/ru/company/southbridge/blog/339126), CIS Docker Benchmark (habr.com/ru/articles/435748), Zero Trust для контейнеров (habr.com/ru/companies/selectel/articles/1017504), Shift Left в Docker (habr.com/ru/companies/otus/articles/858780), Docker Security Best Practices (habr.com/ru/articles/653153)_
