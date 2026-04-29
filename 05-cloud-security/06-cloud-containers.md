# Безопасность контейнеров в облаке: Защита образов, оркестрации и runtime изоляции в Kubernetes и облачных средах

## Что такое безопасность контейнеров?

Представьте контейнерный порт. Тысячи грузовых контейнеров прибывают, хранятся, перемещаются и отправляются. Система безопасности порта — это не одна камера на воротах. Это: проверка контейнера на заводе-отправителе (соответствует ли заявленному содержимому, нет ли контрабанды), защита при транспортировке (пломбы, GPS-трекеры), сканирование в порту прибытия (рентген, собаки), изоляция опасных грузов в отдельной зоне, контроль перемещения погрузчиков (кто и куда везёт контейнер), аудит каждой операции и автоматическая блокировка подозрительной активности.

Теперь замените «контейнер» на Docker-образ, «порт» — на Kubernetes-кластер в облаке, «погрузчик» — на scheduler, перемещающий поды между нодами, и вы получите полную картину контейнерной безопасности. **Безопасность контейнеров** — не однослойная защита «просканировали образ — всё», а многоуровневая архитектура безопасности на всём жизненном цикле: создание образа (build), хранение в реестре (registry), развёртывание (deploy), выполнение (runtime) и сетевое взаимодействие (orchestration). Фундаментальная проблема: контейнер делит ядро с хостом (в отличие от VM, у которой своё ядро), и слабая изоляция контейнера — это highway к privilege escalation до хостовой ОС и всем остальным контейнерам на том же хосте.

## Эволюция и мотивация

Контейнеризация прошла путь от developer convenience tool (Docker, 2013) до стандартного способа запуска production workloads (Kubernetes, 2017+), и безопасность эволюционировала соответственно: от «контейнер изолирован и безопасен сам по себе» до reality «контейнер — это процесс на хосте с ослабленной изоляцией, нужны дополнительные слои защиты».

Три парадигмальных сдвига. Первый — осознание, что shared kernel — фундаментальное ограничение. VM имеет свой kernel, свой драйвер, своё userspace — компрометация VM не означает компрометацию хоста (без exploit гипервизора). Контейнер использует kernel хоста — уязвимость в kernel позволяет container escape: из ограниченного контейнера выйти в полноценную root-среду хоста. Это потребовало mandatory access control (Seccomp, AppArmor, SELinux), сужающего системные вызовы, доступные контейнеру.

Второй — экспоненциальный рост supply-chain через образы. Docker-образ — это слоёный пирог: base image (Ubuntu/Alpine) + зависимости (apt-get packages, npm install, pip install) + код приложения. Каждый слой — потенциальная уязвимость. В 2021 году Log4Shell (CVE-2021-44228) продемонстрировал масштаб проблемы: уязвимость была в Java-библиотеке, которая тянулась как transitive dependency в тысячах образов, и каждая из этих тысяч была уязвима. Software Bill of Materials (SBOM) перешёл из теоретического концепта в mandatory requirement — CIS v8 Control 2.5 (Software Inventory) и Executive Order 14028 (2021) требуют SBOM для всего ПО, включая контейнеры.

Третий — оркестрация как новый слой атаки. Kubernetes — не просто Docker с GUI, а сложная распределённая система с собственным API, RBAC, сетевой моделью, хранилищем секретов и admission webhooks. Количество CVE для Kubernetes components и неправильных конфигураций кластеров растёт экспоненциально, и misconfigured Kubernetes RBAC/Network Policy стал source номер один для lateral movement в облачных атаках 2022-2025. MITRE ATT&CK Container Matrix отдельно описывает container-specific техники атаки.

## Зачем нужна контейнерная безопасность?

**Сценарий 1: Уязвимый base image.** Стартап использует `node:latest` как base image для всех микросервисов. В феврале 2024 года в `node:latest` обнаружена critical CVE, эксплуатация которой даёт RCE на контейнере. 37 production-подов базируются на этом образе. Без системы сканирования образов — команда не знает об уязвимости, пока не произойдёт инцидент. С архитектурой безопасности: (a) CI/CD pipeline — Trivy сканирует образ на этапе build и блокирует push при CRITICAL CVE, (b) Registry scan — ECR/ACR/GCR continuously отслеживает уже хранящиеся образы и генерирует finding, если CVE обнаружена в pushed ранее образе, (c) Admission Controller (Kyverno/OPA Gatekeeper) запрещает deployment подов из образов с CRITICAL CVE без exception waiver. Zero-day уязвимость — образы уже в production: runtime detection (Falco/Tetragon) мониторит abnormal process execution из контейнера и генерирует alert за секунды.

**Сценарий 2: Container escape через привилегированный режим.** DevOps-инженер, борясь с проблемой монтирования NFS-тома, запускает под с `privileged: true` — «временно». Privileged-контейнер имеет доступ ко всем устройствам хоста, всем capabilities, всем kernel modules. Через 3 месяца уязвимость в веб-приложении позволяет злоумышленнику получить shell в этом контейнере — и мгновенно host escape, потому что privileged container = эквивалент root на хосте. С архитектурой безопасности: (a) Pod Security Standards enforced через admission controller — Restricted по умолчанию, Privileged требует explicit exception label, (b) Kyverno policy: `privileged: true` запрещён для production namespace, (c) Falco runtime rule: alert если контейнер монтирует host filesystem (sensitive mount). NIST SP 800-190 (Container Security) раздел 4.1 требует ограничения container capabilities.

**Сценарий 3: Секреты в ConfigMap и незашифрованные Kubernetes Secrets.** Команда сохраняет API-ключ к платежному шлюзу в Kubernetes Secret, считая его «защищённым». Но Kubernetes Secret в etcd хранится в base64 (не шифрование, а encoding) по умолчанию. Злоумышленник, скомпрометировавший pod с `cluster-admin` RBAC (или получивший доступ к etcd), видит plaintext secret. С архитектурой: (a) Encryption at rest для Kubernetes Secrets (EncryptionConfiguration с AES-256-CBC или KMS plugin), (b) External Secrets Operator синхронизирует секреты из AWS Secrets Manager/Azure Key Vault в Kubernetes Secrets, с автоматической ротацией — plaintext secret никогда не существует в Kubernetes статически больше чем несколько минут, (c) Kyverno policy: запрет монтирования Secret volume для подов без explicit `use-secret: "payment-api-key"` label. CIS Kubernetes Benchmark 5.3.1 (Secrets Encryption) требует encryption at rest.

**Сценарий 4: Lateral movement внутри кластера.** Злоумышленник через RCE в веб-поде получает limited shell. С дефолтным Kubernetes networking: под может обращаться к любому другому поду в кластере на любом порту. Атакующий сканирует кластер, находит PostgreSQL pod на порту 5432, эксфильтрует данные клиентов. С Network Policy: разрешено только web-pod → api-pod:8080, api-pod → db-pod:5432. Любой другой трафик отбрасывается на уровне CNI (Calico/Cilium) — сканирование и lateral movement невозможны. CIS Kubernetes Benchmark 5.3.2 требует Network Policies для critical namespaces.

**Сценарий 5: Неизменяемый (immutable) рантайм для compliance.** Аудитор PCI DSS запрашивает: «Как вы гарантируете, что злоумышленник или администратор не может модифицировать работающий production-контейнер?» С архитектурой: (a) `readOnlyRootFilesystem: true` — контейнер не может писать в свою файловую систему (writeable volumes — только emptyDir/tmp), (b) `runAsNonRoot: true`, (c) `allowPrivilegeEscalation: false` — запрет setuid-битов и дочерних процессов с повышенными правами, (d) Kyverno validation: любой production pod без этих трёх флагов → reject deployment. CIS Kubernetes Benchmark 5.2.6 (Read-only Root Filesystem) обязателен для production.

## Основные концепции

### Безопасность образа (Image Security)

Образ — фундамент контейнера. Компрометация образа = компрометация всех контейнеров, запущенных из этого образа. Стратегия безопасности образа — shift-left: проверка на каждом этапе создания, а не после deployment.

**Минимализация поверхности атаки (Distroless, Alpine):** каждый пакет в образе — потенциальная уязвимость. Минимальный образ (Distroless — нет shell, package manager, debugging tools) радикально сокращает поверхность: exploit RCE — злоумышленник получает shell (но shell'а нет в Distroless), хочет скачать инструменты через curl (но curl нет). Это не защита от детерминированной атаки (атакующий может инжектить статически скомпилированный бинарник), но радикально усложняет post-exploitation.

**SBOM (Software Bill of Materials):** машиночитаемый манифест всех компонентов образа: пакеты, библиотеки, их версии и provenance (откуда взялись). CycloneDX, SPDX — форматы SBOM. При обнаружении CVE (например, OpenSSL), наличие SBOM позволяет за минуты, а не дни, ответить на вопрос: «Какие из наших 500 production-образов содержат уязвимую версию OpenSSL?». Executive Order 14028 и CIS v8 Control 2.5 требуют SBOM.

**Подпись образов (Image Signing):** Sigstore/Cosign — подпись образа с verification через OIDC, привязанную к CI/CD identity. Admission Controller проверяет подпись перед разрешением deployment — нельзя деплоить образ, подписанный не-валидным CI/CD пайплайном. Это блокирует supply-chain вектор: даже если злоумышленник скомпрометировал registry, его unsigned образ не пройдёт admission. OWASP Top 10 CI/CD Security (2022) требует artefact integrity verification на каждом этапе.

### Безопасность Kubernetes API и RBAC

Kubernetes API — центральная точка контроля кластера. Компрометация `cluster-admin` ServiceAccount (через украденный токен из скомпрометированного пода) = полный контроль кластера: создание/удаление ресурсов, чтение Secrets, выполнение команд в любом поде.

**RBAC Least Privilege:** каждый ServiceAccount (для пода), User (для разработчика) и Group имеет минимальный набор разрешений. ServiceAccount web-app не имеет права читать Secrets других namespace или создавать Ingress. Audit Policy логирует все RBAC-отклонения (forbidden API calls) — это ранний индикатор атаки: кто-то пытается сделать то, на что у него нет прав.

**Admission Controllers:** Kyverno/OPA Gatekeeper — dynamic policy engine, который intercept'ит каждый API-запрос на создание/изменение ресурса и валидирует/мутирует его согласно политикам. Kyverno policy: «запретить создание пода без `readOnlyRootFilesystem: true` в namespace production». Без Admission Controllers Kubernetes RBAC — coarse-grained (разрешить/запретить create pod), а не property-level (разрешить create pod, но запретить `privileged: true`). Admission Controllers дают property-level enforcement.

**Audit Logging и Threat Detection:** Kubernetes Audit Policy определяет, какие API-вызовы логировать. Для security — логировать все RequestResponses на: secrets, serviceaccounts, roles/clusterroles, pods/exec, pods/attach. Audit logs стримятся в SIEM; на них строятся detection правила: «ServiceAccount в namespace development запросил secrets в namespace production → CRITICAL alert». Falco с Kubernetes audit events даёт runtime обнаружение: Secret accessed by unexpected pod.

### Runtime Security: Seccomp, AppArmor/SELinux, Capabilities

Runtime защита — последний рубеж. Если образ уязвим, RBAC скомпрометирован, admission controller обойдён — runtime security ограничивает, что контейнер может сделать внутри ядра.

**Seccomp (Secure Computing Mode):** фильтр системных вызовов, которые контейнер может использовать. Default Seccomp profile блокирует ~300 из ~435 системных вызовов, включая `mount`, `kexec`, `reboot`, `ptrace` (один процесс контейнера не может attach'иться к другому). Кастомный Seccomp profile для конкретного приложения: Node.js серверу не нужен `clone` (создание дочерних процессов), `chmod` или `ioctl` (управление устройствами) — их можно заблокировать, сужая возможные действия даже успешного RCE-exploit.

**AppArmor / SELinux (Mandatory Access Control, MAC):** в отличие от user/group прав (DAC — discretionary), MAC-политики определяют, что процесс может делать на уровне меток безопасности, независимо от его root-статуса. AppArmor профиль: контейнер может читать только `/usr/share/nginx/html/*`, писать только `/var/log/nginx/*`, не может читать `/etc/shadow` хоста (даже если mount host filesystem). Это ограничивает blast radius container escape даже до хост-уровня.

**Linux Capabilities:** разбиение root-привилегий на детальные единицы: `CAP_NET_BIND_SERVICE` (биндить на порт ниже 1024), `CAP_SYS_TIME` (менять системное время), `CAP_SYS_ADMIN` (очень широкий admin доступ). Контейнер запускается с минимальными capabilities (`NET_BIND_SERVICE`, `CHOWN`) и без `CAP_SYS_ADMIN` — невозможно монтировать filesystems, загружать kernel modules, использовать `clone(CLONE_NEWNS)` (создание новых namespaces — prerequisite для container escape). CIS Kubernetes Benchmark 5.2.8 требует `Drop ALL capabilities`.

## Сравнение стратегий контейнерной изоляции

| Критерий | Namespace Isolation (Docker default) | gVisor (Sentry) | Firecracker microVM (AWS Fargate) |
|----------|--------------------------------------|-----------------|------------------------------------|
| Модель изоляции | Process-level: контейнер = изолированная группа процессов, разделяющая kernel хоста | User-space kernel: контейнер видит эмулированное ядро (Sentry), реальное ядро не доступно | Virtual machine: каждый контейнер — легковесная VM с собственным kernel (LKVM), hardware virtualization |
| Защита от kernel exploits | Нет: уязвимость в kernel хоста → container escape | Высокая: exploit ядра — уязвляет эмулированное ядро Sentry, не реальное | Полная: каждый контейнер в VM, нет shared kernel, exploit kernel in VM не даёт доступ к хосту |
| Производительность (overhead) | ~0%: native kernel execution | 5-15%: system call interception overhead | 3-8%: lightweight VM overhead (меньше, чем полная QEMU/KVM) |
| Совместимость с приложениями | Полная: все system calls доступны | ~95%: ограниченная поддержка `/proc`, `/sys`, некоторых ioctls | Полная: приложение видит реальный Linux kernel, просто в VM |
| Типичный сценарий | Trusted workloads (internal tools, CI/CD runners), Development environments | Multi-tenant SaaS, где tenant'ы не доверяют друг другу и платформе | Serverless containers (AWS Fargate, GCP Cloud Run), где isolation critical между tenant'ами |
| Стандарты | NIST 800-190 baseline (Seccomp + SELinux как дополнение) | NIST 800-190 hardened, NIST 800-207 Zero Trust (tenant isolation) | NIST 800-53 SC-39 (Process Isolation), FedRAMP (government workloads) |

Выбор: Namespace Isolation + Seccomp + SELinux — достаточен для single-tenant приложений (ваше приложение не обслуживает чужих tenant'ов). Если контейнер скомпрометирован, kernel exploit required for escape — risk mitigated через kernel hardening и Seccomp.

gVisor — intermediate: не микро-VM, но гораздо сильнее изолирует application kernel от host kernel. Подходит для multi-tenant SaaS, где важно гарантировать tenant isolation без полного overhead микро-VM.

Firecracker microVM — максимальная изоляция, но наибольший overhead на холодный старт (100-200ms против 10ms namespaced container). AWS Fargate и GCP Cloud Run используют микро-VM для изоляции между клиентами — mandatory requirement для публичных serverless container-сервисов, потому что один клиент не должен иметь технической возможности escape к другому.

## Уроки из реальных инцидентов

### Инцидент CVE-2019-5736 (runc Container Escape, 2019)

Хронология: исследователи Adam Iwaniuk и Borys Popławski из Capsule8 обнаружили уязвимость в runc — стандартном container runtime для Docker и Kubernetes. Уязвимость позволяла злоумышленнику с root-доступом внутри контейнера перезаписать runc binary на хосте, и при следующем запуске любого контейнера (через `docker exec` или `kubectl exec`) — вредоносный runc выполнялся как root на хосте, давая полный host escape. Масштаб: уязвимость затрагивала практически все контейнеризованные среды (любой Docker runtime, все managed Kubernetes сервисы, все cloud container platform).

Корневая причина: runc не проверял, что file descriptor, переданный из контейнера, действительно принадлежит тому `/proc/self/exe`, который контейнер видит. Атакующий с root'ом в контейнере мог подменить `/proc/self/exe` через chroot и символическую ссылку, указывающую на runc binary хоста.

Как архитектура безопасности смягчила бы: (a) Seccomp profile, блокирующий `ptrace` и `chroot` (syscalls, использованные в эксплойте) — exploit не сработал бы, даже при RCE внутри контейнера, (b) Non-root container: злоумышленник не имеет root внутри контейнера, недостаточно прав для chroot, даже при успешном RCE (приложение работает от user nobody, атакующий получает shell nobody), (c) Read-only root filesystem — атакующий не может записать exploit binary на диск контейнера, нужен другой путь доставки.

Урок: даже production-grade runtime (runc) имеет уязвимости. Container escape — не теоретический, а реальный класс атак, и mitigation через Seccomp + non-root + read-only FS — mandatory baseline, а не optional hardening. NIST SP 800-190 Section 4 (Container Runtime Risks) описывает этот класс атак и рекомендует mitigation.

### Типичный сценарий инцидента: S3-ключи в environment variables контейнера

Хронология: DevOps создал Kubernetes Secret `aws-prod-keys` с `AWS_ACCESS_KEY_ID` и `AWS_SECRET_ACCESS_KEY` от production IAM-пользователя. Secret монтируется во все production-поды как environment variables. Веб-приложение пишет verbose debug log'и (через `console.log(process.env)`) — весь environment dump попадает в CloudWatch. CloudWatch logs читаются командой мониторинга и разработчиками (широкий доступ). Через месяц security researcher (по программе Bug Bounty) находит AWS access keys в CloudWatch logs репозитория.

Причина: долгоживущие IAM-user keys вместо IRSA/OIDC federation, environment variables вместо Secret Manager, verbose logging sensitive environment variables.

Remediation с контейнерной архитектурой: (a) IRSA (IAM Roles for Service Accounts) в EKS или Workload Identity в GKE — под не хранит ключи вообще, временные STS-credentials через OIDC federation, (b) Secrets никогда не передаются как env vars (среда контейнера = plaintext, видна в `docker inspect`, `kubectl describe pod`, debug logs), монтируются как volume с `readOnly: true` и `tmpfs` (в памяти, не на диске), (c) Kyverno policy: запрет environment variables с паттерном `*SECRET*`, `*KEY*`, `*TOKEN*`, (d) CloudWatch Logs Data Protection Policy — автоматическое masking sensitive patterns (AKIA-pattern, password, token) на уровне ingest в CloudWatch.

Урок: контейнер и оркестратор не «защищают» секреты автоматически. Kubernetes Secret — не зашифрованное хранилище, а API-обёртка, которая маскирует данные в `kubectl describe`, но не в файловой системе пода и логах. Реальный management секретов — external KMS + OIDC без постоянных credentials.

## Инструменты и средства контейнерной безопасности

| Класс инструмента | Назначение | Позиция в жизненном цикле | Связь со стандартами |
|-------------------|------------|---------------------------|----------------------|
| Image Vulnerability Scanner (Trivy, Grype, Clair, AWS ECR Scan) | Сканирование образов на CVE, malware, secrets в слоях, неправильные конфигурации (root user, setuid binaries) | Build (CI pipeline) + Registry (continuous scan) | CIS v8 Control 2.5 (Software Inventory), NIST SP 800-190 Section 3 |
| Admission Controller / Policy Engine (Kyverno, OPA Gatekeeper) | Property-level validation Kubernetes resources: запрет privileged pod, проверка runAsNonRoot, readOnlyRootFS, image registry whitelist | Pre-deployment: каждый API create/update request | CIS Kubernetes Benchmark 5.2, OWASP ASVS V10 (Service Security) |
| Kubernetes RBAC Audit & Analysis (KubiScan, RBAC Police, Azure Defender for Kubernetes) | Сканирование существующих RBAC-конфигураций: поиск опасных комбинаций (pods/create + pods/exec), excessive wildcard `verbs: ["*"]`, поды с default cluster-admin ServiceAccount | Continuous scan + pre-deployment review | CIS Kubernetes Benchmark 5.1 |
| Runtime Security / Threat Detection (Falco, Tetragon, Sysdig Secure) | Обнаружение аномального поведения: process spawned inside container (unexpected shell), sensitive file read (`/etc/shadow`), network connection to unknown IP, cryptomining pattern (CPU + non-standard binary execution) | Runtime: kernel event monitoring через eBPF | NIST SP 800-190 Section 4, MITRE ATT&CK Container Matrix |
| Network Policy Engine (Calico, Cilium, AWS VPC CNI Network Policies) | Микросегментация на уровне pod: разрешить web → api:8080, запретить всё остальное; L7 policies — разрешить GET /health, запретить POST /admin | Runtime: enforcement через CNI (Container Network Interface) | NIST 800-207 Zero Trust, CIS Kubernetes Benchmark 5.3.2 |
| Secrets Management (External Secrets Operator, Vault, Secrets Store CSI Driver) | Синхронизация секретов из облачных KMS/Secrets Manager в Kubernetes, без permanent storage в etcd; автоматическая ротация; инжекция секретов в pod через tmpfs volume, а не env vars | Runtime: pod startup — mount secret volume перед запуском контейнера | CIS Kubernetes Benchmark 5.3.1, NIST 800-53 IA-5 |

Runtime Security через eBPF — технологический прорыв 2020-2025. Вместо kernel module (который risky и часто incompatible с managed Kubernetes), eBPF-программы безопасно attach'ятся к kernel tracepoints и фильтруют события с near-zero overhead. Falco на eBPF: мониторинг всех `execve` вызовов внутри контейнеров, сравнение с baseline (expected processes for this container), alert на: shell spawned in NGINX container, unexpected outbound network connection, modification of system files inside container. Мetric: detection latency менее 1 секунды от системного вызова до alert в SIEM.

External Secrets Operator + CSI Driver — shift-left для секретов: секрет никогда не существует в Kubernetes статически. При старте пода, CSI Driver (Container Storage Interface) запрашивает секрет у AWS Secrets Manager / Azure Key Vault, монитрует его как tmpfs volume (только в памяти данного пода), и удаляет после termination пода. Если etcd скомпрометирован — секрета там нет. Если под скомпрометирован — злоумышленник имеет доступ к секрету на время жизни пода, но не persistent.

## Архитектурные решения

Построение защищённой контейнерной среды требует integration security на всех уровнях стека:

- **Supply-Chain (Build Pipeline):** SBOM generation (Syft) → Image Signing (Cosign) → Vulnerability Scan (Trivy) → Policy Check (Kyverno CLI) → Push to Registry. Если любой шаг fail — образ не попадает в registry.
- **Registry:** непрерывный scan existing образов (на новые CVE, обнаруженные после push), Immutable Tags (запрет перезаписи тегов — не допустить supply-chain подмену существующего `v1.2.3` новым вредоносным `v1.2.3`), Replication в отдельный security account.
- **Orchestration (Admission Control):** каждый deployment request валидируется Kyverno: проверка image signature, проверка CVE severity (CRITICAL = block без waiver), проверка Pod Security Standards (Restricted baseline), проверка Network Policy existence для namespace (enforce mandatory).
- **Runtime:** eBPF-based threat detection (Falco/Tetragon) → SIEM alert → авто-изоляция (Kyverno mutation удаляет скомпрометированный под и запрещает его перезапуск через label `quarantine: true`).
- **Networking:** default-deny Network Policy в production namespace (весь ingress/egress запрещён), затем explicit Allow Policies для каждого сервиса.

**Метрики эффективности:**
- Image Vulnerability Coverage: процент production-образов, проходящих vulnerability scan (цель: 100%)
- Critical CVE Remediation Time: от обнаружения CVE до деплоя patched образа (цель: менее 24 часов)
- Pod Security Standard Compliance: процент production-подов с Restricted PSS (цель: 100% минус легитимные exceptions)
- Network Policy Coverage: процент namespaces с active NetworkPolicy (цель: 100% production)
- Secret Static Storage Rate: доля секретов, хранящихся в Kubernetes Secrets vs внешний KMS (цель: 0% для production-critical, migration target)

**Красные флаги деградации:** увеличение числа подов с `privileged: true`; Pod Security Admission violations growth (Admission Controller блокирует, но команды пытаются circumvent); runtime detection finding «Privileged Pod Launched»; Kyverno/Admission Controller в статусе Degraded (не проверяет новые поды); network policy coverage падает ниже 95%.

**Сценарий миграции legacy monolithic Docker deployments на Kubernetes-безопасность:** этап 1 — внедрение Pod Security Standards (baseline minimum — запрет privileged). Этап 2 — внедрение Network Policy в monitor-only (логирование нарушений без блокировки, baseline traffic). Этап 3 — внедрение IRSA/OIDC federation вместо статических IAM-ключей. Этап 4 — Admission Controller с enforced Kyverno policies, включая image signature verification. Этап 5 — eBPF runtime detection с авто-реакцией для production.

## Подготовка к собеседованию: Common pitfalls & key questions

**Типичная ошибка 1: «Kubernetes Secret — защищённое хранилище секретов».**
Kubernetes Secret — base64-encoded (не шифрование), хранится в etcd в plaintext по умолчанию, доступен любому pod с RBAC-правом на Secret namespace. Это API-обёртка, а не криптографическая защита. Encryption at rest для Secrets — опциональная (EncryptionConfiguration), и без неё Secret = plaintext. Интервьюер проверяет понимание этого architectural gap.

**Типичная ошибка 2: «Docker scan на CI — coverage 100%».**
CI scan покрывает только момент push. CVE, обнаруженная после push (в библиотеке, которая была clean на момент push, но потом в ней нашли CVE) — не покрывается. Нужен также непрерывный scan registry. SBOM нужен не только для CI scan, но и для retroactive поиска: «какие из всех наших образов содержат OpenSSL 3.1.0, когда в нём нашли CVE?».

**Типичная ошибка 3: «Istio/Linkerd Service Mesh заменяет Network Policies».**
Service Mesh выполняет mTLS и L7 авторизацию на уровне сервисной identity. Network Policies — L3/L4 (IP/port) enforcement. Они комплементарны, не альтернативы. Service Mesh не блокирует raw TCP connection на kernel level — Network Policy да. CNI-level (Calico, Cilium) NetworkPolicy enforcement работает на уровне kernel eBPF/ixtables, независимо от mesh sidecar. Кандидат должен понимать layered model: Network Policy = coarse, Service Mesh = fine-grained.

**Сложный вопрос 1: «Как организовать multi-tenancy в Kubernetes-кластере, где tenant'ы (разные команды) не должны видеть поды/секреты друг друга и не должны иметь возможности lateral movement?»**
Проверяется: архитектурное понимание изоляции в Kubernetes. Ответ: (a) Namespace-level isolation — каждый tenant в своём namespace с unique ServiceAccount, RBAC ограничивает доступ к своему namespace (не cluster-wide), (b) Network Policy per namespace: default-deny ingress/egress (изоляция на сетевом уровне), (c) Resource limits per namespace (ResourceQuota — защита от DoS одним tenant'ом), (d) для strict isolation (compliance: HIPAA, PCI DSS) — отдельный node pool per tenant с nodeSelector и tolerations (tenant A только на nodes с label tenant=A), и Cilium NetworkPolicy на уровне nodes + Pod Security Context enforcement, (e) Falco custom rules per namespace. Компромисс: namespaced isolation — shared kernel, container escape одного tenant — risk для всех. Node-level isolation — kernel отделён, но дороже (dedicated nodes не утилизируются эффективно). Для regulated tenants — mandatory node-level.

**Сложный вопрос 2: «В Kubernetes-кластере 5000 подов, сотни CVE findings в день. SOC тонет. Как приоритизировать CVE по реальному риску?»**
Проверяется: понимание vulnerability prioritization beyond CVSS score. Ответ: (a) Exploitability: есть ли known public exploit (CISA KEV catalogue)? Is the vulnerability remotely exploitable or local-only? (b) Accessibility: находится ли vulnerable package в runtime-доступном слое? Если CVE в build-time tooling, который не попадает в финальный production-образ — severity снижается, (c) Reachability (eBPF-based): analysis tracing показывает, использует ли production-контейнер вообще vulnerable library/path? Если NGINX pod в production никогда не вызывает `gzip.so` — CVE in zlib, даже CRITICAL score, не reachable, (d) Business Context: какой criticality service? Payment processing (PCI DSS) vs internal blog — priority разная, (e) Compensation Controls: есть ли already active mitigation (WAF blocking specific exploit, IAM limit limiting blast radius) — снижает urgency. CIS v8 Control 7.1 (Vulnerability Management) требует контекстной приоритизации.

**Сложный вопрос 3: «Вы security architect cloud-native банка. Все production-сервисы — в Kubernetes (EKS/AKS/GKE). Регулятор требует гарантии, что разработчики не могут деплоить production-изменения без approval, и что audit trail доказывает compliance. Как реализовать?»**
Проверяется: compliance через technical controls. Ответ: GitOps-based deployment (Flux/ArgoCD) — production deploy только через Git merge, approve через mandatory Code Review от security team, CI pipeline с Kyverno CLI scan на соответствие security policies (PSS, RBAC, Network Policies, image signature). Kubernetes RBAC — разработчики не имеют `pods/create` и `deployments/update` в production namespace, только service account ArgoCD имеет deploy rights. GitOps reconciliation гарантирует: любое manual изменение в production (через `kubectl edit` разработчиком) автоматически revert'ится через 3 минуты (ArgoCD self-healing). Audit Trail: Git history (who approved what) + Kubernetes Audit Log (who deployed what) + ArgoCD deployment history (when applied). NIST SP 800-53 CM-3 (Configuration Change Control) выполняется через GitOps workflow: каждая production-изменение — auditable Git commit + automated enforcement.

## Чек-лист понимания

- [ ] Почему namespace-based изоляция в Kubernetes недостаточна для production security без Seccomp и SELinux?
- [ ] В каком случае image vulnerability scanner даёт false negative (пропускает уязвимый образ)?
- [ ] Как IRSA/OIDC federation в Kubernetes устраняет два класса угроз: статические IAM credentials и secrets sprawl?
- [ ] Почему Kubernetes RBAC с wildcard `verbs: ["*"]` и `resources: ["*"]` — security-critical anti-pattern?
- [ ] В каком случае Network Policy не блокирует lateral movement между подами, даже если default-deny configured?
- [ ] Как read-only root filesystem и non-root user митигируют container escape, даже при successful RCE exploit?
- [ ] Почему Distroless образ не является silver bullet, и какой класс атак он не закрывает?
- [ ] Как eBPF-based runtime detection (Falco/Tetragon) дополняет image scanning, и почему оба необходимы?
- [ ] В каком случае Service Mesh mTLS не обеспечивает end-to-end protection между сервисами?
- [ ] Почему admission controller (Kyverno/OPA Gatekeeper) необходим дополнительно к Pod Security Standards, и как их сочетать?

### Ответы на чек-лист

1. **Ответ:** Namespace isolation — logical separation в рамках одного kernel. Кернел-уязвимость = container escape = lateral movement во все namespaces на том же хосте. Seccomp сужает доступные syscalls (эксплойт для kernel escalation требует syscalls, которые Seccomp может блокировать). SELinux/AppArmor добавляет mandatory access control — даже root в контейнере не может получить доступ к host filesystem, потому что MAC label на уровне процесса контролирует, какие файлы доступны. NIST SP 800-190 требует defence-in-depth: namespace + MAC + seccomp + capabilities.

2. **Ответ:** (a) Vulnerability database (NVD, OSV, GitHub Advisory) задерживается: уязвимость disclosure происходит в понедельник, CVE присваивается в среду — scan во вторник видит «чистый» образ, (b) Vendor-specific packages: сканер знает upstream Ubuntu/Debian CVE, но не vendor-specific patches (AWS Linux AMI, Azure Linux), (c) Transitive dependencies через source code: библиотека, встроенная через `COPY . .` (не через package manager) — сканер не анализирует application-level dependencies (npm/pip/maven), для этого нужен Software Composition Analysis (SCA) отдельно.

3. **Ответ:** (a) Статические IAM credentials: stored in Kubernetes Secret → base64 encoded → читается любым pod с RBAC к Secret. IRSA убирает IAM-пользователя вообще: trust relationship между Kubernetes Service Account (в конкретном namespace) и IAM Role. Pod стартует, OIDC-токен (подписанный кластером) обменивается на STS temporary credentials (1 час) — ключей в кластере нет. (b) Secrets sprawl: при статических credentials, каждые 90 дней надо ротировать ключ и обновить во всех pods → Secrets → CI/CD — огромное operational surface. IRSA убирает lifecycle: temporary credentials автоматически expire. CIS v8 Control 16.11 (Secrets Management) и NIST 800-53 IA-5 требуют этого подхода.

4. **Ответ:** wildcard RBAC даёт subject неограниченный доступ ко всем resources в scope (namespace/cluster), включая: `secrets/*` (читать все секреты), `pods/exec` (exec в любой под), `roles/create` (создать новую роль с ещё большим доступом — privilege escalation), `persistentvolumes/*` (монтировать чужой volume с sensitive данными). Cluster-admin wildcard — game over при компрометации пода. CIS Kubernetes Benchmark 5.1.3 требует принципа least privilege: каждый RBAC role явно перечисляет resources и verbs. Kyverno policy должна блокировать wildcard `"*"` в production namespace.

5. **Ответ:** (a) Network Policy работает на L3/L4 (IP/port). Если под-злоумышленник использует разрешённый порт (например HTTPS 443) к целевому поду, который слушает HTTPS, — Network Policy пропускает. (b) DNS-based exfiltration: под отправляет DNS-запросы с encoded данными на external DNS server (порт 53 UDP, который обычно разрешён для DNS). Network Policy видит «разрешённый DNS-трафик», а не exfiltration. Для L7 detection нужен Service Mesh, DNS monitoring или eBPF. (c) CNI plugin bug — Network Policy configured but not enforced (misconfigured Calico/Cilium).

6. **Ответ:** RCE exploit — злоумышленник запускает shell (или reverse-shell) в контейнере. Read-only root FS: атакующий не может записать exploit binary/script на диск контейнера, не может модифицировать `.bashrc`, cron jobs, system configuration files. Но атакующий может (a) выполнять in-memory-only операции (запуск reverse-shell через `/dev/tcp`), (b) записывать в `tmpfs` `/tmp` (если монтирован), (c) использовать /dev/shm (shared memory) как временное хранилище. Non-root user: добавляет ограничения — атакующий не может bind на привилегированные порты (<1024), не может установить setuid бит, не может запустить `iptables`, не может читать файлы с `---`, владельцем root. Это defence-in-depth, не silver bullet. NIST SP 800-190 Section 4 требует комбинации.

7. **Ответ:** Distroless убирает shell и package manager — злоумышленник, получивший RCE, не может интерактивно `bash` и скачивать tools через `apt-get`. Но он не закрывает: (a) kernel exploits (container escape) — distroless не влияет на доступные syscalls; (b) reverse-shell — злоумышленник может создать собственный TCP-соединение на внешний сервер (даже без shell, через `python -c` если Python в образе) — distroless не убирает Python; (c) memory corruption attacks (ROP chains) — distroless не влияет. Это defence against post-exploitation convenience, не против детерминированных атак.

8. **Ответ:** Image scanning — pre-deployment check: образ с CVE не попадает в production (block). Но если CVE обнаружена после deployment (zero-day), или scanning failed (false negative), контейнер в production с уязвимостью. Runtime detection — post-deployment: Falco/Tetragon мониторит actual behaviour and detects exploitation — process spawned where it shouldn't, kernel module loaded, sensitive file opened. Если scanning failed (пропустил уязвимость), но Falco работает (runtime detection) — инцидент обнаруживается на стадии exploitation, до damage (data exfiltration completed). Оба слоя mandatory — shift-left + shift-right.

9. **Ответ:** mTLS между sidecar-прокси шифрует connection и аутентифицирует identity caller'а service. Но прокси terminates TLS — трафик внутри sidecar container (между прокси и приложением) — plaintext loopback (localhost). Если скомпрометирован sidecar-прокси (уязвимость в Envoy/Linkerd), злоумышленник может читать plaintext трафик до того, как он зашифрован исходящим прокси. End-to-end protection требует application-level encryption (приложение само шифрует payload до отправки sidecar'у), что редко реализуется из-за сложности. NIST SP 800-207 и OWASP ASVS V10 требуют protection in transit but note that mesh proxying — not end-to-end encryption on application layer.

10. **Ответ:** Pod Security Standards (PSS) — built-in admission control проверяет базовые Pod Security Context параметры (privileged, hostNetwork, capabilities, runAsNonRoot) по трём уровням: Privileged, Baseline, Restricted. Он ограничен predefined checks — нельзя проверять custom условия («разрешить image из registry X только для namespace Y», «запретить env vars с `*SECRET*` pattern»). Kyverno/OPA Gatekeeper — programmable policy engine: пишется Rego/CEL policy на любые параметры resources, включая labels/annotations, image metadata, RBAC rules. Комбинация: PSS как mandatory baseline (быстрый, встроенный, low overhead), Kyverno как дополнение для custom organizational policies. CIS Kubernetes Benchmark рекомендует использовать both.

## Ключевые выводы для собеседования

- **Самый критичный принцип:** Контейнер — не boundary security. Shared kernel означает, что каждый pod в кластере — потенциальная точка escape к хосту и ко всем соседним подам. Defence-in-depth (non-root, read-only FS, Seccomp, SELinux, Drop Capabilities) mandatory, не optional.
- **Главный компромисс:** Чем сильнее runtime изоляция (gVisor, Firecracker microVM), тем больше overhead (latency, resource usage, image compatibility) — баланс зависит от threat model. Для multi-tenant SaaS — mandatory; для single-tenant internal — Seccomp + SELinux достаточно.
- **Связь с ключевым стандартом:** NIST SP 800-190 (Container Security) — comprehensive framework, покрывающий image, registry, orchestrator, container, host levels. CIS Kubernetes Benchmark — практическая операционализация в measurable checks. Executive Order 14028 и CIS v8 Control 2.5 требуют SBOM.
- **Практический императив:** Если в production Kubernetes-кластере есть хотя бы один pod с `runAsNonRoot: false` или без Network Policy — lateral movement после компрометации trivial. Каждый production deployment должен проходить mandatory Kyverno/OPA validation.

---
_Статья создана на основе анализа NIST SP 800-190 (Container Security), CIS Kubernetes Benchmark v1.8, OWASP Top 10 CI/CD Security (2022), MITRE ATT&CK Container Matrix, официальной документации Kubernetes (Pod Security Standards, RBAC, Network Policies), Docker Security documentation, материалов облачных провайдеров (AWS EKS Security Guide, Azure AKS Security, GCP GKE Hardening Guide), публичных CVE (runc CVE-2019-5736, Log4Shell CVE-2021-44228) и open-source инструментов (Trivy, Falco, Kyverno, Cosign)._
