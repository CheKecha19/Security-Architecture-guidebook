# Статья #02: Эволюция Docker — от dotCloud до AI-платформы

**Трек:** 01 — История и эволюция  
**Дата:** 2026-05-13  
**Статус:** Готово

---

## TL;DR

За 13 лет Docker прошёл путь от внутренней утилиты парижского PaaS-стартапа до глобальной платформы для сборки, доставки и запуска ПО. Эволюция Docker — это три тектонических сдвига: **монолит → модульная архитектура** (разделение на runC, containerd, Moby), **инструмент разработчика → платформа для enterprise** (Docker Desktop, Docker Hub, Docker Scout), и **контейнеры общего назначения → AI-native runtime** (Model Runner, GPU-ускорение, агентные приложения). Каждый этап был ответом на рыночные угрозы — Kubernetes, Podman, облачные managed-сервисы — и одновременно расширением видения того, чем может быть контейнерная платформа.

---

## 1. Три эпохи Docker: мета-структура эволюции

Историю Docker невозможно понять как плоский список версий. Это три качественно разных эпохи, каждая со своей бизнес-моделью, архитектурными принципами и набором конкурентов.

| Эпоха | Годы | Характер | Ключевой конфликт |
|-------|------|---------|-------------------|
| **Эпоха I: Становление** | 2013–2016 | Docker — это CLI-утилита + демон. Контейнеры для разработчиков | Борьба с legacy (VM, bare metal), захват рынка |
| **Эпоха II: Модуляризация** | 2016–2019 | Docker — экосистема компонентов. OCI-стандарты, оркестрация | Docker Swarm vs Kubernetes, монолит vs микросервисы |
| **Эпоха III: Платформа** | 2019–н.в. | Docker — платформа для всего цикла разработки. AI/ML-нагрузки | Удержание разработчиков vs managed-сервисы облаков, монетизация open-source |

Каждая смена эпохи сопровождалась кризисом идентичности. Docker, Inc. трижды переопределяла, чем она является: PaaS-компания → контейнерная компания → платформа для разработчиков с AI-амбициями.

---

## 2. Эпоха I: Становление (2013–2016)

### 2.1 Дорожная карта, которой не было

Первые три года у Docker не было формального roadmap. Команда Solomon Hykes работала по принципу «разработчики просят → мы делаем». Это дало взрывной рост, но создало проблему: к 2016 году Docker Engine стал монолитным binary, в который было вкомпилировано всё — сетевой стек, оркестрация, управление томами, сборка образов, рантайм.

**Это было одновременно силой и слабостью.** Сила: `docker run` решал всё. Слабость: нельзя было заменить один компонент без замены всего Docker.

### 2.2 Ключевые релизы эпохи становления

#### v0.1 — Март 2013: первый публичный релиз

- Основан на LXC как execution driver
- Поддерживает AUFS для слоёв образов
- CLI: `docker run`, `docker pull`, `docker push`, `docker build`
- Запуск — несколько секунд против минут у VM

На этом этапе Docker был надстройкой над LXC. Без LXC он не работал. Это было и преимущество (быстрый старт — LXC уже существовал) и ограничение.

#### v0.9 — Март 2014: отказ от LXC, рождение libcontainer

**Ключевой момент архитектурной независимости.** Docker перестаёт быть «обёрткой над LXC» и становится самостоятельной контейнерной технологией.

```go
// libcontainer давал прямой доступ к namespaces API ядра
// без посредника в виде LXC
ns := &configs.Namespace{
    Type: configs.NEWPID,  // PID namespace — изоляция процессов
}
```

Почему отказались от LXC:
- **Зависимость от версий ядра** — LXC требовал конкретной комбинации патчей
- **Проблемы на разных дистрибутивах** — Ubuntu работал, CentOS — нет
- **Ограниченный контроль** — Docker не мог донастроить изоляцию так, как хотел

libcontainer был написан на Go (переписали с Python — ещё одно архитектурное решение). Это дало:
- Единый бинарник без внешних зависимостей
- Полный контроль над namespaces и cgroups
- Кроссплатформенность (Go компилируется везде)

Позже libcontainer станет runC — OCI-стандартным рантаймом.

#### v1.0 — Июнь 2014: production readiness

Версия 1.0 стала символом: «Docker готов к production». Что это значило на практике:

- Стабильный API (обратная совместимость отныне — обещание)
- Поддержка нескольких storage-драйверов: AUFS, devicemapper, btrfs, overlay
- Docker Hub как продакшен-сервис (не бета)
- Документация уровня enterprise

> **Аналогия:** v1.0 для Docker — как Windows 3.0 для Microsoft. Не первый продукт, но первый, который сказал рынку: «Это не игрушка, это серьёзно».

#### v1.6 — Апрель 2015: Docker Compose (тогда ещё fig)

Docker Compose родился из инструмента **fig**, созданного компанией Orchard. Docker, Inc. приобрела Orchard в июле 2014 года, и fig стал Docker Compose.

```yaml
# docker-compose.yml — декларативное описание multi-container приложений
version: '3'
services:
  web:
    build: .
    ports:
      - "5000:5000"
  redis:
    image: "redis:alpine"
```

Это был качественный скачок: от «запусти один контейнер» к «опиши всю инфраструктуру приложения в одном файле».

#### v1.9 — Ноябрь 2015: Docker Network (libnetwork)

До 1.9 сеть контейнеров была примитивной: link-команды, которые были deprecated почти сразу после появления. v1.9 принесла:

- Multi-host overlay network
- Plugin-based драйверы (bridge, overlay, macvlan)
- Встроенный DNS для service discovery внутри сети

#### v1.12 — Июль 2016: Swarm Mode — оркестрация внутри Docker

**Самый амбициозный релиз — и начало великого конфликта Docker vs Kubernetes.**

Docker 1.12 добавил **Swarm Mode** — встроенную оркестрацию прямо в Docker Engine:

```bash
# Три команды — и у тебя кластер
docker swarm init                    # инициализация swarm
docker swarm join --token ...        # добавить worker
docker service create --replicas 3   # масштабирование
```

Это была гениальная в своей простоте идея: **Docker сам оркестрирует контейнеры, без Kubernetes.** Для малого и среднего бизнеса, для разработчиков, которым не нужна сложность K8s.

Преимущества Swarm Mode:
- **Нулевая конфигурация:** `docker swarm init` → кластер готов
- **Docker-native CLI:** никаких новых команд — всё через привычный `docker`
- **Встроенная безопасность:** mutual TLS между нодами автоматом, ротация сертификатов
- **Rolling updates:** `docker service update --image` — обновление сервиса без downtime
- **Service discovery:** встроенный DNS, контейнеры находят друг друга по имени сервиса

Недостатки, которые решили исход битвы:
- **Слабая экосистема плагинов** — Kubernetes быстро оброс Helm-чартами, операторами, CRD
- **Нет поддержки stateful workloads** — StatefulSets в K8s были проработаны лучше, чем Docker volumes в Swarm
- **Масштабирование** — Swarm начинал испытывать проблемы на 1000+ нодах, Kubernetes держал 5000+
- **Community** — Google, Red Hat, CoreOS, Weaveworks, Mesosphere — все пошли за Kubernetes. Docker остался с маленьким лагерем

Проблема: к этому моменту Kubernetes уже набирал обороты. Google, Red Hat, CoreOS — все делали ставку на Kubernetes. Docker сделал ставку на Swarm. Рынок выбрал Kubernetes.

**Почему Docker проиграл битву оркестраторов, но не проиграл войну?** Потому что к 2018 году Docker принял реальность: Kubernetes — стандарт оркестрации, Docker — стандарт рантайма и developer experience. Swarm не умер — он до сих пор поддерживается и используется для небольших кластеров. Но стратегически Docker перестал позиционировать его как конкурента Kubernetes.

---

## 3. Эпоха II: Модуляризация (2016–2019)

### 3.1 Зачем Docker начал «разбирать себя на запчасти»

К 2016 году стало очевидно: экосистема контейнеров не может принадлежать одной компании. Kubernetes использовал Docker как рантайм, но хотел более тонкого контроля. Red Hat хотела стандартов. Сообщество требовало совместимости.

Docker принял решение: **открыть архитектуру, стандартизировать компоненты, отдать контроль открытым организациям.** Это был риск — потерять монополию — но и единственный путь выжить в экосистеме, где Kubernetes становился стандартом де-факто.

### 3.2 2015 — Open Container Initiative (OCI)

Июнь 2015. Docker, CoreOS, Google, Microsoft, Amazon, Red Hat, IBM — 21 компания создают **Open Container Initiative** под эгидой Linux Foundation.

**Миссия OCI:** создать открытые стандарты для контейнерных runtime и форматов образов. Два главных стандарта:

| Стандарт | Что описывает |
|----------|-------------|
| **OCI Runtime Specification** | Как запускать контейнер (config.json + rootfs = container) |
| **OCI Image Specification** | Как упаковывать образ (manifest, config, layers) |

Docker передал в OCI:
- **runC** — эталонная реализация runtime-спецификации (вырос из libcontainer)
- Формат Docker Image v2.2 — стал основой OCI Image Spec

> **Аналогия:** OCI — это как ISO для контейнеров. Раньше каждый производитель делал «лампочки» (контейнеры) со своим цоколем. OCI сказал: «Вот стандартный цоколь E27 — теперь любая лампочка подходит к любому патрону».

### 3.3 2016 — containerd: runtime-менеджер

Декабрь 2016. Docker выделяет **containerd** — новый компонент, управляющий жизненным циклом контейнеров:

```
┌─────────────────────────┐     Docker до 2016:
│  dockerd (монолит)      │     всё в одном binary
│  ├─ сборка              │
│  ├─ оркестрация         │
│  ├─ сеть                │
│  ├─ containerd (новый)  │
│  └─ runC                │
└─────────────────────────┘

     Docker после 2016:

┌──────────────┐
│  dockerd     │  ← только CLI API + оркестрация (Swarm)
└──────┬───────┘
       │ gRPC
┌──────▼───────┐
│  containerd  │  ← pull/push образов, управление контейнерами, снапшоты
└──────┬───────┘
       │ OCI Runtime API
┌──────▼───────┐
│    runC      │  ← создание и запуск контейнера (namespaces, cgroups)
└──────────────┘
```

containerd стал **прослойкой между dockerd и runC**, управляя:
- Pull/push образов
- Снапшотами (snapshots) — containerd-версия слоёв файловой системы
- Жизненным циклом: create → start → stop → delete
- Передачей задач в runC

### 3.4 2017 — containerd переходит в CNCF

Март 2017. Docker передаёт containerd в **Cloud Native Computing Foundation (CNCF)** — организацию под Linux Foundation, которая также управляет Kubernetes.

Это был стратегический ход:
- **Kubernetes нужен был CRI-совместимый рантайм** (Container Runtime Interface — стандартный способ для Kubernetes взаимодействовать с контейнерным рантаймом)
- containerd стал первым не-Docker-способом запускать контейнеры под Kubernetes
- Docker показал, что он — часть экосистемы, а не монополист

В 2019 году containerd достигает статуса **Graduated** в CNCF — высший уровень зрелости, наряду с Kubernetes, Prometheus, Envoy.

### 3.5 2017 — Moby Project: Docker разбирает себя

Апрель 2017. Docker анонсирует **Moby Project** — фреймворк для сборки контейнерных систем из компонентов.

**Что случилось:** репозиторий `docker/docker` (огромный монолит на 72K звёзд на GitHub) разделяют на:
- **moby/moby** — «Lego set» компонентов: движок, CLI, билдер
- Десятки отдельных репозиториев: `linuxkit`, `containerd`, `runc`, `buildkit`, `swarmkit`…

Docker Engine становится **одним из продуктов, собранных из Moby-компонентов** — не единственным.

> **Аналогия:** Docker разобрал монолитный автомобиль на запчасти. Moby — это каталог запчастей. Docker Engine — это конкретная модель, собранная из этих запчастей. Любой другой может собрать свою «модель» — контейнерную систему со своим набором компонентов.

Для пользователей Docker практически ничего не изменилось — `docker run` работал как прежде. Но для экосистемы это был сигнал: **Docker больше не «чёрный ящик» — это набор открытых, заменяемых компонентов.**

### 3.6 2017 — Новая схема версионирования: YY.MM

С версии 1.13 Docker переходит на календарное версионирование (CalVer): `YY.MM.patch`.

- **17.03** (март 2017) — первый релиз по новой схеме
- **17.06** (июнь 2017) — разделение CE (Community Edition) и EE (Enterprise Edition)

| Версия | Формат | Схема |
|--------|--------|-------|
| 1.0 — 1.13 | SemVer-like | Главная.Минорная.Патч |
| 17.03 — 25.01 | CalVer | Год.Месяц (стабильные ветки) |
| 26.0 — 29.x (2025+) | SemVer | Возврат к семантическому версионированию |

Переход на CalVer совпал с разделением CE/EE:

- **Docker CE (Community Edition):** бесплатный для всех, открытый код, community-поддержка
- **Docker EE (Enterprise Edition):** платный, сертифицированные плагины, enterprise-поддержка, Docker Datacenter (позже — Docker Enterprise Platform)

Это была попытка монетизации: открытый код для разработчиков, платная версия для enterprise'ов, которым нужны SLA и сертификации.

### 3.7 Docker Compose: от fig до Compose v2

История Docker Compose — это микрокосм эволюции Docker. Три фазы одного инструмента.

#### Фаза 1: fig (2013–2014)

**fig** — инструмент компании Orchard для описания multi-container окружений. YAML-файл, одна команда для подъёма всего стека.

```yaml
# fig.yml — прообраз будущего docker-compose.yml
web:
  build: .
  ports:
    - "5000:5000"
  links:
    - redis
redis:
  image: redis
```

В июле 2014 Docker, Inc. приобретает Orchard за нераскрытую сумму. fig становится частью Docker.

#### Фаза 2: Docker Compose v1 (2014–2020)

fig переименован в **Docker Compose**, переписан на Python:

```bash
# Установка — отдельный Python-пакет:
pip install docker-compose

# Запуск — отдельный binary:
docker-compose up -d
```

Ключевые возможности Compose v1:
- Поддержка нескольких файлов (docker-compose.override.yml)
- Переменные окружения (${VARIABLE})
- Масштабирование: `docker-compose up --scale web=3`
- Сети и тома в Compose-файле

Проблемы:
- Python-зависимости — venv, конфликты версий, медленный старт
- Отдельный binary — `docker-compose` vs `docker` — рассинхрон версий
- Нет GPU-поддержки

#### Фаза 3: Docker Compose v2 (2020–н.в.)

Июль 2020. Docker Compose v2 — полная переработка:

```bash
# Установка — плагин к Docker CLI (никакого pip):
docker compose  # это plugin, не отдельный binary
```

Преимущества:
- **Скорость:** Go вместо Python, запуск мгновенный
- **Интеграция:** `docker compose` — часть `docker` CLI, единый API
- **GPU-поддержка:** `deploy.resources.reservations.devices` для доступа к GPU внутри Compose
- **Compose Watch (2024):** автоматическая синхронизация изменений кода с контейнером — без пересборки образа
- **Compose Build Specification (2025):** унификация синтаксиса сборки, совместимость с BuildKit

Compose v2 — не просто «версия 2». Это **архитектурная смена платформы**: инструмент переехал из экосистемы Python в экосистему Go, став нативным компонентом Docker CLI.

### 3.8 Эволюция Docker на Windows и Mac

Отдельная сюжетная линия — Docker на неродных платформах.

#### Docker на Mac

- **2014–2016: Docker Toolbox** — VirtualBox + boot2docker (мини-Linux VM). Медленно, сложно, fsync-ад.
- **2016: Docker for Mac** — нативный xhyve (позже HyperKit) гипервизор Apple. Файловая синхронизация osxfs.
- **2018: gRPC FUSE (mutagen) синхронизация** — скорость работы с mounted volumes выросла в разы.
- **2020: Apple Silicon (M1)** — нативная ARM64-поддержка, пересборка 90% образов под aarch64.
- **2023: VirtioFS** — замена osxfs, почти-native производительность файловых операций.

#### Docker на Windows

- **2014: Docker Toolbox** — VirtualBox. «Зачем ты мучаешь себя?» — стандартный вопрос на конференциях.
- **2016: Docker for Windows** — Hyper-V, только Pro/Enterprise. Windows-контейнеры рядом с Linux-контейнерами.
- **2019: WSL2 backend** — революция. Ядро Linux внутри Windows, Docker работает почти как на bare-metal Linux.
- **2020: Windows Home поддержка** — WSL2 позволил Docker Desktop работать на Windows 10/11 Home (раньше только Pro).
- **2024: GPU passthrough** — Docker Desktop под WSL2 получил сквозной доступ к GPU (CUDA, DirectML).

**Значение:** Без поддержки Mac и Windows Docker не стал бы «инструментом каждого разработчика». Большинство разработчиков сидят на Mac/Windows, а продакшен — на Linux. Docker Desktop решил эту дихотомию.

---

## 4. Эпоха III: Платформа и AI (2019–2026)

### 4.1 2019 — Продажа enterprise-бизнеса Mirantis

Ноябрь 2019. **Тектонический сдвиг:**

- Docker, Inc. продаёт **Docker Enterprise Platform** компании Mirantis
- Увольняет часть команды, привлекает $35M инвестиций
- Назначает нового CEO — Scott Johnston (ранее COO Docker)

**Что осталось у Docker, Inc.:**
- Docker Engine (open-source ядро, moby/moby)
- Docker Desktop (для Mac/Windows)
- Docker Hub (реестр образов)
- Docker Compose

**Что ушло Mirantis:**
- Docker Enterprise Engine (сейчас Mirantis Container Runtime)
- Docker Trusted Registry
- Docker Universal Control Plane (UI для управления кластерами)
- Команда, ответственная за enterprise-продажи и enterprise-поддержку

Это был болезненный, но необходимый pivot. Docker понял: **война оркестраторов проиграна** — Kubernetes победил. Будущее Docker — не в конкуренции с Kubernetes, а в том, чтобы быть лучшим инструментом для разработчиков, которые запускают код (в том числе в Kubernetes).

**Финансовый контекст:** К 2019 году Docker, Inc. привлекла ~$270M венчурных инвестиций, но так и не вышла на прибыльность. Enterprise-продажи приносили ~$30M/год — капля в море по сравнению с затратами на разработку двух платформ (Engine и Enterprise). Продажа Mirantis позволила сфокусироваться на разработчиках и сократить burn rate.

Scott Johnston переопределил миссию: **«Docker помогает разработчикам быстро превращать идеи в код, работающий везде.»** Никакой оркестрации, никаких дата-центров — фокус на developer experience.

### 4.2 2020–2021: COVID, удалёнка и бум Docker

Пандемия COVID-19 неожиданно стала катализатором для Docker:

- **Удалённая разработка** — миллионы разработчиков перешли на домашние машины. Docker Desktop стал стандартом для локальной разработки.
- **Инфраструктура как код** — DevOps-инженеры, запертые дома, переводили всё в контейнеры.
- **Рост Docker Hub** — pull'ы выросли на 40% за 2020 год.

Это дало Docker финансовую подушку для перехода на freemium-модель (2021).

### 4.3 2018–2020 — BuildKit: революция сборки

**BuildKit** — проект, начатый Docker в 2017 году как часть Moby. К 2018 году стал доступен как экспериментальный backend для `docker build`, к 2020 — стабилен, а к 2023 стал единственным builder'ом Docker.

**Зачем потребовалась замена:** legacy builder читал Dockerfile и выполнял инструкции последовательно — шаг за шагом. Но в мире микросервисов с сотнями образов, multi-arch сборками (ARM для Mac M1 + x86 для прода), security scanning и CI/CD это было узким местом.

| Legacy builder | BuildKit |
|----------------|----------|
| Последовательная сборка | Параллельное исполнение независимых стадий (DAG — направленный ациклический граф) |
| Кэш только по слоям | Content-based кэширование (хеш входных данных слоя определяет кэш-хит) |
| Нет секретов в build-time | `--secret` — проброс секретов на время сборки без сохранения в конечном слое |
| Нет SSH-проброса | `--ssh` — доступ к SSH-ключам хоста для `git clone` приватных репозиториев |
| Одна платформа за раз | Multi-platform сборка (linux/amd64, linux/arm64, windows/amd64) за один вызов |
| Кэш локальный | Кэш-экспорт в registry — общие кэши для всех CI/CD-раннеров |
| Нет инкрементальной сборки | Пересборка только изменившихся слоёв с точностью до файла |

BuildKit стал бэкендом для `docker buildx` — CLI-плагина:

```bash
# Полноценная multi-platform сборка с общим кэшем и секретами:
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --secret id=npm_token,src=./.npmrc \
  --cache-to type=registry,ref=myrepo/cache:latest,mode=max \
  --cache-from type=registry,ref=myrepo/cache:latest \
  --output type=image,push=true \
  -t myrepo/myapp:latest .
```

BuildKit стал де-факто стандартом сборки — и не только для Docker. Kubernetes-билдеры (Kaniko, Tekton) перешли на BuildKit-подход, а сам BuildKit можно запустить как standalone-сервис сборки без Docker Engine вообще.

#### Docker Bake (2023+) — сборка сложных проектов

Для проектов с десятками образов появился **Docker Bake**:

```hcl
// docker-bake.hcl — описание сборочного графа
group "default" {
  targets = ["api", "worker", "frontend"]
}

target "api" {
  context = "./api"
  platforms = ["linux/amd64", "linux/arm64"]
  args = { BUILDKIT_INLINE_CACHE = "1" }
}

target "worker" {
  inherits = ["api"]  // переиспользование всей конфигурации api
  context = "./worker"
}
```

Bake решает проблему «у нас 50 Dockerfile'ов и правильный порядок сборки с общим кэшем». CI/CD превращается в одну команду: `docker buildx bake`.

### 4.3 Docker Desktop: от бесплатного к Personal/Pro

**2021 — перелом в монетизации:**

- Docker Desktop перестаёт быть бесплатным для enterprise (компании >250 сотрудников или >$10M дохода)
- Появляются планы: **Personal** (бесплатный для небольших команд), **Pro** ($5/мес), **Team** ($7/мес), **Business** ($21/мес)

Это вызвало волну негодования — разработчики привыкли, что Docker Desktop бесплатный. Docker ответил: «Docker Engine на Linux остаётся бесплатным всегда. Desktop — это продукт с GUI, который мы развиваем отдельно».

И действительно: Docker Desktop за несколько лет превратился из простого GUI для запуска контейнеров в:

- **WSL 2-интеграция** (Windows) — производительность близка к bare Linux
- **Встроенный Kubernetes** — одной кнопкой запустить K8s-кластер локально
- **Docker Extensions** — marketplace плагинов (Portainer, Snyk, JFrog)
- **Dev Environments** — Git-репозитории, автоматически запускающиеся в контейнере

### 4.4 Docker Scout (2023): Software Supply Chain Security

Октябрь 2023. Docker запускает **Docker Scout** — анализ уязвимостей и состава образов (SBOM — Software Bill of Materials).

```
$ docker scout quickview myimage
✓ SBOM of image already cached, 1.3s
  ✓ No vulnerable packages detected
```

Docker Scout анализирует:
- **CVE** в пакетах внутри образа
- **Рекомендации по обновлению** базового образа
- **SBOM** — полный список компонентов (для compliance)

Это ответ на рост атак через цепочку поставок (вспомни log4shell, xz backdoor): Docker позиционирует себя как инструмент безопасности для разработчиков, а не просто «запускалку контейнеров».

### 4.5 Docker init (2023): автоматическая докеризация

Октябрь 2023 — ещё одна веха упрощения: команда `docker init`.

```bash
# В любой директории с проектом:
docker init

# Docker анализирует проект (package.json, requirements.txt, go.mod)
# и генерирует Dockerfile, compose.yaml, .dockerignore
```

Проблема «как написать правильный Dockerfile» существовала 10 лет. `docker init` — это как Copilot для докеризации: Docker сам понимает, какой язык/фреймворк, и генерирует best-practice конфигурацию.

### 4.6 Docker + AI/ML (2024–2026): новая идентичность

С 2024 года Docker активно перестраивается под AI/ML-нагрузки. Это отдельный вектор эволюции, заслуживающий детального разбора.

#### Docker + NVIDIA (Март 2024)

Docker анонсирует стратегическое партнёрство с NVIDIA:

- **GPU-доступ внутри контейнеров** через `--gpus` (Docker Desktop + WSL2)
- Интеграция с NVIDIA Container Toolkit
- Образы с предустановленными CUDA, cuDNN, TensorRT

```bash
# Запуск контейнера с доступом к GPU:
docker run --gpus all nvidia/cuda:12.4.0-runtime-ubuntu22.04 nvidia-smi
```

#### Docker Model Runner (Апрель 2025)

Самый амбициозный AI-релиз Docker. **Model Runner** — возможность управлять AI-моделями прямо через Docker CLI.

```bash
# Pull модели как образа:
docker model pull ollama/llama3:8b

# Запуск модели в изолированном окружении:
docker model run ollama/llama3:8b "Explain Docker layers"

# Список доступных моделей:
docker model ls

# Запуск модели с GPU-ускорением:
docker model run --gpus all nvidia/mistral:7b --serve
```

Модели хранятся в Docker Hub как OCI-артефакты. OCI-формат образов расширен для хранения весов моделей (GGUF, safetensors). Это превращает Docker Hub в **реестр AI-моделей**, а Docker Engine — в **рантайм для ML-инференса**.

**Что это даёт разработчику:** До Model Runner: скачай Ollama, настрой, привяжи к Docker-сети, пробрасывай порты, решай конфликты GPU-памяти. После Model Runner: `docker model pull` → `docker model run`. Модель запущена в изолированном контейнере с GPU-доступом.

**Безопасность:** Модели в Docker Hub подписываются через Docker Content Trust (Notary), цепочка поставок верифицируется — разработчик знает, что модель не подменена.

#### Docker Model Runner: внутреннее устройство

Под капотом Model Runner — это не «просто контейнер с моделью». Это специализированный containerd shim для AI-нагрузок:

```
┌──────────────┐
│ docker model │  ← CLI: pull, run, ls, rm
└──────┬───────┘
       │
┌──────▼──────────────────────┐
│ dockerd                      │
└──────┬──────────────────────┘
       │
┌──────▼──────────────────────┐
│ containerd                   │
│  ├─ model-shim (новый!)      │  ← OCI-артефакты, GGUF/safetensors парсинг
│  ├─ runc                     │  ← обычные контейнеры
│  └─ wasm-shim                │  ← Wasm-модули
└─────────────────────────────┘
```

`model-shim` делает: парсит OCI-артефакт (манифест, веса, конфиг инференса), выбирает рантайм (llama.cpp, vLLM, TensorRT-LLM) в зависимости от формата модели, аллоцирует GPU-память через nvidia-container-toolkit, expose'ит OpenAI-совместимый API endpoint для совместимости с LangChain, LlamaIndex, CrewAI.

#### Docker + GenAI Stack (2024)

Июль 2024. Docker публикует серию статей «Docker Labs GenAI» и запускает **GenAI Stack** — преднастроенный шаблон для RAG (Retrieval-Augmented Generation): LLM + векторная БД + приложение, всё в одном Compose-файле. Разработчик поднимает полный AI-стек одной командой `docker compose up`.

#### Agentic AI (2025–2026)

Docker запускает **Docker Agent** — среду для запуска AI-агентов в контейнерах.

Ключевая инновация: **контейнерная песочница для AI-агентов.** Каждый агент — контейнер или Compose-группа контейнеров с ограниченной файловой системой, сетевым доступом и выполнением кода. Это решает главную проблему agentic AI: ты не запускаешь AI-агента на своей машине с полными правами — ты запускаешь его в контейнере, где он может сломать только самого себя.

> **Стратегия:** Docker хочет быть для AI-приложений тем же, чем был для микросервисов в 2015 — стандартным способом «упакуй и запусти».

### 4.7 Docker 20.10–29.x: стабилизация и инновации

#### Docker Engine 20.10 (декабрь 2020)

Знаковый LTS-релиз (поддерживался до декабря 2023):

- **Rootless mode:** запуск Docker-демона без root-прав (experimental → stable). Критически важный ответ на растущую популярность Podman, который с самого начала продвигал rootless-контейнеры как архитектурное преимущество.
- **cgroup v2:** полноценная поддержка — необходимо для современных дистрибутивов (Fedora 31+, Ubuntu 21.10+). Без этого Docker не запускался на свежих версиях.
- **Dual logging:** драйверы могут писать в несколько destination одновременно (loki + json-file, например)
- **BuildKit по умолчанию:** переменная `DOCKER_BUILDKIT=1` больше не нужна — BuildKit стал стандартным builder'ом

#### Docker Engine 23.0 (февраль 2023)

- **Полный отказ от legacy builder:** `docker build` теперь всегда использует BuildKit. Legacy builder убран из кодовой базы.
- **SBOM и provenance** — встроенная аттестация образов (SLSA Build Level 3, подписи через Sigstore Cosign)
- **containerd 1.6** — интеграция с новейшим containerd, стабильность, исправления утечек памяти

#### Docker Engine 26.0 (март 2024)

- **Полный возврат к SemVer** — после 6 лет календарного версионирования (с 17.03 по 25.0) Docker возвращается к семантическому. Сообщество просило — Docker сделал.
- **Wasm-поддержка (beta):** containerd shim для запуска WebAssembly-модулей. Wasm-«контейнер» стартует за миллисекунды и потребляет минимум памяти — идеально для serverless/FaaS.
- **docker init** стабилен — полностью готов для production-использования

#### Docker Engine 29.0 (2026)

Текущая актуальная версия (29.4.2 на май 2026):

- **Model Runner stable** — AI-модели (LLM, embedding, image-gen) как первоклассные объекты Docker: pull, run, serve, list
- **Wasm stable** — запуск Wasm-модулей наравне с обычными контейнерами, поддержка WASI Preview 2
- **LTS-политика:** официально объявленные LTS-ветки (минимум 2 года поддержки) для enterprise-пользователей
- **Build attestation по умолчанию:** каждый образ при сборке автоматически получает SBOM (список компонентов) и provenance-аттестацию (SLSA Build Level 3) — для compliance и security audit'ов

---

## 5. Эволюция Docker Hub: от реестра образов к marketplace

Docker Hub прошёл путь от простого реестра образов в 2013 до глобальной платформы распространения контента в 2026. Его эволюция отражает смену бизнес-модели Docker: от «бесплатной утилиты» к «платформе с платными сервисами».

| Этап | Год | Что было |
|------|-----|---------|
| **v1: Registry** | 2013 | Pull/push образов через HTTP API. Никакого веб-интерфейса — только CLI |
| **v2: Docker Hub** | 2014 | Веб-интерфейс, automated builds (GitHub-интеграция), организации, приватные репозитории |
| **v3: Trusted content** | 2017 | Official Images (проверенные Docker, Inc.), Verified Publishers (проверенные вендоры — Microsoft, Oracle, MongoDB), Docker Certified (сертифицированные для Docker EE) |
| **v4: Marketplace** | 2020 | Docker Extensions marketplace, rate limiting на бесплатные pull'ы (100 запросов/6ч для анонимов), Pro/Team/Business планы |
| **v5: AI Hub** | 2025 | Model Runner — AI-модели (Llama, Mistral, Gemma) как OCI-артефакты. GPU-образы (NVIDIA CUDA, TensorRT). Docker Hub стал AI model registry |

В 2020 году Docker Hub ввёл **rate limiting** — 100 бесплатных pull'ов за 6 часов для анонимных пользователей. Это было воспринято сообществом крайне негативно, но Docker объяснил: «Мы обслуживаем миллиарды pull'ов в месяц, инфраструктура стоит денег». Rate limiting подтолкнул компании к покупке платных подписок и одновременно стимулировал создание зеркал (registry mirrors) внутри организаций.

Сегодня Docker Hub — это:
- **15M+ репозиториев**
- **317B+ pull'ов суммарно** (на октябрь 2023)
- Официальные образы от Microsoft, Oracle, MongoDB, Redis, NVIDIA
- AI-модели: Llama, Mistral, Gemma — доступны через `docker model pull`

---

## 6. Эволюция архитектуры: монолит → микросервисы

Подводя итог архитектурной эволюции — путь от одного бинарника к сети взаимодействующих сервисов:

```
2013:                         2026:

┌─────────────┐               ┌──────────────┐
│  docker     │               │  Docker       │  ← CLI + GUI
│  (монолит)  │               │  Desktop      │
└─────────────┘               └──────┬───────┘
                                    │
                         ┌──────────▼──────────┐
                         │  dockerd            │  ← Engine API
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼─────────────────┐
              │                     │                  │
     ┌────────▼────────┐  ┌─────────▼────────┐ ┌──────▼───────┐
     │  BuildKit       │  │  containerd      │ │  SwarmKit    │
     │  (сборка)       │  │  (runtime mgr)   │ │  (cluster)   │
     └─────────────────┘  └────────┬─────────┘ └──────────────┘
                                   │
                          ┌────────▼────────┐
                          │     runC        │  ← OCI Runtime
                          │  (namespaces)   │
                          └─────────────────┘
```

Каждый компонент — отдельный проект с собственным lifecycle, репозиторием и набором maintainer'ов. Все — open-source. Docker Engine 29 — это дистрибутив, собранный из Moby-компонентов, примерно как Ubuntu — дистрибутив, собранный из Debian-пакетов.

---

## 7. График: ключевые версии и даты

```
2013 ─┬─ v0.1: LXC, первый публичный релиз
      │
2014 ─┼─ v0.9: libcontainer, отказ от LXC
      ├─ v1.0: production readiness
      │
2015 ─┼─ v1.6: Docker Compose (ex fig)
      ├─ v1.9: Docker Network (libnetwork)
      │
2016 ─┼─ v1.12: Swarm Mode (оркестрация внутри Docker)
      ├─ containerd v0.2: выделение runtime-менеджера
      │
2017 ─┼─ v17.03: CalVer, CE/EE разделение
      ├─ containerd → CNCF (март 2017, принят)
      ├─ Moby Project анонсирован (апрель 2017)
      │
2018 ─┼─ v18.06: Docker Desktop стабилен (Mac/Win)
      ├─ BuildKit: экспериментальный builder
      │
2019 ─┼─ v19.03: containerd graduated CNCF
      ├─ Docker Enterprise продана Mirantis (ноябрь 2019)
      │
2020 ─┼─ v20.10: Docker Compose v2 (Go), BuildKit стабилен
      ├─ Docker Hub rate limiting (ноябрь 2020)
      │
2021 ─┼─ Docker Desktop Personal/Pro: бесплатный → freemium
      │
2022 ─┼─ v23.0: SemVer (частичный возврат)
      ├─ Docker Extensions marketplace
      │
2023 ─┼─ Docker Scout (октябрь 2023) — SBOM + CVE анализ
      ├─ docker init — автоматическая докеризация
      │
2024 ─┼─ v26.0: Полный возврат к SemVer
      ├─ Docker + NVIDIA (GPU-доступ внутри контейнеров)
      │
2025 ─┼─ v28.0: Docker Model Runner (AI-модели как образы)
      ├─ Docker Agent — AI-агенты в контейнерах
      │
2026 ─┴─ v29.0: Wasm-поддержка, Model Runner stable, LTS
```

---

## 8. Итог: уроки эволюции

1. **Технологии умирают не от слабости, а от монолитности.** Docker выжил как open-source проект именно потому, что вовремя разобрал себя на компоненты (runC, containerd, Moby) — вместо того, чтобы защищать монолитную кодовую базу.

2. **Конкуренция ускоряет эволюцию.** Kubernetes заставил Docker модуляризироваться. Podman — улучшить rootless-режим. Cloud-managed сервисы — добавить AI/ML. Без конкуренции Docker мог бы застрять в 2015 году.

3. **Open-source ≠ бесплатно.** Docker Desktop, Docker Hub limits, Enterprise-подписки — Docker нашёл способ монетизироваться, не закрывая код Engine. Это редкий баланс, которого достигают единицы (Red Hat, GitLab, MongoDB — с оговорками).

4. **Платформа побеждает инструмент.** `docker run` — это инструмент. Docker Desktop + Docker Hub + Docker Scout + Model Runner — это платформа. Эволюция Docker — это превращение из CLI-утилиты в экосистему, покрывающую полный цикл разработки.

5. **AI — не хайп, а вторая жизнь.** Если бы Docker остался «просто контейнерами», к 2026 году он мог бы стать legacy. AI/ML-нагрузки открыли абсолютно новый рынок — для GPU-контейнеров, Model Runner, Agentic AI, где Docker — не «один из», а пионер.

---

## Ключевые даты эволюции

| Дата | Событие |
|------|---------|
| 2013, март | Docker 0.1 — первый публичный релиз на LXC |
| 2014, март | Docker 0.9 — libcontainer, отказ от LXC |
| 2014, июнь | Docker 1.0 — production readiness |
| 2015, июнь | Создана Open Container Initiative (OCI) |
| 2016, июль | Docker 1.12 — Swarm Mode (оркестрация внутри Docker) |
| 2016, декабрь | containerd выделен как отдельный компонент |
| 2017, март | containerd принят в CNCF |
| 2017, апрель | Moby Project — Docker разделён на компоненты |
| 2018 | BuildKit — новый builder, multi-platform сборка |
| 2019, ноябрь | Docker Enterprise продана Mirantis, $35M инвестиции, реструктуризация |
| 2020, июль | Docker Compose v2 (Go, CLI plugin) |
| 2021, август | Docker Desktop — Personal/Pro планы, enterprise больше не бесплатен |
| 2023, октябрь | Docker Scout (SBOM + CVE), docker init (авто-докеризация) |
| 2024, март | Docker + NVIDIA — GPU-доступ в контейнерах |
| 2025, апрель | Docker Model Runner — AI-модели как OCI-артефакты |
| 2026, май | Docker 29.4 — Wasm, Model Runner stable, LTS |

---

## Источники

1. **Docker, Inc. — Docker Blog: Introducing Docker 1.13 (2017):** Переход на CalVer, основные фичи 1.13. Источник: docker.com/blog
2. **Docker, Inc. — Docker Blog: Introducing the Moby Project (2017):** Анонс Moby Project, разделение монолита на компоненты. Источник: docker.com/blog
3. **CNCF — containerd Project Journey Report:** Хронология containerd: выделение из Docker, принятие в CNCF, graduated status. Источник: cncf.io/reports
4. **Wikipedia — Docker (software):** Timeline релизов, OCI-стандартизация, CE/EE разделение. Источник: wikipedia.org
5. **TechCrunch — Mirantis acquires Docker Enterprise (2019):** Сделка Mirantis—Docker, $35M инвестиции, смена CEO. Источник: techcrunch.com
6. **TechTarget — Docker Enterprise spun off to Mirantis (2019):** Детали продажи enterprise-бизнеса, реструктуризация компании. Источник: techtarget.com
7. **Docker, Inc. — Docker Blog: Docker Partners with NVIDIA (2024):** GPU-доступ в Docker-контейнерах, партнёрство NVIDIA. Источник: docker.com/blog
8. **Docker, Inc. — Docker Blog: Introducing Docker Model Runner (2025):** AI-модели как OCI-артефакты, Model Runner CLI. Источник: docker.com/blog
9. **GitHub — moby/moby CHANGELOG.md (v1.0.0):** Список фич первого production-релиза Docker. Источник: github.com/moby/moby
10. **Red Hat — The History of Containers (2015, обновлено):** OCI-стандартизация, runC, containerd в контексте экосистемы. Источник: redhat.com
11. **Docker Docs — Engine release notes (prior releases, 1.13–18.09):** Детальные changelog'и версий. Источник: docs.docker.com
12. **Communications of the ACM — A Decade of Docker Containers (2026):** Ретроспектива 10+ лет Docker — от стартапа до индустриального стандарта. Источник: cacm.acm.org

---

## Связанные статьи

- [01-history.md](01-history.md) — История создания Docker: dotCloud, Solomon Hykes, pivot 2013
- [03-design-decisions.md](03-design-decisions.md) — Ключевые архитектурные решения Docker
- [../02-basics/01-what-is.md](../02-basics/01-what-is.md) — Что такое Docker
- [../10-future/02-trends.md](../10-future/02-trends.md) — Будущее Docker и контейнеризации
