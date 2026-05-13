# 07-операции.05 — Автоматизация Apache Kafka: IaC, GitOps, CI/CD

**Нижняя строка:** автоматизация Kafka — это не «опционально, если есть время». Ручное управление кластером из десятков брокеров, сотен топиков и тысяч ACL гарантированно приводит к конфигурационному дрейфу, человеческим ошибкам и инцидентам. Три столпа production-grade автоматизации: Infrastructure as Code (Terraform/Pulumi/Ansible) для провижининга, GitOps (ArgoCD/Flux) для синхронизации состояния, CI/CD (Jenkins/GitHub Actions/GitLab CI) для поставки изменений. Выбор стека зависит от рантайма: Kubernetes → Strimzi + ArgoCD, bare metal / VM → Ansible + Terraform, managed → Terraform-provider облака.

---

## 1. Зачем автоматизировать Kafka

**Ключевая мысль:** Kafka — распределённая, stateful-система. Ручное управление не просто неудобно — оно опасно. Автоматизация превращает операции из «молитвы перед каждым apply» в воспроизводимый, версионируемый, тестируемый процесс.

### 1.1 Проблемы ручного управления

В production-кластере средней руки (10 брокеров, 200 топиков, 50 consumer groups, 100 ACL) накапливается колоссальный объём конфигурационной работы:

- **Создание топиков** с правильным числом партиций, фактором репликации и retention
- **Управление ACL** для producers, consumers, Kafka Connect, Streams
- **Обновление конфигурации брокеров** (`server.properties` с ~200 параметрами)
- **Развёртывание новых брокеров** при масштабировании
- **Rolling upgrade** с соблюдением inter.broker.protocol.version
- **Настройка Kafka Connect connectors** (source/sink)
- **Schema Registry** — регистрация и эволюция схем

**Аналогия:** управлять production-кластером вручную — это как пытаться заменить двигатель автомобиля, пока он едет по трассе. Ни у кого нет «плана обслуживания» — всё происходит ad hoc. Через месяц никто не помнит, почему для `orders-v3` поставили `compression.type=snappy`, а для `orders-v2` — `gzip`. Это и есть конфигурационный дрейф.

### 1.2 Четыре кита автоматизации Kafka

Автоматизация Kafka строится на четырёх уровнях зрелости:

```
Уровень 1: Инвентаризация        Уровень 2: Декларативность
┌──────────────────────┐         ┌─────────────────────────┐
│ Вся конфигурация     │    →    │ Инфраструктура описана   │
│ в одном месте (git)  │         │ как код (Terraform, YAML)│
│ Минимум: README.md   │         │ Желаемое состояние = код │
└──────────────────────┘         └─────────────────────────┘
         ↓                                 ↓
Уровень 3: Автосинхронизация      Уровень 4: Полный CI/CD
┌─────────────────────────┐       ┌──────────────────────────┐
│ GitOps: Git —           │   →   │ PR → Plan → Apply →      │
│ единственный source     │       │ Verify → Merge           │
│ of truth.               │       │ Автоматическое           │
│ ArgoCD/Flux reconciler. │       │ тестирование изменений.  │
│ Drift detection.        │       │ Rollback одной кнопкой.  │
└─────────────────────────┘       └──────────────────────────┘
```

**Production-ready минимум — Уровень 2.** Уровень 3 желателен для Kubernetes-based деплоев. Уровень 4 — для кластеров с частыми изменениями (>10 изменений конфигурации в неделю).

### 1.3 Что даёт автоматизация: измеримые результаты

| Метрика | Без автоматизации | С автоматизацией |
|---------|-------------------|------------------|
| Создание топика | 15–30 мин (тикет → админ → CLI) | 2–5 мин (PR → merge → apply) |
| Rolling upgrade кластера | 2–4 часа (риск ошибки) | 30–60 мин (автоматический rolling restart) |
| Конфигурационный дрейф | Есть всегда (>20% параметров различаются) | Исключён (Git = truth) |
| Восстановление кластера | Дни (ручное воссоздание) | Часы (terraform apply + playbook) |
| Аудит изменений | «Кто-то менял что-то в пятницу» | `git log` — кто, что, когда, зачем |

---

## 2. Infrastructure as Code: выбор инструмента

**Ключевая мысль:** универсального инструмента нет. Выбор зависит от рантайма (Kubernetes, bare metal, managed) и существующего опыта команды. Terraform доминирует для cloud-провижининга, Strimzi — для Kubernetes, Ansible — для конфигурации VM.

### 2.1 Матрица выбора IaC-инструмента

| Сценарий | Инструмент | Почему |
|----------|-----------|--------|
| Kafka на Kubernetes | **Strimzi** (K8s Operator) + ArgoCD | Declarative CRD, rolling update «из коробки», K8s-native |
| Managed Kafka (Confluent Cloud, MSK, Aiven) | **Terraform** (cloud provider's module) | Управление всей cloud-инфраструктурой в одном языке |
| Kafka на bare metal / VM | **Ansible** + Terraform | Terraform provisioning VM, Ansible — конфигурация Kafka |
| Гибрид (VM + K8s + managed) | **Terraform** (мульти-провайдер) | Единый workflow для разнородной инфраструктуры |
| Команда разработчиков без Ops | **Pulumi** (Python/TS/Go) | Использование знакомых языков программирования |

### 2.2 Terraform: детальный разбор

Terraform от HashiCorp — декларативный IaC-инструмент, ставший де-факто стандартом для управления cloud-инфраструктурой. Для Kafka он работает на трёх уровнях:

**Уровень 1: Инфраструктурное обеспечение** — создание VM, сетей, дисков:
```hcl
# Создание VM для Kafka-брокеров в AWS
resource "aws_instance" "kafka_broker" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "m5.xlarge"

  root_block_device {
    volume_type = "gp3"
    volume_size = 200  # быстрый диск для log.dirs
    iops        = 5000
  }

  vpc_security_group_ids = [aws_security_group.kafka.id]
  subnet_id              = aws_subnet.kafka[count.index].id

  tags = {
    Name = "kafka-broker-${count.index + 1}"
    Role = "kafka"
  }
}
```

**Уровень 2: Управление ресурсами Kafka** — топики, ACL, квоты через kafka provider:
```hcl
provider "kafka" {
  bootstrap_servers = ["broker1:9092", "broker2:9092", "broker3:9092"]
  tls_enabled       = true
  client_cert       = file("certs/terraform.pem")
  client_key        = file("certs/terraform-key.pem")
  ca_cert           = file("certs/ca.pem")
}

resource "kafka_topic" "orders" {
  name               = "orders.v3"
  partitions         = 12
  replication_factor = 3

  config = {
    "retention.ms"         = "604800000"    # 7 дней
    "compression.type"     = "snappy"
    "cleanup.policy"       = "delete"
    "min.insync.replicas"  = "2"
    "max.message.bytes"    = "10485760"     # 10 MB
  }

  # Защита от случайного удаления production-топиков
  lifecycle {
    prevent_destroy = true
  }
}

resource "kafka_acl" "orders_producer_write" {
  resource_name       = "orders.v3"
  resource_type       = "Topic"
  acl_principal       = "User:orders-service"
  acl_host            = "*"
  acl_operation       = "Write"
  acl_permission_type = "Allow"
}
```

**Уровень 3: Managed-сервисы** — Confluent Cloud, MSK через специализированные провайдеры:
```hcl
# Confluent Cloud — полный пример
resource "confluent_environment" "production" {
  display_name = "production"
}

resource "confluent_kafka_cluster" "main" {
  display_name = "main-cluster"
  availability = "MULTI_ZONE"
  cloud        = "AWS"
  region       = "eu-west-1"
  environment {
    id = confluent_environment.production.id
  }

  standard {}

  lifecycle {
    prevent_destroy = true
  }
}

# Топики в Confluent Cloud с управлением через REST API
resource "confluent_kafka_topic" "page_views" {
  kafka_cluster {
    id = confluent_kafka_cluster.main.id
  }
  topic_name    = "page_views"
  partitions    = 30
  rest_endpoint = confluent_kafka_cluster.main.rest_endpoint
  credentials {
    key    = confluent_api_key.kafka_admin.id
    secret = confluent_api_key.kafka_admin.secret
  }

  config = {
    "retention.ms" = "259200000"  # 3 дня
  }
}
```

**Архитектурный паттерн для Terraform + Kafka:**

```
terraform-kafka/
├── modules/
│   ├── kafka-broker/        # Модуль для одного брокера
│   │   ├── main.tf          # VM, диск, сеть
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── kafka-topic/         # Модуль для стандартизированного топика
│       ├── main.tf          # Topic + ACL + квоты
│       └── variables.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars # dev-specific: 1 broker, RF=1
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars # staging: 3 brokers, RF=2
│   └── production/
│       ├── main.tf
│       ├── terraform.tfvars # prod: 6 brokers, RF=3, prevent_destroy
│       └── topics/
│           ├── orders.tf    # Конфигурация топиков как код
│           ├── payments.tf
│           └── users.tf
└── global/
    ├── backend.tf           # S3 + DynamoDB lock для state
    └── providers.tf
```

**Best practices при работе с Terraform:**

1. **Remote state с блокировками.** State должен храниться в S3/GCS с DynamoDB-блокировкой — это предотвращает конфликты при одновременных операциях.
2. **Plan перед Apply — всегда.** `terraform plan` показывает diff. Сделайте это частью CI. Без плана — слепой полёт.
3. **prevent_destroy на production.** `lifecycle { prevent_destroy = true }` спасает от катастрофических опечаток. Снять флаг можно только руками.
4. **Модули для стандартизации.** Модуль `kafka-topic` гарантирует, что `min.insync.replicas` всегда >= 2 во всех топиках, а `compression.type` не указан как `gzip` по историческим причинам.
5. **Секреты — только через Vault / Secrets Manager.** Никаких API-ключей в `variables.tf` или `terraform.tfvars`. Используйте `vault_generic_secret` datasource или `aws_secretsmanager_secret_version`.

### 2.3 Pulumi: Infrastructure as Code на реальных языках

Pulumi занимает ту же нишу, что и Terraform, но использует языки программирования общего назначения (TypeScript, Python, Go, C#, Java) вместо HCL. Это даёт преимущества для команд, которые предпочитают выражать инфраструктурную логику через конструкции языка (циклы, условия, классы).

**Пример: Kafka-кластер на Confluent Cloud через Pulumi (Python):**

```python
import pulumi
import pulumi_confluentcloud as confluent

# Окружение
env = confluent.Environment("production",
    display_name="production")

# Kafka-кластер
cluster = confluent.KafkaCluster("main",
    display_name="main",
    availability="MULTI_ZONE",
    cloud="AWS",
    region="eu-west-1",
    environment=confluent.KafkaClusterEnvironmentArgs(
        id=env.id,
    ),
    standard=confluent.KafkaClusterStandardArgs())

# Функция для стандартизации создания топиков
def create_topic(name: str, partitions: int, retention_days: int):
    retention_ms = retention_days * 86400000
    return confluent.KafkaTopic(name,
        kafka_cluster=confluent.KafkaTopicKafkaClusterArgs(
            id=cluster.id,
        ),
        topic_name=name,
        partitions=partitions,
        rest_endpoint=cluster.rest_endpoint,
        credentials=confluent.KafkaTopicCredentialsArgs(
            key=api_key.id,
            secret=api_key.secret,
        ),
        config={
            "retention.ms": str(retention_ms),
            "min.insync.replicas": "2",
        })

# Топики создаются вызовом функции — DRY без копипасты
topics = {
    "orders":    create_topic("orders", 12, 7),
    "payments":  create_topic("payments", 6, 30),
    "pageviews": create_topic("pageviews", 30, 3),
}

pulumi.export("cluster_id", cluster.id)
pulumi.export("bootstrap", cluster.bootstrap_endpoint)
```

**Когда выбирать Pulumi вместо Terraform:**

- Команда — разработчики, не ops-инженеры. Для них `if/else` и `for` привычнее, чем HCL-синтаксис
- Инфраструктура содержит сложную логику (вычисление retention на основе SLA, генерация конфигов из внешнего источника)
- Нужна тесная интеграция с другими системами в том же языке (например, Pulumi + boto3 для AWS)

**Когда выбирать Terraform вместо Pulumi:**

- Больше примеров, документации, community support для Kafka
- HCL декларативность снижает риск побочных эффектов (сайд-эффектов «умного кода»)
- Команда — ops-инженеры с опытом Terraform

### 2.4 Ansible: конфигурационный менеджмент

Ansible остаётся основным инструментом конфигурации Kafka на bare metal и VM. В отличие от Terraform/Pulumi, он работает по процедурной модели (шаги выполняются последовательно), что ближе к классическому «runbook».

**Пример: Ansible playbook для развёртывания Kafka-кластера на трёх VM:**

```yaml
---
- name: Развёртывание Kafka-кластера (KRaft mode)
  hosts: kafka_nodes
  become: yes
  vars:
    kafka_version: "4.1.0"
    kafka_scala_version: "2.13"
    kafka_home: "/opt/kafka"
    data_dir: "/var/lib/kafka"

  tasks:
    - name: Установка системных зависимостей
      apt:
        name:
          - openjdk-17-jdk-headless
          - netcat-openbsd
          - jq
        state: present
        update_cache: yes

    - name: Создание пользователя kafka
      user:
        name: kafka
        system: yes
        shell: /usr/sbin/nologin
        home: "{{ kafka_home }}"

    - name: Загрузка Kafka
      get_url:
        url: "https://downloads.apache.org/kafka/{{ kafka_version }}/kafka_{{ kafka_scala_version }}-{{ kafka_version }}.tgz"
        dest: "/tmp/kafka.tgz"
        checksum: "sha512:..."

    - name: Распаковка Kafka
      unarchive:
        src: "/tmp/kafka.tgz"
        dest: "{{ kafka_home }}"
        owner: kafka
        group: kafka
        extra_opts: ["--strip-components=1"]
        remote_src: yes

    - name: Создание data-директории
      file:
        path: "{{ data_dir }}"
        owner: kafka
        group: kafka
        state: directory
        mode: '0750'

    - name: Настройка KRaft-cluster-id
      shell: |
        {{ kafka_home }}/bin/kafka-storage.sh random-uuid
      register: cluster_id
      run_once: yes

    - name: Форматирование хранилища
      shell: |
        {{ kafka_home }}/bin/kafka-storage.sh format \
          --config {{ kafka_home }}/config/kraft/server.properties \
          --cluster-id {{ cluster_id.stdout }}
      become_user: kafka

    - name: Генерация server.properties из шаблона
      template:
        src: server.properties.j2
        dest: "{{ kafka_home }}/config/kraft/server.properties"
        owner: kafka
        group: kafka
        mode: '0640'

    - name: Создание systemd unit
      template:
        src: kafka.service.j2
        dest: /etc/systemd/system/kafka.service

    - name: Запуск Kafka
      systemd:
        name: kafka
        state: started
        enabled: yes
        daemon_reload: yes
```

**Ansible-роли для организации кода:**

```
ansible-kafka/
├── roles/
│   ├── kafka-broker/
│   │   ├── tasks/main.yml          # Инсталляция и конфигурация
│   │   ├── handlers/main.yml       # Перезапуск брокера
│   │   ├── templates/
│   │   │   ├── server.properties.j2  # Jinja2-шаблон конфига
│   │   │   ├── kafka.service.j2      # systemd unit
│   │   │   └── log4j2.xml.j2         # Конфигурация логирования
│   │   ├── defaults/main.yml         # Значения по умолчанию
│   │   └── vars/main.yml             # Переменные роли
│   └── kafka-topics/
│       └── tasks/main.yml            # Создание/удаление топиков
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml               # Общие переменные
│   │       └── kafka_nodes.yml       # Kafka-specific переменные
│   └── staging/
│       └── hosts.yml
├── playbooks/
│   ├── deploy_cluster.yml            # Основной playbook
│   ├── rolling_upgrade.yml           # Обновление без даунтайма
│   └── manage_topics.yml             # CRUD для топиков
└── ansible.cfg
```

**Ключевые практики Ansible для Kafka:**

- **Idempotency — критически.** Каждая задача должна проверять текущее состояние перед действием. `creates:`, `shell: cmd && cmd || echo 'skipped'`, проверка наличия файла.
- **Rolling serial execution.** `serial: 1` для обновлений, чтобы не положить весь кластер одновременно. Между итерациями — проверка здоровья брокера.
- **Ansible Vault для секретов.** Храните inter-broker SSL-сертификаты, truststore-пароли и API-ключи зашифрованными.
- **Проверка конфигурации до apply.** Используйте `--check --diff` для предварительной валидации изменений.

---

## 3. GitOps для Kafka: ArgoCD и Flux

**Ключевая мысль:** GitOps превращает Git в единственный источник истины для Kafka-инфраструктуры. Это не просто CI/CD для YAML-файлов — это модель, в которой желаемое состояние (Git) непрерывно сверяется с фактическим (Kubernetes), и любое расхождение либо автоматически исправляется, либо алертится.

### 3.1 GitOps-паттерн для Kafka

Принципиальная схема GitOps для Kafka на Kubernetes:

```
┌─────────┐    git push     ┌──────────┐    poll/sync    ┌──────────────┐
│ Dev Team │ ─────────────→ │ Git Repo │ ──────────────→ │ ArgoCD/Flux  │
│          │                │ (truth)  │                 │ (reconciler) │
└─────────┘                └──────────┘                 └──────┬───────┘
                                                               │
                                                    apply/sync │
                                                               ↓
┌──────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                           │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐       │
│  │ Strimzi CRD │  │ Kafka CRs    │  │ KafkaTopic CRs      │       │
│  │ (Operator)  │  │ (cluster)    │  │ (topics)            │       │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬──────────┘       │
│         │               │                     │                   │
│         │   manages     │                     │                   │
│         ↓               ↓                     ↓                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              Kafka Cluster (actual state)                 │     │
│  │  Broker-0 │ Broker-1 │ Broker-2 │ Zookeeper-0..2 (legacy)│     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

**Ключевое свойство GitOps:** при любом ручном изменении в кластере (`kubectl edit kafka` на продакшене) ArgoCD обнаружит расхождение (drift) и либо вернёт состояние к Git-дескриптору (selfHeal), либо поднимет алерт (OutOfSync).

### 3.2 Strimzi + ArgoCD: практическое руководство

Strimzi — CNCF-incubating Kubernetes Operator для Kafka, самый зрелый K8s-native способ управления Kafka.

**Шаг 1: Структура Git-репозитория для GitOps:**

```
kafka-gitops/
├── argocd/
│   └── root-app.yaml              # App of Apps — root deployment
├── infrastructure/
│   └── strimzi-operator/
│       ├── namespace.yaml
│       ├── operator-group.yaml
│       └── subscription.yaml       # Установка Strimzi через OLM/Helm
├── clusters/
│   └── production/
│       ├── kafka-cluster/
│       │   ├── kafka.yaml          # Kafka CR — описание кластера
│       │   ├── kafkatopics/        # Топики как отдельные CR
│       │   │   ├── orders.yaml
│       │   │   ├── payments.yaml
│       │   │   └── pageviews.yaml
│       │   └── kafkausers/         # Пользователи и ACL
│       │       ├── orders-service.yaml
│       │       └── analytics-service.yaml
│       ├── kafka-connect/
│       │   ├── connect-cluster.yaml
│       │   └── connectors/
│       │       ├── s3-sink.yaml
│       │       └── debezium-pg-source.yaml
│       └── kafka-mirrormaker/
│           └── mm2-cluster.yaml
└── envs/
    ├── dev/
    │   └── kustomization.yaml     # Override для dev: RF=1, 1 broker
    └── staging/
        └── kustomization.yaml     # Override для staging: RF=2
```

**Шаг 2: Kafka CR — полный production-дескриптор:**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-cluster
  namespace: kafka
spec:
  kafka:
    version: 4.1.0
    replicas: 6                       # 3 брокера в каждой зоне (multi-zone)
    image: quay.io/strimzi/kafka:latest-kafka-4.1.0
    resources:
      requests:
        cpu: "4"
        memory: 16Gi
      limits:
        cpu: "8"
        memory: 24Gi
    jvmOptions:
      -Xms: 8g
      -Xmx: 8g
      gcLoggingEnabled: true
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        configuration:
          brokerCertChainAndKey:
            secretName: kafka-broker-certs
            certificate: user.crt
            key: user.key
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
    storage:
      type: jbod                      # JBOD для производительности
      volumes:
        - id: 0
          type: persistent-claim
          size: 500Gi
          class: io2                  # High-throughput storage
          deleteClaim: false          # Данные переживают удаление кластера
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      log.retention.hours: 168         # 7 дней
      auto.create.topics.enable: false # Запрет авто-создания топиков
      message.max.bytes: 10485760      # 10 MB
      num.partitions: 12               # Дефолтное для новых топиков
      compression.type: producer       # Позволить producer выбирать
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: metrics-config.yml
    template:
      pod:
        topologySpreadConstraints:     # Распределение по зонам
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: DoNotSchedule
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                    - key: strimzi.io/name
                      operator: In
                      values:
                        - production-cluster-kafka
                topologyKey: kubernetes.io/hostname

  entityOperator:
    topicOperator:
      reconciliationIntervalSeconds: 60   # Частота проверки топиков
    userOperator:
      reconciliationIntervalSeconds: 60

  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"

  cruiseControl:
    config:
      goals: >
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        RackAwareGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        ReplicaCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        DiskCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        NetworkInboundCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        NetworkOutboundCapacityGoal,
        com.linkedin.kafka.cruisecontrol.analyzer.goals.
        CpuCapacityGoal
```

**Шаг 3: ArgoCD Application для Kafka-кластера (App of Apps pattern):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kafka-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: infrastructure
  source:
    repoURL: https://github.com/org/kafka-gitops.git
    targetRevision: main
    path: clusters/production/kafka-cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: kafka
  syncPolicy:
    automated:
      prune: true       # Удалять ресурсы, которых нет в Git
      selfHeal: true    # Автоматически исправлять drift
    syncOptions:
      - CreateNamespace=true
      - Validate=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

**Шаг 4: App of Apps — корневой деплоймент:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-kafka
  namespace: argocd
spec:
  project: infrastructure
  source:
    repoURL: https://github.com/org/kafka-gitops.git
    targetRevision: main
    path: argocd/                    # Папка, содержащая дочерние App-манифесты
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3.3 Flux: альтернатива ArgoCD

Flux (FluxCD) — второй крупный GitOps-инструмент, изначально созданный Weaveworks и переданный в CNCF. Он выполняет ту же функцию, что и ArgoCD, но архитектурно ближе к Kubernetes (использует controllers, а не единый сервис):

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: kafka-gitops
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/kafka-gitops
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: kafka-production
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/production/kafka-cluster
  prune: true
  sourceRef:
    kind: GitRepository
    name: kafka-gitops
  healthChecks:
    - apiVersion: kafka.strimzi.io/v1beta2
      kind: Kafka
      name: production-cluster
      namespace: kafka
```

**ArgoCD vs Flux — критерии выбора:**

| Критерий | ArgoCD | Flux |
|----------|--------|------|
| UI | Web UI + CLI | CLI-only (нет GUI) |
| Multi-tenancy | Projects + RBAC в интерфейсе | Через Kubernetes RBAC |
| Learning curve | Ниже (GUI помогает) | Выше (чистый CLI + CRD) |
| Масштабируемость | Application-level (до ~500 apps на экземпляр) | Controller-level (хорошо горизонтально масштабируется) |
| Экосистема | Шире (больше community examples для Kafka) | Активно растёт, сильная интеграция с Weave GitOps |

### 3.4 GitOps для топиков: управление как кодом

**Важнейшая практика:** топики должны управляться через GitOps, а не через KafkaTopic CR вручную.

**Антипаттерн (как НЕ надо):**
```bash
# Так делать нельзя на production
kubectl apply -f ad-hoc-topic.yaml  # В обход Git
# или
kafka-topics.sh --create --topic new-topic ...  # CLI напрямую
```

**Правильный GitOps workflow:**
1. Разработчик создаёт PR в `kafka-gitops` репозиторий с новым `KafkaTopic` YAML
2. CI проверяет: валидность YAML, `min.insync.replicas >= 2`, `replicationFactor >= 3`, naming convention
3. После approve и merge → ArgoCD синхронизирует → Strimzi Topic Operator создаёт топик
4. Через 60 секунд (reconciliation interval) топик готов

---

## 4. CI/CD для Kafka-кластера

**Ключевая мысль:** CI/CD для Kafka отличается от «обычного» CI/CD. Вы не деплоите бинарник — вы управляете состоянием stateful-системы. Pipeline должен быть разделён на безопасные стадии: lint → plan → авто-тесты → apply с approval gate.

### 4.1 Архитектура CI/CD pipeline для Kafka

```
┌───────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                              │
│                                                                    │
│  PR Created                                                        │
│     │                                                              │
│     ▼                                                              │
│  ┌─────────┐    ┌──────────┐    ┌────────────┐    ┌────────────┐  │
│  │  Lint   │ →  │  Plan    │ →  │  Unit +    │ →  │  Approval  │  │
│  │  (YAML  │    │ (Terra-  │    │  Integra-  │    │  Gate      │  │
│  │   HCL)  │    │  form)   │    │  tion      │    │  (human/   │  │
│  └─────────┘    └──────────┘    │  Tests)    │    │   auto)    │  │
│                                 └────────────┘    └─────┬──────┘  │
│                                                         │         │
│                          ┌──────────────────────────────┘         │
│                          ▼                                         │
│  ┌──────────────┐   ┌──────────┐   ┌──────────────┐   ┌────────┐ │
│  │ Terraform    │   │  Kafka   │   │  Smoke Test   │   │ Merge  │ │
│  │ Apply /      │ → │  Rolling │ → │  (produce/    │ → │  PR    │ │
│  │ ArgoCD Sync  │   │  Update  │   │   consume)    │   │        │ │
│  └──────────────┘   └──────────┘   └──────────────┘   └────────┘ │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

### 4.2 GitHub Actions: Terraform + Kafka

**Пример: CI/CD pipeline для Terraform-managed Kafka (GitHub Actions):**

```yaml
name: Kafka Infrastructure CI/CD

on:
  pull_request:
    paths:
      - 'terraform-kafka/**'
    branches: [main]

env:
  TF_VERSION: "1.8.0"
  AWS_REGION: "eu-west-1"

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  lint:
    name: Линтинг Terraform
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - name: Terraform fmt check
        run: terraform fmt -check -recursive terraform-kafka/
      - name: TFLint (custom rules)
        run: |
          tflint --config terraform-kafka/.tflint.hcl terraform-kafka/
      - name: Проверка naming convention для топиков
        run: |
          # Все топики должны быть в snake_case
          ! grep -rP 'topic_name\s*=\s*"[^"]*[A-Z]' terraform-kafka/

  plan:
    name: Terraform Plan
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456:role/github-actions-terraform
          aws-region: ${{ env.AWS_REGION }}
      - uses: hashicorp/setup-terraform@v3
      - name: Terraform Init & Plan
        id: plan
        run: |
          cd terraform-kafka/environments/${{ github.base_ref }}
          terraform init
          terraform plan -no-color -out=tfplan
          # Сохраняем план как комментарий в PR
          terraform show -no-color tfplan > /tmp/plan.txt
      - name: PR-комментарий с планом
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('/tmp/plan.txt', 'utf8');
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 📋 Terraform Plan\n\`\`\`\n${plan.substring(0, 60000)}\n\`\`\``
            });

  test:
    name: Интеграционные тесты
    needs: [lint, plan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Развёртывание тестового кластера
        run: |
          # Docker Compose для локального тестирования Kafka-конфигурации
          docker compose -f docker/kafka-test-cluster.yml up -d
          # Ждём готовности
          for i in $(seq 1 30); do
            if docker compose exec broker kafka-broker-api-versions.sh > /dev/null 2>&1; then
              echo "Kafka ready"
              break
            fi
            sleep 2
          done
      - name: Тест создания топика через конфигурацию
        run: |
          docker compose exec broker kafka-topics.sh \
            --bootstrap-server localhost:9092 \
            --create --topic test-automation \
            --partitions 3 --replication-factor 1
      - name: Тест produce/consume
        run: |
          echo "test-message-$(date)" | \
            docker compose exec -T broker kafka-console-producer.sh \
            --bootstrap-server localhost:9092 \
            --topic test-automation
      - name: Тест ACL через Admin API
        run: |
          docker compose exec broker kafka-acls.sh \
            --bootstrap-server localhost:9092 \
            --add --allow-principal User:testuser \
            --operation Read --topic test-automation

  apply:
    name: Terraform Apply (requires approval)
    if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456:role/github-actions-terraform
          aws-region: ${{ env.AWS_REGION }}
      - uses: hashicorp/setup-terraform@v3
      - name: Terraform Apply
        run: |
          cd terraform-kafka/environments/production
          terraform init
          terraform apply -auto-approve
      - name: Verify cluster health
        run: |
          # Health check: все брокеры в кластере
          BROKERS=$(aws ec2 describe-instances \
            --filters "Name=tag:Role,Values=kafka" "Name=instance-state-name,Values=running" \
            --query "Reservations[*].Instances[*].PrivateIpAddress" \
            --output text | wc -w)
          if [ "$BROKERS" -lt 3 ]; then
            echo "ERROR: Expected >= 3 brokers, found $BROKERS"
            exit 1
          fi
          echo "OK: $BROKERS brokers running"
```

### 4.3 GitLab CI: полный пайплайн для Ansible + Kafka

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - deploy

variables:
  ANSIBLE_FORCE_COLOR: "true"

# Валидация Ansible playbooks
validate_playbook:
  stage: validate
  image: python:3.12
  before_script:
    - pip install ansible ansible-lint yamllint
  script:
    - yamllint ansible-kafka/
    - ansible-lint ansible-kafka/playbooks/deploy_cluster.yml --exclude ansible-kafka/roles/
    - ansible-playbook ansible-kafka/playbooks/deploy_cluster.yml --syntax-check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Дымовые тесты с molecule
molecule_test:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - apk add python3 py3-pip gcc python3-dev musl-dev libffi-dev openssl-dev
    - pip install molecule molecule-docker ansible docker
  script:
    - cd ansible-kafka/roles/kafka-broker
    - molecule test --scenario-name default
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Деплой на production (manual approval)
deploy_kafka_production:
  stage: deploy
  when: manual                              # Human approval gate
  image: python:3.12
  before_script:
    - pip install ansible
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | base64 -d > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "$ANSIBLE_VAULT_PASSWORD" > ~/.vault_pass
  script:
    - >-
      ansible-playbook ansible-kafka/playbooks/deploy_cluster.yml
      --inventory ansible-kafka/inventories/production/hosts.yml
      --vault-password-file ~/.vault_pass
      --diff
  after_script:
    # Health check после деплоя
    - >-
      ansible kafka_nodes -m shell
      -a 'nc -z localhost 9092 && echo "HEALTHY" || echo "UNHEALTHY"'
      --inventory ansible-kafka/inventories/production/hosts.yml
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment:
    name: production
```

### 4.4 Jenkins: классический CI/CD с Confluent-интеграцией

Jenkins до сих пор широко используется в enterprise-средах для автоматизации Confluent Platform. Решение `kafka-cp-deploy-manager` от Platformatory — пример интеграции Jenkins с Confluent Ansible:

```
Jenkins Pipeline (упрощённая конфигурация):
┌────────────────────────────────────────────┐
│ 1. Параметры сборки (UI-форма)             │
│    • Git repo URL                           │
│    • Confluent Platform version             │
│    • Target environment (dev/staging/prod)  │
│    • Action (install/upgrade/rollback)      │
├────────────────────────────────────────────┤
│ 2. Checkout state file из Git               │
├────────────────────────────────────────────┤
│ 3. Ansible playbook на target environment   │
│    с параметром version и action            │
├────────────────────────────────────────────┤
│ 4. Post-deploy validation:                  │
│    • kafka-topics --describe (все CR)       │
│    • kafka-consumer-groups (лаги)           │
│    • health check всех сервисов             │
├────────────────────────────────────────────┤
│ 5. Rollback gate: при ошибке —              │
│    автоматический откат к предыдущей        │
│    версии (тэг в Git)                       │
└────────────────────────────────────────────┘
```

---

## 5. Автоматизация rolling upgrade

**Ключевая мысль:** rolling upgrade — самый опасный эксплуатационный сценарий. Автоматизация снижает риск с «всё может пойти не так» до «предсказуемый, контролируемый процесс».

### 5.1 Фазы автоматического rolling upgrade

```
Фаза 1: Pre-upgrade checks          Фаза 2: Rolling restart
┌────────────────────────┐          ┌────────────────────────┐
│ • Все брокеры онлайн   │          │ for each broker         │
│ • Consumer lag < порог │     →    │   controlledShutdown    │
│ • ISR = RF (нет URP)  │          │   → restart             │
│ • Достаточно диска     │          │   → wait ISR=RF         │
│ • Inter.broker.protocol│          │   → health check        │
│   version совместим    │          │   → next broker         │
└────────────────────────┘          └────────────────────────┘
         ↓                                   ↓
Фаза 3: Protocol upgrade             Фаза 4: Verify
┌────────────────────────┐          ┌────────────────────────┐
│ inter.broker.protocol  │     →    │ Все брокеры на новой    │
│ .version = NEW_VERSION │          │ версии                  │
│ → rolling restart      │          │ Consumer lag = 0        │
│ (только если меняется  │          │ ISR=RF для всех         │
│  wire protocol)        │          │ партиций                │
└────────────────────────┘          └────────────────────────┘
```

### 5.2 Ansible playbook для rolling upgrade

```yaml
---
- name: Rolling upgrade Kafka cluster
  hosts: kafka_nodes
  serial: 1                        # По одному брокеру за раз
  any_errors_fatal: true           # Отмена при ошибке
  vars:
    kafka_home: /opt/kafka
    new_version: "4.2.0"
    # Время ожидания между брокерами (даём ISR восстановиться)
    wait_between_brokers: 120

  pre_tasks:
    - name: Pre-upgrade: проверка состояния кластера
      assert:
        that:
          - "{{ item }}"
        fail_msg: "Кластер не в стабильном состоянии, отмена обновления"
      loop:
        - "under_replicated_partitions == 0"
        - "offline_partitions == 0"
        - "active_controller_count == 1"

  tasks:
    - name: Скачивание новой версии Kafka
      get_url:
        url: "https://downloads.apache.org/kafka/{{ new_version }}/kafka_{{ kafka_scala_version }}-{{ new_version }}.tgz"
        dest: "/tmp/kafka-new.tgz"

    - name: Распаковка новой версии
      unarchive:
        src: "/tmp/kafka-new.tgz"
        dest: "/opt/"
        remote_src: yes

    - name: Controlled shutdown брокера (нежно закрываем)
      shell: |
        {{ kafka_home }}/bin/kafka-server-stop.sh
      ignore_errors: yes

    - name: Ожидание завершения controlled shutdown
      wait_for:
        port: 9092
        state: stopped
        timeout: 30

    - name: Обновление symlink на новую версию
      file:
        src: "/opt/kafka_{{ kafka_scala_version }}-{{ new_version }}"
        dest: "{{ kafka_home }}"
        state: link
        force: yes

    - name: Запуск брокера
      systemd:
        name: kafka
        state: started

    - name: Ожидание готовности брокера
      wait_for:
        port: 9092
        state: started
        timeout: 60

    - name: Ожидание восстановления ISR
      pause:
        seconds: "{{ wait_between_brokers }}"

    - name: Post-upgrade: проверка брокера
      shell: |
        {{ kafka_home }}/bin/kafka-topics.sh \
          --bootstrap-server localhost:9092 \
          --describe --under-replicated-partitions | wc -l
      register: urp_count
      until: urp_count.stdout | int == 0
      retries: 20
      delay: 15

    - name: Обновление protocol version (после ВСЕХ брокеров)
      shell: |
        # Меняем inter.broker.protocol.version на новую (запускается на последней итерации)
        {{ kafka_home }}/bin/kafka-configs.sh \
          --bootstrap-server localhost:9092 \
          --entity-type brokers --entity-default \
          --alter --add-config inter.broker.protocol.version={{ new_version }}
      when: inventory_hostname == ansible_play_hosts[-1]
      run_once: yes
```

### 5.3 Strimzi: автоматический rolling update через Operator

В мире Strimzi rolling upgrade тривиален — достаточно изменить `spec.kafka.version` в Kafka CR:

```yaml
# Было:
spec:
  kafka:
    version: 4.1.0

# Стало:
spec:
  kafka:
    version: 4.2.0   # Strimzi сам сделает rolling restart
```

Strimzi автоматически:
- Выполнит controlled shutdown каждого pod'а по очереди
- Дождётся перебалансировки партиций (ISR восстановления)
- Создаст новый pod с обновлённым образом
- Проверит readiness пробу
- Перейдёт к следующему pod'у

При обнаружении проблемы — остановит процесс, оставив часть брокеров на старой версии (совместимость inter.broker.protocol гарантирована).

### 5.4 Автоматизация ротации сертификатов

Ротация SSL-сертификатов — регулярная операция, которую необходимо автоматизировать:

```bash
# Скрипт для автоматической ротации TLS-сертификатов (Kafka + TLS)
# 1. Генерация новых сертификатов
./generate-certs.sh --domains broker1.kafka.internal,broker2.kafka.internal,... \
                    --validity 365 \
                    --output-dir /etc/kafka/certs/new

# 2. Дистрибуция на все брокеры (rolling — по одному)
for broker in broker1 broker2 broker3; do
  echo "Ротация сертификата на $broker..."

  # Копирование новых сертификатов
  scp /etc/kafka/certs/new/* $broker:/etc/kafka/certs/new/

  # Controlled shutdown
  ssh $broker "systemctl stop kafka"

  # Замена сертификатов
  ssh $broker "cp /etc/kafka/certs/new/* /etc/kafka/certs/ && chown kafka:kafka /etc/kafka/certs/*"

  # Запуск
  ssh $broker "systemctl start kafka"

  # Проверка: подключение по TLS
  echo | openssl s_client -connect $broker:9093 -servername $broker 2>/dev/null | \
    openssl x509 -noout -enddate

  # Ожидание стабилизации
  sleep 60
done

# 3. Обновление сертификатов в Kafka Connect, Schema Registry
# (если используют те же сертификаты)
```

---

## 6. Полная автоматизация: от нуля до production

**Ключевая мысль:** конечная цель — воспроизводимая процедура развёртывания полного Kafka-стека за часы, а не за дни.

### 6.1 Чеклист: что должно быть автоматизировано

- [ ] **Provisioning инфраструктуры:** VM / K8s namespace, сети, диски, security groups
- [ ] **Deployment Kafka:** брокеры, конфигурация (KRaft или ZK), listeners, SSL
- [ ] **Управление топиками:** создание, изменение конфигурации, удаление (с retention-периодом)
- [ ] **Управление доступом:** ACL, SASL/SCRAM пользователи, mTLS-сертификаты
- [ ] **Kafka Connect:** кластер, connectors (source/sink), конфигурация
- [ ] **Schema Registry:** instance, compatibility mode
- [ ] **Мониторинг:** Prometheus JMX Exporter, Grafana dashboards, алерты
- [ ] **Логирование:** структурированные логи (JSON), агрегация в ELK/Loki
- [ ] **Бэкапы:** MirrorMaker 2 репликация, или tiered storage (KIP-405), или холодные бэкапы
- [ ] **Обновления:** rolling upgrade, откат, canary deployment
- [ ] **Disaster Recovery:** процедура восстановления из бэкапа / переключения на DR site

### 6.2 Единый Terraform-проект для полного стека

```hcl
# main.tf — оркестрация полного стека Kafka в AWS

# ── Инфраструктура ──
module "network" {
  source = "./modules/network"
  name   = "kafka-prod"
  azs    = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}

module "kafka_security" {
  source     = "./modules/kafka-security"
  vpc_id     = module.network.vpc_id
  kafka_port = 9092
}

# ── Kafka-брокеры (6 шт, по 2 в AZ) ──
module "kafka_cluster" {
  source             = "./modules/kafka-cluster"
  broker_count       = 6
  instance_type      = "m5.2xlarge"
  kafka_version      = "4.1.0"
  kafka_mode         = "kraft"           # KRaft mode, ZK не нужен
  subnets            = module.network.private_subnets
  security_group_id  = module.kafka_security.sg_id
  data_volume_size   = 500
  data_volume_type   = "io2"
  data_volume_iops   = 5000
}

# ── Kafka Connect (отдельный кластер) ──
module "kafka_connect" {
  source            = "./modules/kafka-connect"
  worker_count      = 3
  kafka_bootstrap   = module.kafka_cluster.bootstrap_endpoint
  connectors = {
    "s3-sink" = {
      "connector.class" = "io.confluent.connect.s3.S3SinkConnector"
      "s3.bucket.name"  = "kafka-data-lake"
      "topics"          = "orders,payments,pageviews"
      "flush.size"      = "1000"
    }
  }
}

# ── Schema Registry ──
module "schema_registry" {
  source           = "./modules/schema-registry"
  kafka_bootstrap  = module.kafka_cluster.bootstrap_endpoint
  mode             = "READWRITE"
  compatibility    = "FORWARD"
}

# ── Мониторинг ──
module "kafka_monitoring" {
  source               = "./modules/kafka-monitoring"
  kafka_brokers        = module.kafka_cluster.broker_ips
  jmx_exporter_port    = 9404
  prometheus_workspace = aws_prometheus_workspace.kafka.id
  alert_rules = {
    "ConsumerLagHigh"       = "sum(kafka_consumer_group_lag) > 10000 AND environment='production'",
    "UnderReplicatedPartitions" = "kafka_server_under_replicated_partitions > 0",
    "OfflinePartitions"     = "kafka_controller_offline_partitions_count > 0",
  }
}

# ── Tiered Storage (KIP-405) — S3 бэкенд для долгосрочного хранения ──
module "tiered_storage" {
  source       = "./modules/tiered-storage"
  bucket_name  = "kafka-tiered-storage-prod"
  kafka_role   = module.kafka_cluster.broker_role_arn
}
```

### 6.3 Сценарии автоматизации: таблица решений

| Операция | K8s / Strimzi | VM / Ansible | Managed (Confluent/MSK) |
|----------|---------------|--------------|-------------------------|
| Создание кластера | `kubectl apply -f kafka.yaml` (1 cmd) | `ansible-playbook deploy.yml` (1 cmd) | `terraform apply` (1 cmd) |
| Добавление брокера | `spec.kafka.replicas: 3 → 4` в Git | Terraform vm_count + Ansible join | `terraform apply` (изменить node count) |
| Rolling upgrade | `spec.kafka.version: X → Y` в Git | Ansible playbook (serial: 1) | `terraform apply` (изменить version) |
| Новый топик | PR с KafkaTopic YAML | Terraform kafka_topic resource | Terraform resource |
| Новый ACL | PR с KafkaUser YAML | Terraform kafka_acl resource | Terraform resource |
| Конфигурация брокера | `spec.kafka.config` в Kafka CR | Ansible template → rolling restart | Terraform configuration block |
| Бэкап/восстановление | MirrorMaker 2 CR | Ansible playbook | Terraform + облачный снапшот |

---

## 7. Антипаттерны автоматизации Kafka

### 7.1 «Автоматизируем всё сразу»

**Проблема:** команда пытается внедрить полный GitOps + Terraform + CI/CD за один спринт.

**Реальность:** автоматизация должна внедряться итеративно:
1. Спринт 1: все конфигурации в Git (даже если применяются руками)
2. Спринт 2: Terraform для provisioning + Ansible для конфигурации
3. Спринт 3: CI (plan + lint на PR)
4. Спринт 4: CD (apply после merge)

### 7.2 «Смешивание manual и automated state»

**Проблема:** часть топиков создаётся через Terraform, часть — руками через CLI.

**Решение:** либо полный переход на IaC, либо explicit разделение зон ответственности. Terraform не должен управлять ресурсами, которые создаются вне его state.

### 7.3 «Нет health check после деплоя»

**Проблема:** `terraform apply` прошёл успешно, но брокер не запустился из-за ошибки в конфиге.

**Решение:** после каждого apply — обязательная верификация:
```bash
# Проверка здоровья после деплоя (обязательная часть CI/CD)
kafka-topics.sh --bootstrap-server broker:9092 --list | wc -l    # топики доступны
kafka-consumer-groups.sh --bootstrap-server broker:9092 --list   # группы видны
kafka-metadata-shell.sh --snapshot /tmp/kraft-metadata snapshot  # KRaft quorum ok
```

### 7.4 «Секреты в коде»

**Проблема:** пароли, API-ключи, SSL-ключи лежат в `terraform.tfvars` или Ansible `vars/`.

**Решение:** все секреты — только через:
- HashiCorp Vault (Terraform `vault_generic_secret` datasource)
- AWS Secrets Manager
- Ansible Vault (с паролем, переданным через CI/CD переменную)
- Sealed Secrets (для Kubernetes/Flux)

### 7.5 «Автоматическое удаление без защиты»

**Проблема:** кто-то удалил KafkaTopic CR из Git → ArgoCD с `prune: true` удалил топик → данные потеряны.

**Решение:**
- `lifecycle { prevent_destroy = true }` на все data-resources в Terraform
- `prune: false` на production-окружениях (или `prune: true` только в dev)
- Топики с `cleanup.policy=compact` не должны удаляться автоматически никогда

---

## 8. Чеклист: готовность автоматизации к production

- [ ] Вся конфигурация Kafka (брокеры, топики, ACL, коннекторы) хранится в Git
- [ ] Terraform state хранится в remote backend (S3 + DynamoDB lock)
- [ ] Ansible Vault (или Vault) для всех секретов
- [ ] CI проверяет: `terraform validate`, `terraform fmt`, `ansible-lint`, валидацию YAML
- [ ] CI генерирует `terraform plan` и публикует как комментарий в PR
- [ ] CD требует approval gate (manual approval или multi-factor policy)
- [ ] После каждого apply — автоматический health check (брокеры, топики, consumer groups)
- [ ] `prevent_destroy = true` на production-ресурсах
- [ ] GitOps reconciler настроен: `selfHeal: true` (drift correction)
- [ ] Процедура rolling upgrade автоматизирована и протестирована
- [ ] Процедура rollback: один механизм (`terraform destroy` с `prevent_destroy` off, `argocd rollback`, Ansible revert commit)
- [ ] DR-сценарий: возможность развернуть кластер «с нуля» за <4 часа
- [ ] Мониторинг автоматизации: failure алерты на CI/CD pipeline, ArgoCD OutOfSync
- [ ] Документация: runbook для каждой автоматизированной операции (что делать, если автоматизация сломалась)

---

## Сводная таблица инструментов

| Категория | Инструменты | Сильные стороны | Ограничения |
|-----------|-------------|-----------------|-------------|
| IaC (Cloud) | Terraform, Pulumi, CloudFormation, Bicep | Единый язык для всей инфраструктуры; mature ecosystem | HCL (Terraform) — не язык программирования; state management может быть проблемой |
| Configuration Mgmt | Ansible, Chef, Puppet | Гибкость; подходит для VM/bare metal; Jinja2-шаблоны | Процедурный (не декларативный); сложнее для cloud-native |
| K8s Operator | Strimzi, Confluent for Kubernetes (CFK) | K8s-native; rolling update из коробки; PVC management | Только для Kubernetes; требует K8s-компетенций |
| GitOps | ArgoCD, FluxCD | Авто-синхронизация; drift detection; audit trail | Требует Git-дисциплины; сложность настройки multi-tenancy |
| CI/CD | GitHub Actions, GitLab CI, Jenkins, Tekton | Автоматизация полного цикла; approval gates; интеграционное тестирование | Pipeline as YAML может быть громоздким; Jenkins требует администрирования |
| Secrets | HashiCorp Vault, AWS Secrets Manager, Sealed Secrets, SOPS | Шифрование; rotation; audit | Vault — сложен в настройке; Sealed Secrets — только для K8s |

---

## Источники

| # | Источник | Тип | Примечание |
|---|----------|-----|------------|
| 1 | Apache Kafka Documentation (kafka.apache.org) — Operations, KRaft Configuration, Security, Upgrade Guide | Tier 1 (Official) | Первичный источник всех конфигурационных параметров и процедур обновления |
| 2 | Strimzi Documentation (strimzi.io/docs) — Deploying and Managing Strimzi, Custom Resource API Reference | Tier 1 (Official) | Полный API Kafka CR, KafkaTopic, KafkaUser, KafkaConnect, MirrorMaker |
| 3 | Confluent Documentation (docs.confluent.io) — Terraform Provider, Pulumi Provider, Confluent for Kubernetes | Tier 1 (Official) | Провайдеры Confluent для Terraform/Pulumi, CFK |
| 4 | HashiCorp Terraform Registry — Kafka Provider by Mongey, Confluentinc Provider, AWS MSK Module | Tier 1 (Official) | Документация Terraform-провайдеров для Kafka |
| 5 | Conduktor Glossary — "Infrastructure as Code for Kafka Deployments" (conduktor.io/glossary) | Tier 2 (Vendor) | Глубокий обзор IaC-подходов, GitOps-паттернов, сравнение инструментов |
| 6 | AutoMQ Blog — "Manage Kafka with Terraform: Why & How" (March 2025) | Tier 2 (Vendor) | Практическое руководство с примерами для Confluent, MSK, self-hosted |
| 7 | Platformatory Blog — "Automating Kafka Deployments with CI/CD" (platformatory.io) | Tier 2 (Vendor) | Подход CI/CD для Confluent Platform через Jenkins + Ansible |
| 8 | Software Patterns Lexicon — "Infrastructure as Code for Kafka" (softwarepatternslexicon.com) | Tier 3 | Сравнение Terraform/Ansible/Puppet с примерами кода |
| 9 | Civo Learn — "Kafka on Kubernetes: A Strimzi & GitOps Guide" (civo.com/learn) | Tier 2 (Community) | Пошаговое руководство Strimzi + ArgoCD + App of Apps |
| 10 | Pulumi Documentation (pulumi.com) — Confluent Cloud Provider, Kafka Provider | Tier 1 (Official) | Интеграция Pulumi с Kafka (TypeScript/Python/Go) |
| 11 | Apache Kafka KIPs: KIP-631 (KRaft Controller), KIP-405 (Tiered Storage), KIP-500 (ZooKeeper Removal), KIP-848 (New Consumer Group Protocol) | Tier 1 (KIPs) | Архитектурные решения, влияющие на автоматизацию и деплой |
| 12 | GitHub — strimzi-kafka-operator (github.com/strimzi/strimzi-kafka-operator) | Tier 1 (Open Source) | Исходный код Strimzi оператора, примеры CR, best practices |
