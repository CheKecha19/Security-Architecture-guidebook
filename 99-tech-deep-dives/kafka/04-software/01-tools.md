# Инструменты для работы с Apache Kafka: CLI, GUI, мониторинг и управление

**Трек 04 — Software** | Статья 1 из 4  
**Проект:** kafka | **Профиль:** A (Infrastructure Platform)  
**Дата:** 2026-05-13

---

## Содержание

1. [Обзор: зачем нужны инструменты](#обзор-зачем-нужны-инструменты)
2. [CLI-инструменты Kafka (из коробки)](#cli-инструменты-kafka-из-коробки)
3. [Сторонние CLI-инструменты](#сторонние-cli-инструменты)
4. [GUI-инструменты для управления Kafka](#gui-инструменты-для-управления-kafka)
5. [Инструменты мониторинга и наблюдаемости](#инструменты-мониторинга-и-наблюдаемости)
6. [Консоли управления и платформы](#консоли-управления-и-платформы)
7. [Сравнительная таблица инструментов](#сравнительная-таблица-инструментов)
8. [Рекомендации по выбору](#рекомендации-по-выбору)
9. [Источники](#источники)

---

## Обзор: зачем нужны инструменты

Apache Kafka — распределённая платформа, и работать с ней исключительно через код неудобно. Разработчикам нужно создавать топики, просматривать сообщения, проверять лаги (отставания) consumer-групп. Администраторам — перебалансировать кластер, менять конфигурации, отслеживать здоровье брокеров. Без инструментов это превращается в угадывание по логам.

**Ключевая мысль:** инструменты Kafka делятся на три слоя — CLI для быстрых операций, GUI для визуального управления и мониторинг для production-наблюдаемости. Правильный выбор на каждом уровне определяет, будет ли ваша работа с Kafka эффективной или мучительной.

Инструментарий Kafka можно представить в виде пирамиды:

```
        ┌──────────────┐
        │ Мониторинг   │  ← Prometheus, Grafana, Burrow, Cruise Control
        │ & Дашборды   │
        ├──────────────┤
        │   GUI / UI   │  ← AKHQ, Kafdrop, kafka-ui, Conduktor, Kpow
        │  управление   │
        ├──────────────┤
        │  CLI-утилиты  │  ← kafka-topics, kcat, confluent CLI, jmx-exporter
        └──────────────┘
```

---

## CLI-инструменты Kafka (из коробки)

Apache Kafka поставляется с набором shell-скриптов в директории `bin/`. Это минимальный необходимый инструментарий для управления кластером, топиками, consumer-группами и проверки метаданных. Все инструменты используют `--bootstrap-server` для подключения (вместо устаревшего `--zookeeper`).

### Управление топиками — `kafka-topics.sh`

Самый часто используемый инструмент. Позволяет создавать, удалять, описывать и изменять топики.

```bash
# Создать топик с 3 партициями и фактором репликации 1
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders \
  --partitions 3 --replication-factor 1

# Просмотреть список всех топиков
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Детальная информация о топике (партиции, лидеры, реплики, ISR)
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic orders

# Увеличить количество партиций (уменьшить нельзя!)
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic orders --partitions 6

# Удалить топик
kafka-topics.sh --bootstrap-server localhost:9092 \
  --delete --topic orders
```

**Важно:** `--zookeeper` устарел начиная с Kafka 2.8 и удалён в 3.x. Все операции теперь через `--bootstrap-server`.

### Продюсирование и консьюминг сообщений — `kafka-console-producer.sh` и `kafka-console-consumer.sh`

Утилиты для быстрой проверки — «положить и прочитать сообщение». Незаменимы при отладке.

```bash
# Отправить сообщения (интерактивный режим — каждая строка = одно сообщение)
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders

# С ключом и партиционированием (key=value; разделитель табуляция)
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --property "parse.key=true" \
  --property "key.separator=:"

# Прочитать сообщения с самого начала топика
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --from-beginning

# Консьюминг с конкретной consumer-группой (для отслеживания offset)
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --group my-group

# Чтение с показом ключей, таймстемпов и партиций
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --from-beginning \
  --property print.key=true \
  --property print.timestamp=true \
  --property print.partition=true
```

Аналогия: `kafka-console-producer` = `echo "data" > /dev/tcp/kafka/topic`, а `kafka-console-consumer` = `tail -f /var/log/topic.log`.

### Управление consumer-группами — `kafka-consumer-groups.sh`

Инструмент для просмотра состояния consumer-групп, их отставаний (lag) и сброса смещений (offset reset).

```bash
# Список всех consumer-групп
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Детальная информация по группе (lag, текущий offset, хост consumer'а)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# Сбросить offset на начало для конкретной группы
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-earliest --execute

# Сбросить offset на конкретную временную метку
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-datetime 2026-05-01T00:00:00.000 \
  --execute
```

**Когда использовать:** каждый раз, когда в production падает alert «consumer lag > 1000» — первым делом идём в этот инструмент.

### Административные инструменты

| Инструмент | Назначение |
|---|---|
| `kafka-configs.sh` | Просмотр и изменение конфигураций топиков, брокеров, пользователей, client-ов |
| `kafka-acls.sh` | Управление Access Control Lists (права доступа к топикам/группам) |
| `kafka-reassign-partitions.sh` | Ручное перераспределение партиций между брокерами (через JSON-файл плана) |
| `kafka-broker-api-versions.sh` | Показать версию Kafka на брокере и поддерживаемые API-версии |
| `kafka-server-start.sh` / `kafka-server-stop.sh` | Запуск и остановка брокера |
| `kafka-storage.sh` | Форматирование хранилища для KRaft-режима, генерация Cluster ID |
| `kafka-cluster.sh` | Получение Cluster ID, дерегистрация брокера |
| `kafka-metadata-quorum.sh` | Статус кворума в KRaft-режиме (лидер, lag, voters) |
| `kafka-metadata-shell.sh` | Интерактивный просмотр содержимого metadata-лога KRaft-кластера |
| `kafka-features.sh` | Управление feature-флагами (динамическое включение/отключение функций) |
| `kafka-delegation-tokens.sh` | Управление delegation tokens для аутентификации без Kerberos |
| `kafka-leader-election.sh` | Принудительный запуск выборов лидера партиций |
| `kafka-log-dirs.sh` | Информация о директориях хранения логов на брокерах |
| `kafka-mirror-maker.sh` | Репликация данных между кластерами (deprecated → заменён на MirrorMaker 2) |

**Установка:** CLI-инструменты идут в составе дистрибутива Kafka. Достаточно скачать архив с [kafka.apache.org/downloads](https://kafka.apache.org/downloads) и распаковать. Никаких дополнительных зависимостей не требуется — инструменты работают поверх JVM, которая уже нужна для запуска Kafka.

```bash
# Установка Kafka (включает все CLI-инструменты)
wget https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
tar -xzf kafka_2.13-4.0.0.tgz
cd kafka_2.13-4.0.0/bin/
```

Docker-образы также содержат все инструменты:

```bash
docker run -it --rm apache/kafka:latest /bin/bash
# внутри контейнера: bin/kafka-topics.sh ...
```

---

## Сторонние CLI-инструменты

### kcat (ранее kafkacat)

**kcat** — «netcat для Kafka», не-JVM утилита на C от создателя librdkafka (Magnus Edenhill). Не требует Java, работает быстро, минимально потребляет ресурсы.

**Ключевые возможности:**
- Produce и consume сообщений как обычный pipe в Unix
- Metadata list mode — просмотр состояния кластера, топиков, партиций
- Поддержка всех механизмов аутентификации (SASL/SSL/Kerberos)
- Работает с Avro через Schema Registry (при сборке с поддержкой)
- Встроен в официальный репозиторий Ubuntu и Debian

```bash
# Установка
# Ubuntu/Debian
apt install kcat

# macOS
brew install kcat

# Docker
docker run -it --rm edenhill/kcat:1.7.1

# Просмотреть метаданные кластера (режим -L)
kcat -b localhost:9092 -L
# Вывод: список брокеров, топиков, партиций с лидерами и репликами

# Produce: отправить содержимое файла как сообщения (каждая строка = сообщение)
kcat -b localhost:9092 -t orders -P messages.txt

# Consume: прочитать и вывести (ключ + значение, разделённые табом)
kcat -b localhost:9092 -t orders -C -f '%k\t%s\n'

# Consume с таймстемпами и партициями
kcat -b localhost:9092 -t orders -C -f 'Topic %t [%p] at offset %o: key=%k value=%s\n'
```

Сравнение с kafka-console-consumer: kcat быстрее (нет старта JVM), не требует установленной Java, и встраивается в shell-пайплайны (`kcat ... | jq . | grep ...`).

### Confluent CLI

Если вы используете Confluent Platform или Confluent Cloud — `confluent` CLI даёт унифицированный интерфейс:

```bash
# Установка
brew install confluentinc/tap/cli  # macOS
# или скачать с docs.confluent.io/confluent-cli

# Создать топик в Confluent Cloud
confluent kafka topic create orders --partitions 3

# Просмотреть consumer-группы
confluent kafka consumer-group list

# Produce сообщение
confluent kafka topic produce orders --parse-key
```

**Когда выбирать confluent CLI вместо нативных kafka-\* инструментов:** если кластер в Confluent Cloud — confluent CLI автоматически подхватывает API-ключи и конфигурацию из контекста, не нужно указывать bootstrap-сервер вручную.

---

## GUI-инструменты для управления Kafka

CLI эффективен для автоматизации, но для визуальной навигации, просмотра сообщений в реальном времени и быстрой диагностики GUI-инструменты незаменимы. Вот актуальный обзор на 2025–2026 год.

### AKHQ (ранее KafkaHQ)

**Тип:** Open-source (Apache 2.0) | **Stars:** ~3 600  
**GitHub:** [tchiotludo/akhq](https://github.com/tchiotludo/akhq)

AKHQ — один из самых функциональных open-source GUI. Написан на Java (Micronaut), поставляется как JAR или Docker-образ.

**Ключевые возможности:**
- Multi-cluster management — несколько кластеров в одном интерфейсе
- Топики: создание, удаление, просмотр конфигурации, браузинг сообщений
- Consumer-группы: просмотр lag в реальном времени, сброс offset'ов по таймстемпу
- Schema Registry: управление Avro-схемами (создание, обновление, удаление)
- Kafka Connect: управление коннекторами (старт/стоп, конфигурация, мониторинг задач)
- ACL: управление правами доступа
- Автоматическая десериализация Avro (при подключенном Schema Registry)
- Аутентификация: поддержка LDAP, OAuth2, basic auth

**Установка:**

```bash
# Docker (рекомендуемый способ)
docker run -d -p 8080:8080 \
  -e AKHQ_CONFIGURATION=$(cat application.yml | base64) \
  tchiotludo/akhq

# Kubernetes (Helm)
helm repo add akhq https://akhq.io/
helm install akhq akhq/akhq
```

**Ограничения:** на очень больших кластерах (>1000 топиков) интерфейс может замедляться. Нет встроенных дашбордов с графиками метрик — для этого нужен связка Prometheus+Grafana.

### Kafka-UI (Provectus)

**Тип:** Open-source (Apache 2.0) | **Stars:** ~12 000  
**GitHub:** [provectus/kafka-ui](https://github.com/provectus/kafka-ui)

Самый популярный open-source web UI для Kafka по количеству звёзд на GitHub. Активно развивается, написан на Java (Spring Boot) + React.

**Ключевые возможности:**
- Multi-cluster management
- Браузинг сообщений с поддержкой JSON, Avro, Protobuf
- Встроенный лёгкий дашборд с метриками (throughput, lag)
- Динамическое создание и конфигурирование топиков
- Просмотр consumer-групп с per-partition lag
- RBAC (Role-Based Access Control) — гранулярное управление доступом
- Data masking — скрытие чувствительных данных в просматриваемых сообщениях
- Кастомизируемые SerDe-плагины
- OAuth 2.0 аутентификация (GitHub, GitLab, Google)

**Установка:**

```bash
# Docker
docker run -d -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=local \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092 \
  provectuslabs/kafka-ui:latest

# Docker Compose (из официального репозитория)
git clone https://github.com/provectus/kafka-ui.git
cd kafka-ui/documentation/compose
docker-compose -f kafka-ui.yaml up -d

# Kubernetes
helm repo add kafka-ui https://provectus.github.io/kafka-ui-charts
helm install kafka-ui kafka-ui/kafka-ui
```

**Кому подходит:** командам, которым нужно быстро развернуть UI для разработки и тестирования. Богатый набор функций при минимальной конфигурации.

### Kafdrop

**Тип:** Open-source (Apache 2.0) | **Stars:** ~5 700  
**GitHub:** [obsidiandynamics/kafdrop](https://github.com/obsidiandynamics/kafdrop)

Лёгкий web UI на Spring Boot. Минималистичный, но покрывает основные потребности.

**Ключевые возможности:**
- Просмотр брокеров, топиков, партиций
- Браузинг сообщений (JSON, Avro, Protobuf)
- Просмотр consumer-групп и lag
- Создание топиков
- Просмотр ACL
- Поддержка SASL, TLS
- Встроенная поддержка Azure Event Hubs
- Helm-чарт для Kubernetes

**Установка:**

```bash
# Docker
docker run -d -p 9000:9000 \
  -e KAFKA_BROKERCONNECT=kafka:9092 \
  obsidiandynamics/kafdrop:latest

# Java (прямой запуск JAR)
java --add-opens=java.base/sun.nio.ch=ALL-UNNAMED \
  -jar kafdrop-4.1.0.jar \
  --kafka.brokerConnect=localhost:9092
```

**Ограничения:** минимум административных функций (нет управления конфигурациями, нет Kafka Connect/Schema Registry). Не подходит как единственный инструмент для эксплуатации production-кластера.

### Redpanda Console (ранее Kowl)

**Тип:** Open-source (BSL — Business Source License) | **Stars:** ~3 900  
**GitHub:** [redpanda-data/console](https://github.com/redpanda-data/console)

Разработан компанией Redpanda, работает как с Redpanda, так и с Apache Kafka.

**Ключевые возможности:**
- JavaScript-фильтры для поиска сообщений (мощный ad-hoc поиск)
- Автоматическое распознавание кодировки (JSON, Avro, Protobuf, CBOR, MessagePack)
- Time-travel debugging — просмотр сообщений по временной метке
- Редактирование offset'ов consumer-групп на уровне партиций
- Управление ACL и SASL-SCRAM пользователями
- Встроенная работа со Schema Registry
- Kafka Connect управление
- Embed topic documentation из Git-репозитория

**Установка:**

```bash
# Docker
docker run -d -p 8080:8080 \
  -e KAFKA_BROKERS=localhost:9092 \
  docker.redpanda.com/redpandadata/console:latest

# Kubernetes (через Redpanda Operator)
```

**Кому подходит:** командам, которые хотят продвинутые возможности поиска и отладки сообщений. Особенно актуален для Redpanda, но отлично работает и с Apache Kafka.

### Conduktor Console

**Тип:** Коммерческий (проприетарный, есть free tier)  
**Сайт:** [conduktor.io](https://conduktor.io)

Conduktor Console — корпоративная платформа, позиционирующаяся как «Kafka UI для платформенных команд с десятками кластеров».

**Ключевые возможности:**
- Multi-tenancy — изоляция команд и проектов в одном интерфейсе
- RBAC с детальным контролем доступа
- Data masking и шифрование сообщений
- Audit log — все действия пользователей записываются
- Холодное хранение (cold storage)
- Мониторинг и алертинг
- Kafka Connect и Schema Registry из коробки
- Поддержка 20+ команд и множества кластеров в одном экземпляре

**Установка:**

```bash
# Desktop (Windows/macOS/Linux) — скачать с conduktor.io/download
# Docker (для self-hosted Console)
docker run -d -p 8080:8080 conduktor/conduktor-console:latest
```

**Кому подходит:** enterprise-командам с требованиями compliance, аудита и multi-tenancy. Free tier подходит для одного разработчика на одном кластере.

### Другие GUI-инструменты

| Инструмент | Лицензия | Основная фишка |
|---|---|---|
| **CMAK** (Kafka Manager) | Open-source (Apache 2.0) | Разработан Yahoo, управление партициями и репликами, batch-операции |
| **Offset Explorer** (Kafka Tool) | Коммерческий (free для личного использования) | Десктоп-приложение, просмотр и редактирование сообщений, устаревший UI |
| **Kpow** (Factor House) | Коммерческий (free basic) | Аудит, RBAC, LDAP/SAML, data governance, клиенты: Binance, Cash App |
| **Lenses** | Коммерческий | DataOps-платформа, SQL-потоковая обработка, топология данных в виде графа |
| **Kafka IDE** | Коммерческий | Десктоп-приложение, авто-вывод и редактирование схем без Schema Registry |
| **Kadeck** | Коммерческий | Десктоп (Windows/macOS/Linux), enterprise compliance |
| **Aiven Kafka UI Plugin** | Бесплатный | Плагин для JetBrains IDE (IntelliJ) и VS Code |

---

## Инструменты мониторинга и наблюдаемости

CLI и GUI хороши для «ручного» управления, но в production нужен автоматизированный мониторинг с алертами и историческими дашбордами.

### Prometheus + JMX Exporter + Grafana

Стандартный стек для мониторинга Kafka.

**Как это работает:**
1. **JMX Exporter** — Java-агент, прикрепляемый к процессу Kafka-брокера. Собирает JMX MBeans (метрики JVM и Kafka) и отдаёт их по HTTP в формате Prometheus.
2. **Prometheus** — скрапит метрики с JMX Exporter'ов всех брокеров с заданным интервалом.
3. **Grafana** — визуализирует метрики через дашборды.

```bash
# Шаг 1: Скачать JMX Exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/1.0.1/jmx_prometheus_javaagent-1.0.1.jar

# Шаг 2: Конфигурация экспортера (kafka-metrics.yml)
# lowercaseOutputName: true
# rules:
#   - pattern: kafka.server<type=(.+), name=(.+)><>Value
#     name: kafka_server_$1_$2

# Шаг 3: Прикрепить к брокеру
export KAFKA_OPTS="-javaagent:/path/to/jmx_prometheus_javaagent.jar=7071:/path/to/kafka-metrics.yml"

# Шаг 4: Запустить Kafka как обычно — метрики доступны на localhost:7071/metrics
```

**Ключевые метрики для мониторинга:**
- `kafka_server_BrokerTopicMetrics_BytesInPerSec` — входящий трафик
- `kafka_server_BrokerTopicMetrics_BytesOutPerSec` — исходящий трафик
- `kafka_server_ReplicaManager_UnderReplicatedPartitions` — партиции с неполной репликацией (критический индикатор проблем)
- `kafka_server_ReplicaManager_IsrShrinksPerSec` — частота сужения ISR
- `kafka_server_ReplicaFetcherManager_MaxLag` — максимальный lag репликации
- `kafka_network_RequestChannel_RequestQueueSize` — очередь запросов (индикатор перегрузки брокера)
- `kafka_controller_ControllerStats_ActiveControllerCount` — должен быть равен 1

**Готовые Grafana-дашборды:**
- [Kafka Broker Overview (ID: 24002)](https://grafana.com/grafana/dashboards/24002) — для KRaft-режима
- [Strimzi Kafka Dashboard (ID: 24626)](https://grafana.com/grafana/dashboards/24626) — для Strimzi на Kubernetes
- Confluent Control Center — если используется Confluent Platform

### Burrow (LinkedIn)

**Тип:** Open-source (Apache 2.0) | **Stars:** ~3 800  
**GitHub:** [linkedin/Burrow](https://github.com/linkedin/Burrow)

Burrow — специализированный монитор consumer lag без необходимости настройки порогов (threshold-less).

**Как это работает (аналогия):** вместо того чтобы говорить «alert, если lag > 1000», Burrow анализирует поведение consumer-группы во времени через скользящее окно. Если группа стабильно догоняет топик — всё хорошо. Если начала отставать — alert. Это как следить за пульсом пациента, а не просто измерять температуру раз в час.

```bash
# Установка
# Docker
docker run -d -p 8000:8000 \
  -v $(pwd)/burrow-config:/etc/burrow \
  linkedin/burrow

# HTTP API
curl http://localhost:8000/v3/kafka/local/consumer/my-group/status
# Ответ: OK, WARNING, ERROR — статус consumer-группы
```

**Ключевые особенности:**
- Не требует настройки порогов (sliding window evaluation)
- Multi-cluster support
- HTTP endpoint для запроса статуса consumer-групп
- Email и HTTP-нотификации (Slack, PagerDuty и т.д.)
- Интеграция в Prometheus/Grafana через exporter

### Cruise Control (LinkedIn)

**Тип:** Open-source (BSD 2-Clause) | **Stars:** ~2 700  
**GitHub:** [linkedin/cruise-control](https://github.com/linkedin/cruise-control)

Cruise Control — система автоматической балансировки кластера Kafka, созданная LinkedIn для управления кластером, обрабатывающим ~4.5 триллиона сообщений в день.

**Ключевые возможности:**
- Автоматический мониторинг загрузки ресурсов брокеров (CPU, сеть, диск)
- Goal-based ребалансировка: распределение партиций с учётом заданных целей (равномерная нагрузка, минимизация перемещений данных, соблюдение rack-awareness)
- Self-healing: автоматическое восстановление после отказа брокера
- Anomaly detection: обнаружение аномалий (отказ брокера, перегрузка)
- Административные операции: добавить/удалить/понизить брокера, запустить preferred leader election
- REST API и веб-интерфейс (Cruise Control Frontend)

```bash
# Docker
docker run -d -p 9090:9090 \
  -e KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  linkedin/cruise-control:latest

# REST API — проверить состояние кластера
curl http://localhost:9090/kafkacruisecontrol/state

# Запросить предложение по ребалансировке
curl -X POST "http://localhost:9090/kafkacruisecontrol/rebalance?dryrun=true"
```

**Когда использовать:** на кластерах с >10 брокеров, где ручная перебалансировка партиций становится непрактичной.

---

## Консоли управления и платформы

### Confluent Control Center

**Тип:** Проприетарный (в составе Confluent Platform)  
**Сайт:** [confluent.io](https://www.confluent.io/product/confluent-platform/gui-driven-management-and-monitoring/)

Полнофункциональная консоль управления, входящая в Confluent Platform.

**Ключевые возможности:**
- Визуализация потоков данных между топиками, producer и consumer
- End-to-end latency мониторинг
- Мониторинг здоровья кластера, брокеров и топиков
- Управление Schema Registry
- Разработка и запуск ksqlDB-запросов
- Алертинг на основе метрик
- Интеграция с RBAC и audit log (Confluent Platform enterprise)

**Установка:** идёт в составе Confluent Platform (`confluent control-center start`). Не open-source — требуется лицензия Confluent для production-использования.

### Strimzi Operator (для Kubernetes)

**Тип:** Open-source (Apache 2.0) | **Cloud Native Computing Foundation (CNCF) проект**  
**Сайт:** [strimzi.io](https://strimzi.io)

Strimzi — Kubernetes-оператор для управления Kafka. Не GUI в классическом смысле, а инструмент автоматизации развёртывания.

**Ключевые возможности:**
- Декларативное управление Kafka-кластером через Kubernetes Custom Resources
- Встроенная интеграция с Prometheus и Grafana
- Автоматическая настройка JMX Exporter для всех брокеров
- Управление Kafka Connect, MirrorMaker 2, Kafka Bridge через CRD
- Поддержка KRaft (с версии 0.38+)
- Автоматическое обновление версий Kafka без даунтайма
- Cruise Control из коробки (с 0.43+)
- Helm-чарты для быстрой установки

```bash
# Установка Strimzi
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Создание Kafka-кластера (YAML-манифест)
cat <<EOF | kubectl apply -n kafka -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

---

## Сравнительная таблица инструментов

### CLI-инструменты

| Инструмент | Язык | Зависимости | Установка | Лучшее применение |
|---|---|---|---|---|
| kafka-\* (встроенные) | Java | JVM | В составе Kafka | Полный цикл администрирования |
| kcat | C | librdkafka | apt/brew/docker | Быстрый produce/consume в shell-пайплайнах |
| Confluent CLI | Go | Нет | brew/scoop/curl | Работа с Confluent Cloud |

### GUI и управление

| Инструмент | Open Source | Stars GitHub | Интерфейс | Multi-кластер | Connect / SR | RBAC | Установка |
|---|---|---|---|---|---|---|---|
| Kafka-UI (Provectus) | Да (Apache 2.0) | 12 000 | Web | Да | SR | Да | Docker, Helm, K8s |
| AKHQ | Да (Apache 2.0) | 3 600 | Web | Да | Connect + SR | Нет | Docker, Helm, JAR |
| Kafdrop | Да (Apache 2.0) | 5 700 | Web | Нет | Нет | Нет | Docker, JAR |
| Redpanda Console | Да (BSL) | 3 900 | Web | Да | Connect + SR | ACL | Docker, K8s Operator |
| CMAK | Да (Apache 2.0) | 11 900 | Web | Да | Нет | Нет | Сборка из исходников |
| Conduktor | Нет (free tier) | N/A | Desktop + Web | Да | Connect + SR | Да | Desktop app, Docker |
| Kpow | Нет (free basic) | N/A | Web | Да | Connect + SR | Да (LDAP/SAML) | Docker, K8s |
| Confluent CC | Нет (Confluent) | N/A | Web | Да | Connect + SR | Да (ent) | В составе CP |
| Offset Explorer | Нет (free personal) | N/A | Desktop | Да | SR | Нет | Установщик |

### Мониторинг

| Инструмент | Фокус | Open Source | Установка | Масштаб |
|---|---|---|---|---|
| Prometheus + JMX Exporter + Grafana | Все метрики брокера/JVM | Да | Java agent + Docker | Любой |
| Burrow | Consumer lag | Да (Apache 2.0) | Docker, Go binary | Крупные кластеры |
| Cruise Control | Балансировка кластера | Да (BSD 2-Clause) | Docker, JAR | >10 брокеров |
| Confluent Control Center | End-to-end мониторинг | Нет | В составе CP | Confluent Platform |
| Strimzi | Автоматизация на K8s | Да (Apache 2.0) | kubectl apply | Kubernetes-native |

---

## Рекомендации по выбору

### Для разработчика (локальная среда)

Минимальный набор:
1. **kafka-ui (Provectus)** в Docker — даёт всё необходимое для визуальной работы
2. **kcat** — быстрый CLI для shell-скриптов
3. **Aiven Kafka UI Plugin для IntelliJ/VS Code** — если IDE-ориентированный workflow

```bash
# Быстрый старт: kafka-ui + Kafka в Docker Compose
curl -O https://raw.githubusercontent.com/provectus/kafka-ui/master/documentation/compose/kafka-ui.yaml
docker compose -f kafka-ui.yaml up -d
# kafka-ui доступен на http://localhost:8080
```

### Для команды (staging / небольшой production)

1. **AKHQ** — покрывает Kafka Connect и Schema Registry (чего нет в kafka-ui в полном объёме)
2. **Prometheus + JMX Exporter + Grafana** — для мониторинга
3. **Cruise Control** — если >10 брокеров

### Для enterprise (production с compliance)

1. **Conduktor или Kpow** — RBAC, audit log, data masking
2. **Prometheus + Grafana + Burrow** — полный мониторинг
3. **Cruise Control** — автоматическая балансировка
4. **Strimzi** — если на Kubernetes

### Дерево решений

```
Нужен GUI для Kafka?
├── Только CLI? → kafka-* утилиты (из коробки) или kcat (если без Java)
├── Open-source web UI?
│   ├── Нужен Kafka Connect + Schema Registry? → AKHQ
│   ├── Нужен красивый UI + OAuth? → kafka-ui (Provectus)
│   ├── Нужен поиск по сообщениям? → Redpanda Console
│   └── Минималистичный → Kafdrop
├── Enterprise (compliance/RBAC/audit)?
│   ├── Уже на Confluent Platform? → Confluent Control Center
│   ├── Множество команд и кластеров? → Conduktor
│   └── Data governance → Kpow
└── Мониторинг?
    ├── Все метрики + дашборды → Prometheus + Grafana
    ├── Consumer lag → Burrow
    └── Авто-балансировка → Cruise Control
```

### Антипаттерны (чего не стоит делать)

- **Использовать только CMAK (Kafka Manager) в 2026** — инструмент не обновлялся активно, UI устарел, нет поддержки KRaft
- **Разворачивать 5 разных GUI в одном кластере** — каждый инструмент создаёт нагрузку на кластер (metadata-запросы)
- **Мониторить Kafka только по логам** — без Prometheus/Grafana вы узнаете о проблемах, когда пользователи уже жалуются
- **Использовать kafka-console-consumer для production-мониторинга** — это инструмент для отладки, а не для постоянного наблюдения

---

## Связанные статьи

- [02-implementation.md](./02-implementation.md) — развёртывание и конфигурация Kafka
- [03-open-source.md](./03-open-source.md) — open-source проекты вокруг Kafka
- [04-enterprise.md](./04-enterprise.md) — коммерческие и managed-решения
- [../07-operations/01-monitoring.md](../07-operations/01-monitoring.md) — полное руководство по мониторингу

---

## Источники

1. [Apache Kafka CLI Tools — Confluent Documentation](https://docs.confluent.io/kafka/operations-tools/kafka-tools.html) — официальная документация по CLI-инструментам (Tier 1)
2. [Apache Kafka Cheat Sheet — Confluent](https://www.confluent.io/learn/kafka-cheat-sheet/) — шпаргалка по основным командам (Tier 1)
3. [kcat (kafkacat) — Confluent Documentation](https://docs.confluent.io/platform/current/tools/kafkacat-usage.html) — официальная документация kcat (Tier 1)
4. [Top 12 Free Kafka GUI Tools 2025 — AutoMQ Blog](https://automq.com/blog/top-12-free-kafka-gui) — обзор бесплатных GUI (Tier 2)
5. [Top 8 Free Kafka UI Tools 2025 — Aiven Blog](https://aiven.io/blog/top-kafka-ui) — обзор UI-инструментов (Tier 2)
6. [Kafka UI Tools Compared — Conduktor](https://conduktor.io/compare/kafka-ui-tools) — сравнение Kafbat, AKHQ, Conduktor (Tier 2)
7. [Redpanda Console — Kafka Web UI](https://www.redpanda.com/data-streaming/redpanda-console-kafka-ui) — официальная страница продукта (Tier 1)
8. [GitHub: provectus/kafka-ui](https://github.com/provectus/kafka-ui) — репозиторий Kafka-UI (~12K stars) (Tier 1)
9. [GitHub: linkedin/cruise-control](https://github.com/linkedin/cruise-control) — репозиторий Cruise Control (Tier 1)
10. [GitHub: linkedin/Burrow](https://github.com/linkedin/Burrow) — репозиторий Burrow consumer lag checker (Tier 1)
11. [GitHub: tchiotludo/akhq](https://github.com/tchiotludo/akhq) — репозиторий AKHQ (Tier 1)
12. [GitHub: obsidiandynamics/kafdrop](https://github.com/obsidiandynamics/kafdrop) — репозиторий Kafdrop (Tier 1)
13. [Strimzi Overview — strimzi.io](https://strimzi.io/docs/operators/0.49.1/overview) — документация Strimzi Operator (Tier 1)
14. [Grafana Kafka Broker Overview Dashboard](https://grafana.com/grafana/dashboards/24002) — готовый дашборд Grafana (Tier 1)
15. [Conduktor Console](https://conduktor.io/product/kafka-ui) — продуктовая страница Conduktor (Tier 2)
