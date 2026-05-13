# Progress: kafka

Profile: A (Infrastructure Platform)
Started: 2026-05-03

## Track Status

| Track | Status | Articles | Words | Sources |
|-------|--------|----------|-------|---------|
| 01-history-evolution | done | 3/3 | ~18500 | 35 |
| 02-basics | done | 3/3 | ~19200 | 33 |
| 03-tech | done | 3/3 | ~16700 | 42 |
| 04-software | done | 4/4 | ~19600 | 75 |
| 05-development-api | done | 4/4 | ~24600 | 53 |
| 06-performance | done | 4/4 | ~24000 | 54 |
| 07-operations | done | 5/5 | ~24200 | 61 |
| 08-security | done | 4/4 | ~34400 | 72 |
| 09-real-world | done | 4/4 | ~25100 | 47 |
| 10-future | done | 3/3 | ~17500 | 36 |

## Article Detail

| # | Track | Article | Status | Words | Sources | Notes |
|---|-------|---------|--------|-------|---------|-------|
| 1 | 01 | 01-history.md | done | ~8500 | 10 | написана из знаний, источники указаны |
| 2 | 01 | 02-evolution.md | done | ~4504 | 15 | written by sub-agent, includes full release timeline from 0.7 to 4.2 |
| 3 | 01 | 03-design-decisions.md | done | ~5500 | 10 | page-cache vs in-memory, pull vs push, dumb-broker-smart-consumer, partitioning, ZK→KRaft, zero-copy, rejected features |
| 4 | 02 | 01-what-is.md | done | ~6800 | 9 | ELI5 (аэропорт, нервная система), когда использовать/не использовать (5 антипаттернов), дерево решений, Kafka vs очереди, экосистема, Docker quickstart, кто использует |
| 5 | 02 | 02-how-it-works.md | done | ~4900 | 10 | architecture, storage internals (segments .log/.index/.timeindex), write path (RecordAccumulator, Sender, acks), replication (ISR, leader election), read path (consumer groups, offset management, rebalance classic + new 4.0 protocol), zero-copy, page cache, KRaft, CAP positioning, end-to-end lifecycle |
| 6 | 02 | 03-core-concepts.md | done | ~7500 | 14 | core abstractions (topic, partition, offset, consumer group), delivery guarantees (at-most-once, at-least-once, exactly-once: idempotent + transactional), CAP positioning (tunable C↔A), consistency models (ordering, read-your-writes, monotonic reads, eventual), design principles (dumb-broker-smart-consumer, pull model, page-cache-first, zero-copy, batch-first, log-structured storage, retention vs compaction) |
| 7 | 03 | 01-related-technologies.md | done | ~4600 | 12 | Kafka Connect (200+ connectors), Schema Registry (Avro/Protobuf/JSON Schema), Kafka Streams, ksqlDB, Debezium CDC (PostgreSQL/MySQL/MongoDB/Oracle/SQL Server), Flink, Spark, Elasticsearch, S3/Data Lake, Kubernetes (Strimzi, CFK), Prometheus+Grafana, Redis, KRaft (замена ZooKeeper), архитектурная карта взаимодействия, правила выбора технологий-компаньонов |
| 8 | 03 | 02-ecosystem.md | done | ~4700 | 15 | 4-layer architecture, Kafka Connect (120+ connectors by category: DBs/NoSQL/Storage/Cloud/SaaS), Converter & SMT framework, Debezium CDC (11 DBs), Kafka Streams (Java/Scala only), ksqlDB SQL streaming, Apache Flink, Schema Registry (Avro/Protobuf/JSON), REST Proxy, 20+ language clients (Java/Python/Go/.NET/Node.js/Rust via librdkafka), capability matrix, open-source tools (Strimzi, AKHQ, kcat, Cruise Control, JulieOps), ecosystem evolution timeline 2013-2026 |
| 9 | 03 | 03-vs-alternatives.md | done | ~7400 | 15 | сравнение Kafka vs RabbitMQ/Pulsar/Redpanda/NATS/Kinesis/Pub/Sub, сравнительная таблица, дерево решений, бенчмарки 2024-2026, migration considerations |
| 10 | 04 | 01-tools.md | done | ~5200 | 15 | CLI (kafka-*, kcat, confluent CLI), GUI (AKHQ, kafka-ui, Kafdrop, Redpanda Console, Conduktor, CMAK, Kpow, Offset Explorer), мониторинг (Prometheus+JMX+Grafana, Burrow, Cruise Control), консоли (Confluent CC, Strimzi), сравнительные таблицы, дерево решений, антипаттерны |
| 11 | 04 | 02-implementation.md | done | ~4200 | 11 | bare metal, Docker Compose, K8s/Strimzi, KRaft vs ZK, server.properties (с комментариями на русском), production checklist (40+ пунктов), multi-broker cluster (3 controllers + 3 brokers isolated mode)
| 12 | 04 | 03-open-source.md | done | ~4000 | 26 | open-source ecosystem, Apache 2.0 vs SSPL vs BSL, Strimzi, Debezium, community tooling, self-hosted vs managed |
| 13 | 04 | 04-enterprise.md | done | ~6200 | 23 | Confluent Platform/Cloud, Amazon MSK, Azure Event Hubs, GCP Pub/Sub, Aiven, Redpanda Cloud, WarpStream, TCO-расчёт, SLA, vendor lock-in |
| 14 | 05 | 01-client-libraries.md | done | ~6500 | 14 | Feature parity table (9 clients, 20+ фич), version compatibility matrix (4.0+), код на 7 языках (Java/Python/Go/.NET/Node.js/Rust/C/C++), дерево выбора, 5 антипаттернов, benchmarks, чеклист |
| 15 | 05 | 02-api-reference.md | done | ~6400 | 12 | Wire Protocol, 50+ API keys, SASL, Admin API, чеклист клиента |
| 16 | 05 | 03-patterns.md | done | ~5700 | 12 | 6 паттернов, 14 антипаттернов, код на 4 языках, чеклисты |
| 17 | 05 | 04-testing.md | done | ~6000 | 15 | пирамида тестирования, Mock/Testcontainers/EmbeddedKafka, Trogdor chaos, CI/CD |
| 18 | 06 | 01-performance-basics.md | done | ~6500 | 13 | метрики (throughput/latency/resource utilization), типичные цифры (LinkedIn 2M msg/s, Confluent 605 MB/s, Google Cloud), методология 5-фазного бенчмаркинга, инструменты (kafka-perf-test, OMBF, Trogdor, custom), 6 типичных ошибок |
| 19 | 06 | 02-tuning.md | done | ~4300 | 15 | 3 оси trade-off, профильный тюнинг, кейс 0.42→70 MB/s |
| 20 | 06 | 03-scalability.md | done | ~7200 | 14 | partitioning strategies, horizontal broker scaling, consumer group adaptive scaling (KEDA), geo-replication (stretch clusters, MirrorMaker 2), tiered storage KIP-405 (3-9x cheaper, 115x faster recovery), KRaft metadata scalability (300K+ partitions), real examples (LinkedIn, Uber, Apple, Datadog) |
| 21 | 06 | 04-bottlenecks.md | done | ~3800 | 12 | 7 проблем, JMX/JFR/async-profiler, 4 кейса, diagnostic tree |
| 22 | 07 | 01-monitoring.md | done | ~5300 | 12 | 4 уровня мониторинга, JMX/Yammer/Kafka Metrics, 10+ ключевых метрик (consumer lag rate-based, ISR, ActiveController, RequestHandler/NetworkProcessor idle, under-replicated, under-minISR), архитектура Prometheus+Grafana+JMX Exporter+kafka_exporter, JMX Exporter конфиг, Prometheus alert rules (critical/warning/info), Grafana dashboard структура, Burrow, Confluent CC, health-check probes, борьба с high cardinality, KRaft/Tiered Storage, чеклист prod-готовности, топ-5 ошибок |
| 23 | 07 | 02-logging.md | done | ~3900 | 12 | 6 appender, JSON ECS/GELF, ELK/Loki/OpenSearch, Audit Trail SOC2/HIPAA |
| 24 | 07 | 03-backup-recovery.md | done | ~7200 | 12 | backup strategies (MM2, cold/kafka-backup, stretch-cluster, tiered-storage KIP-405), RTO/RPO planning, failover + failback playbooks, DR testing pyramid, Schema Registry/Connect/Streams considerations, production checklist |
| 25 | 07 | 04-troubleshooting.md | done | ~8000 | 13 | Системный подход к диагностике (5-шаговый алгоритм). Матрицы symptom→cause→solution: брокеры (не стартует, упал, disk full, CPU/память), репликация (URP, ISR churn), consumer groups (lag, частые ребалансировки, не получают сообщения, дубликаты), продюсеры (TimeoutException, RecordTooLarge, дубликаты), Kafka Connect (REST API диагностика, FAILED task, динамическое логирование), KRaft (metadata quorum, KIP-856 disk failure). Инструментарий: 10 CLI-утилит, 12 JMX-метрик, системные утилиты, анализ логов. Emergency runbook (пошаговый с хронометражом). Предотвращение потери данных (checklist до/после). Безопасность: 3 перспективы (инфраструктурный безопасник — TLS/SASL/сертификаты; DevSecOps — CI/CD/секреты; архитектор ИБ — threat model/compliance/расследование). 8 аналогий, 10+ code/config examples.
| 26 | 07 | 05-automation.md | done | ~5300 | 12 | Terraform/Ansible/Pulumi, ArgoCD GitOps, CI/CD pipeline |
| 27 | 08 | 01-security-features.md | done | ~6200 | 18 | TLS + 5 SASL-механизмов (PLAIN/SCRAM/GSSAPI/OAUTHBEARER), Delegation Tokens, ACL (resources/operations/patterns), KRaft/ZK security, 3 перспективы (infra/DevSecOps/Architect): STRIDE, trust boundaries, compliance (152-ФЗ/GDPR/PCI-DSS/ISO 27001), 18 источников |
| 28 | 08 | 02-hardening.md | done | ~7200 | 14 | Defence-in-Depth (4 слоя), hardened server.properties (18 параметров), OS hardening (file permissions, systemd, SELinux/AppArmor, FIPS 140-2), Docker hardening (non-root, distroless, read-only, seccomp, capabilities), Kubernetes hardening (PSS restricted, NetworkPolicies deny-by-default, RBAC, Strimzi Pod Security Providers), CIS Benchmark маппинг, compliance mapping (152-ФЗ/GDPR/PCI DSS/NIST/STIG), 3 перспективы с чеклистами, 25-пунктовый итоговый чеклист, 14 источников |
| 29 | 08 | 03-attack-surface.md | rewritten | ~10000 | 20 | Полный каталог 7 категорий поверхностей атаки (сетевые протоколы, клиентские API, межброкерное взаимодействие, metadata layer ZK/KRaft, экосистема Connect/Schema Registry/ksqlDB, административные интерфейсы JMX/CLI/REST, цепочка поставки), категоризация 50+ векторов по CVSS-style (Critical/High/Medium/Low), векторы атак на каждый компонент (Broker — 18 векторов, ZK — 9, KRaft — 8, Connect — 10, Schema Registry — 7, ksqlDB — 6), атаки на цепочку поставки (Java-зависимости, контейнеры, плагины Connect, Helm-чарты, Log4Shell), хронология CVE 2021-2026 (15 CVE), 3 сравнительные матрицы (Компонент×Категория, CVSS×Exploitability, Риски по окружениям), attack tree (полный, 5 ветвей), STRIDE (20 угроз), 3 перспективы (инфраструктурный безопасник, DevSecOps, архитектор ИБ), compliance mapping (152-ФЗ/GDPR/PCI-DSS/ISO 27001/SOC 2/187-ФЗ/NIST), BIA, чеклист 15 пунктов, 20 источников |
| 30 | 08 | 04-admin-security-practices.md | done | ~12500 | 24 | Переписана и значительно расширена (было 7200 слов). Добавлено: 6-шаговый hardening guide (сеть → SASL+TLS → ACL → OS hardening → контейнеры → валидация), полный secrets management (SCRAM-ротация, OAUTHBEARER, Delegation Tokens, Vault Agent architecture, keystore/truststore пароли, External Secrets Operator), compliance-автоматизация (CIS Kafka Benchmark v1.0.0, OpenSCAP, KafkaGuard, Checkov/KICS, Conftest/OPA, Continuous Compliance model), enterprise best practices (Zero Trust для Kafka, Defence in Depth 7 слоёв, segregation of duties, безопасность бэкапов+DR шифрование AES-256-GCM + air-gapped + immutable, security governance с KPI MTTD/MTTR/BIA), 2 новых incident response сценария (ransomware, CI/CD compromise). 3 перспективы с ролевыми чеклистами. Итоговый чеклист 40+ пунктов. 24 источника (+9 новых).
| 31 | 09 | 01-case-studies.md | done | ~5200 | 12 | 8 компаний (LinkedIn/Netflix/Uber/Walmart/Spotify/JD.com/Grab/Tencent + финансы + автопром), сводная таблица, дерево решений «нужен ли Kafka», 5 общих паттернов, 12 источников |
| 32 | 09 | 02-migration.md | done | ~6600 | 10 | 4 сценария: MQ→Kafka, Kafka-кластеры, ZK→KRaft, версионный апгрейд |
| 33 | 09 | 03-anti-patterns.md | done | ~6800 | 12 | 12 анти-паттернов: Kafka как БД, мало/много партиций, null-key, hot partition, acks=0/1, auto-commit, request-response, cross-AZ трафик, connect standalone, отсутствие DLQ, set-and-forget |
| 34 | 09 | 04-benchmarks-comparison.md | done | ~6500 | 13 | 13 источников, независимые бенчмарки 2023-2026, OMB-методология, Kafka vs Redpanda/Pulsar/RabbitMQ Streams/WarpStream, мифы vs реальность, облачные сервисы TCO |
| 35 | 10 | 01-roadmap.md | done | ~4800 | 12 | KIP-процесс, календарь релизов, Queues for Kafka (KIP-932), 4.0→4.2→4.3, будущие KIP 2026-2027, Confluent roadmap |
| 36 | 10 | 02-trends.md | done | ~5500 | 12 | 6 трендов: консолидация, diskless Kafka+Iceberg, shift-left аналитика, RPO=0, региональные облака, Agentic AI. Статистика рынка |
| 37 | 10 | 03-alternatives-emerging.md | done | ~7200 | 12 | 3 категории: Kafka-совместимые (Redpanda/WarpStream/AutoMQ), другая модель (Pulsar/NATS/Bufstream), стриминговые БД (RisingWave/Materialize). Сравнительная таблица, фреймворк принятия решения |

## Decisions
- Profile A: all 10 tracks, full depth
- 3 security perspectives: infrastructure security, DevSecOps, security architect
