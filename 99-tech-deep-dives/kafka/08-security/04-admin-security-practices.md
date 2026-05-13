---
title: "Административные практики безопасности Apache Kafka — Hardening, секреты, аудит, IR и compliance"
track: "08-security"
article: "04"
topic: "kafka"
word-count: 12500
sources: 24
date: 2026-05-13
---

# Административные практики безопасности Apache Kafka — Hardening, секреты, аудит, IR и compliance

**TL;DR:** Встроенные механизмы защиты (TLS, SASL, ACL) — необходимый, но недостаточный уровень. Эксплуатационная безопасность Kafka требует: (1) пошагового hardening-гайда для администраторов с проверкой каждого компонента, (2) централизованного управления секретами (SASL-пароли, ключи API, сертификаты) с автоматической ротацией, (3) интеграции с SIEM для мониторинга событий, (4) playbook реагирования на инциденты с типовыми сценариями компрометации, (5) процедур сбора forensic-данных, (6) автоматизации compliance-проверок (CIS Benchmarks, OpenSCAP, KafkaGuard, Checkov), (7) enterprise best practices — zero-trust, audit trails, DR с безопасностью. Статья даёт полный набор административных практик с трёх перспектив: инфраструктурный безопасник, DevSecOps-инженер и архитектор ИБ. Время чтения: ~65 минут.

---

## Содержание

- [1. Hardening-гайд для администраторов Kafka — пошаговые инструкции](#1-hardening-гайд-для-администраторов-kafka--пошаговые-инструкции)
  - [1.1 Предварительная подготовка — аудит текущего состояния](#11-предварительная-подготовка--аудит-текущего-состояния)
  - [1.2 Шаг 1 — сетевая изоляция](#12-шаг-1--сетевая-изоляция)
  - [1.3 Шаг 2 — включение аутентификации и шифрования](#13-шаг-2--включение-аутентификации-и-шифрования)
  - [1.4 Шаг 3 — настройка авторизации (ACL least privilege)](#14-шаг-3--настройка-авторизации-acl-least-privilege)
  - [1.5 Шаг 4 — OS-level hardening для Kafka-брокеров](#15-шаг-4--os-level-hardening-для-kafka-брокеров)
  - [1.6 Шаг 5 — безопасность Kafka в контейнерах](#16-шаг-5--безопасность-kafka-в-контейнерах)
  - [1.7 Шаг 6 — валидация hardening: чеклист и автотесты](#17-шаг-6--валидация-hardening-чеклист-и-автотесты)
- [2. Управление секретами: пароли, ключи, сертификаты](#2-управление-секретами-пароли-ключи-сертификаты)
  - [2.1 Категории секретов в экосистеме Kafka](#21-категории-секретов-в-экосистеме-kafka)
  - [2.2 Хранение SASL-паролей (SCRAM) и их ротация](#22-хранение-sasl-паролей-scram-и-их-ротация)
  - [2.3 API-ключи и токены — OAUTHBEARER, Delegation Tokens](#23-api-ключи-и-токены--oauthbearer-delegation-tokens)
  - [2.4 Управление TLS-сертификатами: Vault PKI и cert-manager](#24-управление-tls-сертификатами-vault-pki-и-cert-manager)
  - [2.5 Keystore/Truststore пароли — безопасное хранение](#25-keystoretruststore-пароли--безопасное-хранение)
  - [2.6 Полная автоматизация через Vault Agent](#26-полная-автоматизация-через-vault-agent)
  - [2.7 Ротация секретов: чеклист и процедуры](#27-ротация-секретов-чеклист-и-процедуры)
- [3. Аудит и мониторинг безопасности](#3-аудит-и-мониторинг-безопасности)
  - [3.1 SIEM-интеграция — какие события Kafka отправлять](#31-siem-интеграция--какие-события-kafka-отправлять)
  - [3.2 Категории событий для SIEM и пороги алертов](#32-категории-событий-для-siem-и-пороги-алертов)
  - [3.3 Мониторинг security-метрик через JMX/Prometheus](#33-мониторинг-security-метрик-через-jmxprometheus)
  - [3.4 Filebeat → Elasticsearch → SIEM: сквозной пример](#34-filebeat--elasticsearch--siem-сквозной-пример)
- [4. Реагирование на инциденты: playbook и сценарии](#4-реагирование-на-инциденты-playbook-и-сценарии)
  - [4.1 Четыре фазы реагирования](#41-четыре-фазы-реагирования)
  - [4.2 Сценарий 1: компрометация брокера](#42-сценарий-1-компрометация-брокера)
  - [4.3 Сценарий 2: утечка данных через несанкционированное чтение](#43-сценарий-2-утечка-данных-через-несанкционированное-чтение)
  - [4.4 Сценарий 3: ransomware / шифрование data-директорий](#44-сценарий-3-ransomware--шифрование-data-директорий)
  - [4.5 Сценарий 4: компрометация CI/CD pipeline → вредоносный deployment](#45-сценарий-4-компрометация-cicd-pipeline--вредоносный-deployment)
  - [4.6 Коммуникации и эскалация](#46-коммуникации-и-эскалация)
- [5. Forensic-сбор данных в Kafka](#5-forensic-сбор-данных-в-kafka)
  - [5.1 Порядок сбора улик — RFC 3227](#51-порядок-сбора-улик--rfc-3227)
  - [5.2 Скрипт автоматического сбора forensic-данных](#52-скрипт-автоматического-сбора-forensic-данных)
  - [5.3 Анализ логов авторизации после инцидента](#53-анализ-логов-авторизации-после-инцидента)
  - [5.4 Целостность улик и chain-of-custody](#54-целостность-улик-и-chain-of-custody)
- [6. Compliance-автоматизация: CIS Benchmarks, OpenSCAP, автоматические проверки](#6-compliance-автоматизация-cis-benchmarks-openscap-автоматические-проверки)
  - [6.1 Что такое CIS Apache Kafka Benchmark](#61-что-такое-cis-apache-kafka-benchmark)
  - [6.2 Автоматизация CIS-проверок через OpenSCAP](#62-автоматизация-cis-проверок-через-openscap)
  - [6.3 KafkaGuard — специализированный сканер безопасности Kafka](#63-kafkaguard--специализированный-сканер-безопасности-kafka)
  - [6.4 Checkov и KICS — проверка Kafka-конфигураций в IaC](#64-checkov-и-kics--проверка-kafka-конфигураций-в-iac)
  - [6.5 Conftest / OPA — policy-as-code валидация до деплоя](#65-conftest--opa--policy-as-code-валидация-до-деплоя)
  - [6.6 Conduktor-подход: непрерывная compliance-валидация](#66-conduktor-подход-непрерывная-compliance-валидация)
  - [6.7 Автоматизированный compliance-пайплайн для Kafka](#67-автоматизированный-compliance-пайплайн-для-kafka)
- [7. Enterprise best practices для безопасности Kafka](#7-enterprise-best-practices-для-безопасности-kafka)
  - [7.1 Zero Trust архитектура для Kafka](#71-zero-trust-архитектура-для-kafka)
  - [7.2 Defence in Depth для Kafka-экосистемы](#72-defence-in-depth-для-kafka-экосистемы)
  - [7.3 Segregation of duties и least privilege](#73-segregation-of-duties-и-least-privilege)
  - [7.4 Безопасность резервного копирования и DR](#74-безопасность-резервного-копирования-и-dr)
  - [7.5 Enterprise security governance для Kafka](#75-enterprise-security-governance-для-kafka)
- [8. Три перспективы безопасности](#8-три-перспективы-безопасности)
  - [8.1 Инфраструктурный безопасник](#81-инфраструктурный-безопасник)
  - [8.2 DevSecOps-инженер](#82-devsecops-инженер)
  - [8.3 Архитектор информационной безопасности](#83-архитектор-информационной-безопасности)
- [9. Итоговый чеклист административных практик](#9-итоговый-чеклист-административных-практик)
- [10. Источники](#10-источники)
- [11. Связанные статьи](#11-связанные-статьи)

---

## 1. Hardening-гайд для администраторов Kafka — пошаговые инструкции

**Ключевая мысль:** По умолчанию Kafka не имеет НИ ОДНОГО включённого механизма безопасности. Каждый production-кластер должен пройти 6-шаговый hardening: сетевая изоляция → аутентификация + шифрование → ACL least privilege → OS-level hardening → контейнерная безопасность (если применимо) → автоматическая валидация. Ниже — полный пошаговый гайд.

**Аналогия:** Настройка безопасности Kafka похожа на постройку современного банковского хранилища. Сначала вы возводите стены (сетевая изоляция), затем ставите замки на двери (аутентификация), определяете кто в какое помещение может зайти (ACL), укрепляете фундамент (OS hardening), и наконец ставите сигнализацию и камеры (аудит и валидация). Пропуск любого шага оставляет уязвимость.

### 1.1 Предварительная подготовка — аудит текущего состояния

Прежде чем hardening'ить, нужно понять, что у вас уже работает. Выполните аудит:

```bash
# 1. Проверить конфигурацию брокера
kafka-configs.sh --bootstrap-server broker1:9092 --entity-type brokers --describe --all

# 2. Проверить ACL всех ресурсов
kafka-acls.sh --bootstrap-server broker1:9092 --list

# 3. Проверить какие порты слушает процесс Kafka
ss -tlnp | grep java

# 4. Проверить файл server.properties на наличие security-параметров
grep -E "^(listeners|security|ssl|sasl|authorizer)" /etc/kafka/server.properties

# 5. Проверить версию Kafka (известные CVE)
kafka-broker-api-versions.sh --bootstrap-server broker1:9092 | head -20
```

Результат аудита даст вам baseline — отправную точку для hardening. Запишите все найденные отличия от security baseline.

### 1.2 Шаг 1 — сетевая изоляция

**Цель:** Kafka-брокеры должны быть доступны только авторизованным клиентам и другим брокерам. Никакого внешнего доступа.

**1.2.1 Файрвол на уровне ОС (iptables / nftables):**

```bash
# Политика по умолчанию — DROP для INPUT
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Разрешить loopback
iptables -A INPUT -i lo -j ACCEPT

# Разрешить established/related соединения
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Разрешить SSH с jump-хоста (админская подсеть)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/24 -j ACCEPT

# Клиентский трафик Kafka (SASL_SSL) — только из внутренних подсетей
iptables -A INPUT -p tcp --dport 9093 -s 10.0.0.0/8 -j ACCEPT

# Межброкерный трафик — только между брокерами в кластерной подсети
iptables -A INPUT -p tcp --dport 9094 -s 10.0.1.0/24 -j ACCEPT

# Controller listener (KRaft) — только controller-ноды
iptables -A INPUT -p tcp --dport 9095 -s 10.0.2.0/24 -j ACCEPT

# Prometheus JMX Exporter (если используется)
iptables -A INPUT -p tcp --dport 9404 -s 10.0.100.0/24 -j ACCEPT

# Сохранить правила
iptables-save > /etc/iptables/rules.v4
```

**1.2.2 Сетевые политики Kubernetes (если Strimzi):**

```yaml
# kafka-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-brokers-deny-all
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      strimzi.io/component-type: kafka
  policyTypes:
    - Ingress
  ingress:
    # Разрешить только от других брокеров Kafka
    - from:
        - podSelector:
            matchLabels:
              strimzi.io/component-type: kafka
      ports:
        - port: 9091  # replication
        - port: 9094  # inter-broker
    # Разрешить от клиентов в том же namespace + approved namespaces
    - from:
        - namespaceSelector:
            matchLabels:
              kafka-access: approved
      ports:
        - port: 9093  # client listener
    # Мониторинг
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - port: 9404
```

**1.2.3 Проверка открытых портов после настройки:**

```bash
# Сканирование с другого хоста в сети
nmap -sV -p 22,2181,9092,9093,9094,9095,9999,9404 kafka-broker-1.internal

# Проверка, что порт 9092 (PLAINTEXT) НЕ открыт
nmap -p 9092 kafka-broker-1.internal  # Должен показать "filtered" или "closed"
```

### 1.3 Шаг 2 — включение аутентификации и шифрования

**Цель:** Отключить PLAINTEXT-листенеры. Все подключения — через SASL_SSL (аутентификация + TLS).

**1.3.1 Базовая конфигурация в server.properties:**

```properties
# Отключить PLAINTEXT — использовать только SASL_SSL
listeners=SASL_SSL://0.0.0.0:9093

# Межброкерное общение — тоже SASL_SSL
security.inter.broker.protocol=SASL_SSL

# SASL-механизмы: только SCRAM-SHA-512 (не PLAIN!)
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# TLS 1.2 и 1.3 — ничего старше
ssl.enabled.protocols=TLSv1.2,TLSv1.3

# Сильные cipher suites (без RC4, DES, 3DES, NULL)
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

# Keystore и Truststore
ssl.keystore.location=/etc/kafka/secrets/kafka.keystore.jks
ssl.keystore.password=${KAFKA_KEYSTORE_PASSWORD}  # из переменной окружения!
ssl.key.password=${KAFKA_KEY_PASSWORD}
ssl.truststore.location=/etc/kafka/secrets/kafka.truststore.jks
ssl.truststore.password=${KAFKA_TRUSTSTORE_PASSWORD}

# Верификация hostname в сертификатах
ssl.endpoint.identification.algorithm=HTTPS

# Включить авторизацию
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer

# Super users — минимальный набор (обычно admin-принципалы)
super.users=User:admin;User:kafka-controller
```

**1.3.2 Создание SCRAM-credentials для брокеров:**

```bash
# Создать SCRAM-credentials для каждого брокера
kafka-configs.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --entity-type users --entity-name broker1 \
  --alter --add-config 'SCRAM-SHA-512=[password=Broker1SecretP@ss!2026]'

# Для межброкерной связи
kafka-configs.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --entity-type users --entity-name broker2 \
  --alter --add-config 'SCRAM-SHA-512=[password=Broker2SecretP@ss!2026]'
```

**1.3.3 JAAS-конфигурация для брокеров (kafka_server_jaas.conf):**

```
KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="broker1"
  password="Broker1SecretP@ss!2026"
  user_broker1="Broker1SecretP@ss!2026"
  user_broker2="Broker2SecretP@ss!2026"
  user_broker3="Broker3SecretP@ss!2026";
};
```

**1.3.4 Проверка:**

```bash
# Попытка подключения без аутентификации — ДОЛЖНА вернуть ошибку
kcat -b broker1:9093 -L  # Должен упасть с "Authentication failed"

# Подключение с SASL — должно работать
kcat -b broker1:9093 \
  -X security.protocol=SASL_SSL \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=admin \
  -X sasl.password=AdminP@ss!2026 \
  -L
```

### 1.4 Шаг 3 — настройка авторизации (ACL least privilege)

**Цель:** Deny by default. Разрешать только явно указанные операции для конкретных principal на конкретные ресурсы.

**1.4.1 Принципы ACL-модели:**

```bash
# Базовый принцип: DENY ALL, затем разрешать точечно
# Super users (admin, controller) имеют полный доступ без ACL
# Все остальные — только через явные ACL-правила
```

**1.4.2 Создание минимальных ACL — пример для микросервиса 'order-service':**

```bash
# order-service читает из topic 'inventory', пишет в 'orders'
# и использует consumer group 'order-service-group'

# READ на inventory (источник данных)
kafka-acls.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --add --allow-principal User:order-service \
  --operation Read --topic inventory

# WRITE на orders (выходной топик)
kafka-acls.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --add --allow-principal User:order-service \
  --operation Write --topic orders

# READ на consumer group
kafka-acls.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --add --allow-principal User:order-service \
  --operation Read --group order-service-group
```

**1.4.3 Антипаттерны ACL — чего НЕ делать:**

```bash
# ❌ Wildcard на все топики
kafka-acls.sh --add --allow-principal User:some-service --operation Read --topic '*'  # КАТЕГОРИЧЕСКИ НЕТ!

# ❌ Wildcard на все группы
kafka-acls.sh --add --allow-principal User:some-service --operation Read --group '*'  # НЕТ!

# ❌ ALL operations (Read + Write + Create + Delete + Describe + Alter + ...)
kafka-acls.sh --add --allow-principal User:some-service --operation All --topic orders  # Только если точно нужно!

# ✅ Правильно: точечные разрешения
kafka-acls.sh --add --allow-principal User:some-service --operation Read --topic orders
```

**1.4.4 Аудит ACL перед продакшеном:**

```bash
# Вывести все ACL и проверить на wildcards
kafka-acls.sh --bootstrap-server broker1:9093 --list | grep -E '\*|All'

# Проверить ACL конкретного principal
kafka-acls.sh --bootstrap-server broker1:9093 --list --principal User:order-service
```

### 1.5 Шаг 4 — OS-level hardening для Kafka-брокеров

**Цель:** Защитить хосты, на которых работает Kafka. Минимизировать поверхность атаки на уровне ОС.

**1.5.1 Systemd-hardening (kafka.service):**

```ini
# /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Broker
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

# --- Hardening directives ---
# Изолировать /tmp
PrivateTmp=true

# Защитить системные директории от записи
ProtectSystem=strict
ReadWritePaths=/var/lib/kafka /var/log/kafka /opt/kafka/tmp

# Запретить доступ к /home, /root
ProtectHome=true

# Запретить получение новых привилегий
NoNewPrivileges=true

# Заблокировать изменение kernel-параметров
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true

# Ограничить сетевые семейства
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Запретить system calls которые не нужны Kafka
SystemCallFilter=~@clock @debug @module @mount @obsolete @raw-io @reboot @swap

# Запретить namespace creation
RestrictNamespaces=true

# Установка лимитов
LimitNOFILE=131072
LimitNPROC=32768

[Install]
WantedBy=multi-user.target
```

**1.5.2 File permissions:**

```bash
# Владелец data и log директорий — kafka:kafka
chown -R kafka:kafka /var/lib/kafka/
chown -R kafka:kafka /var/log/kafka/

# Конфиги — read-only для kafka, только root может менять
chown root:root /etc/kafka/server.properties
chmod 644 /etc/kafka/server.properties

# Секреты (keystore, truststore, JAAS) — только kafka может читать
chown kafka:kafka /etc/kafka/secrets/
chmod 700 /etc/kafka/secrets/
chmod 600 /etc/kafka/secrets/*

# Бинарники Kafka — root:root, read/execute
chown -R root:root /opt/kafka/bin/
chmod -R 755 /opt/kafka/bin/
```

**1.5.3 Auditd-правила для мониторинга OS-level событий:**

```bash
# /etc/audit/rules.d/kafka.rules

# Отслеживать изменения server.properties
-w /etc/kafka/server.properties -p wa -k kafka_config_change

# Отслеживать доступ к data-директориям (только запись и удаление)
-w /var/lib/kafka/data/ -p wa -k kafka_data_access

# Отслеживать изменения логов (попытки заметания следов)
-w /var/log/kafka/ -p wa -k kafka_log_modify

# Отслеживать доступ к секретам
-w /etc/kafka/secrets/ -p rwa -k kafka_secrets_access

# Отслеживать выполнение Kafka-бинарников (нестандартные аргументы?)
-w /opt/kafka/bin/kafka-server-start.sh -p x -k kafka_binary_exec

# Применить правила
auditctl -R /etc/audit/rules.d/kafka.rules
```

**1.5.4 SELinux / AppArmor (опционально, для high-security сред):**

Если вы используете SELinux, создайте custom policy для Kafka:

```bash
# Создать policy module
cat > kafka.te << 'EOF'
module kafka 1.0;

require {
    type kafka_t;
    type kafka_log_t;
    type kafka_data_t;
    type kafka_config_t;
    type kafka_secrets_t;
    type kafka_port_t;
    class file { read write create open getattr setattr unlink };
    class dir { read write search add_name remove_name };
    class tcp_socket { name_bind name_connect };
}

# Разрешить Kafka-процессу:
allow kafka_t kafka_log_t:file { read write create open };
allow kafka_t kafka_data_t:file { read write create open unlink };
allow kafka_t kafka_config_t:file { read open getattr };
allow kafka_t kafka_secrets_t:file { read open getattr };
allow kafka_t kafka_port_t:tcp_socket name_bind;
EOF

# Скомпилировать и установить
checkmodule -M -m -o kafka.mod kafka.te
semodule_package -o kafka.pp -m kafka.mod
semodule -i kafka.pp
```

### 1.6 Шаг 5 — безопасность Kafka в контейнерах

**Цель:** Если Kafka запускается в Docker/Kubernetes — hardening контейнеров обязателен.

**1.6.1 Docker — безопасный Dockerfile для Kafka:**

```dockerfile
# Dockerfile — безопасный образ Kafka
FROM apache/kafka:3.9.0

# Создать непривилегированного пользователя
RUN groupadd -r kafka -g 1000 && \
    useradd -r -g kafka -u 1000 -d /opt/kafka kafka && \
    mkdir -p /var/lib/kafka/data /var/log/kafka && \
    chown -R kafka:kafka /opt/kafka /var/lib/kafka /var/log/kafka

# Переключиться на непривилегированного пользователя
USER kafka:kafka

# Скопировать hardened конфиг
COPY --chown=kafka:kafka server.properties /etc/kafka/
COPY --chown=kafka:kafka --chmod=600 secrets/ /etc/kafka/secrets/

EXPOSE 9093
```

**Docker run с security-опциями:**

```bash
docker run -d \
  --name kafka-broker-1 \
  --user 1000:1000 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=256M \
  --tmpfs /var/log/kafka:rw,noexec,size=2G \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --security-opt seccomp=/etc/docker/seccomp-kafka.json \
  --memory=4g \
  --memory-swap=4g \
  --pids-limit=1000 \
  -v /data/kafka:/var/lib/kafka/data:rw \
  kafka-hardened:3.9.0
```

**1.6.2 Kubernetes — Strimzi pod security:**

```yaml
# Strimzi Kafka spec с security context
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: secure-cluster
spec:
  kafka:
    version: 3.9.0
    replicas: 3
    template:
      pod:
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          fsGroup: 1000
          seccompProfile:
            type: RuntimeDefault
        topologySpreadConstraints:
          - maxSkew: 1
            topologyKey: kubernetes.io/hostname
            whenUnsatisfiable: DoNotSchedule
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: scram-sha-512
      - name: replication
        port: 9094
        type: internal
        tls: true
        authentication:
          type: scram-sha-512
    authorization:
      type: simple
      superUsers:
        - admin
    config:
      ssl.enabled.protocols: "TLSv1.2,TLSv1.3"
      ssl.cipher.suites: "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
      ssl.endpoint.identification.algorithm: "HTTPS"
```

### 1.7 Шаг 6 — валидация hardening: чеклист и автотесты

**Цель:** Автоматизировать проверку hardening, чтобы ни один брокер не отклонился от baseline.

**1.7.1 Скрипт валидации hardening (hardening-validate.sh):**

```bash
#!/bin/bash
# hardening-validate.sh — автоматическая проверка hardening Kafka-брокера
BROKER=$1
EXIT_CODE=0

echo "=== Kafka Hardening Validation for $BROKER ==="

# Проверка 1: PLAINTEXT listener НЕ открыт
echo -n "[1] PLAINTEXT listener (9092) closed... "
if nmap -p 9092 $BROKER | grep -q "open"; then
    echo "❌ FAIL — port 9092 is OPEN!"
    EXIT_CODE=1
else
    echo "✅ PASS"
fi

# Проверка 2: SASL_SSL работает
echo -n "[2] SASL_SSL listener (9093) accepting... "
if kcat -b $BROKER:9093 -X security.protocol=SASL_SSL \
    -X sasl.mechanisms=SCRAM-SHA-512 \
    -X sasl.username=validator -X sasl.password=validator-pass -L &>/dev/null; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    EXIT_CODE=1
fi

# Проверка 3: Подключение без SASL отклоняется
echo -n "[3] Unauthenticated connection rejected... "
if kcat -b $BROKER:9093 -X security.protocol=SSL -L 2>&1 | grep -qi "authentication"; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    EXIT_CODE=1
fi

# Проверка 4: TLS 1.0/1.1 отключены
echo -n "[4] TLS 1.0/1.1 disabled... "
if openssl s_client -connect $BROKER:9093 -tls1 2>&1 | grep -q "CONNECTED"; then
    echo "❌ FAIL — TLS 1.0 works!"
    EXIT_CODE=1
else
    echo "✅ PASS"
fi

# Проверка 5: ACL authorizer включён
echo -n "[5] ACL authorizer enabled... "
if kafka-acls.sh --bootstrap-server $BROKER:9093 \
    --command-config admin.conf --list &>/dev/null; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    EXIT_CODE=1
fi

# Проверка 6: Файловые права на секреты
echo -n "[6] Secrets file permissions (600)... "
SECRETS_DIR="/etc/kafka/secrets"
if [ -d "$SECRETS_DIR" ]; then
    BAD_PERMS=$(find "$SECRETS_DIR" -type f ! -perm 600 | wc -l)
    if [ "$BAD_PERMS" -eq 0 ]; then
        echo "✅ PASS"
    else
        echo "❌ FAIL — $BAD_PERMS file(s) with insecure permissions"
        EXIT_CODE=1
    fi
else
    echo "⚠️ SKIP — directory not found"
fi

# Проверка 7: Kafka процесс не от root
echo -n "[7] Kafka running as non-root... "
KAFKA_USER=$(ps aux | grep kafka.Kafka | grep -v grep | awk '{print $1}' | head -1)
if [ "$KAFKA_USER" != "root" ] && [ -n "$KAFKA_USER" ]; then
    echo "✅ PASS (user: $KAFKA_USER)"
else
    echo "❌ FAIL — running as $KAFKA_USER"
    EXIT_CODE=1
fi

echo ""
echo "=== Result: $([ $EXIT_CODE -eq 0 ] && echo '✅ ALL CHECKS PASSED' || echo '❌ SOME CHECKS FAILED') ==="
exit $EXIT_CODE
```

**1.7.2 Интеграция в CI/CD:**

Добавьте запуск hardening-validate.sh как финальный step в пайплайне деплоя — после rolling update брокеров. Если проверка не прошла — rollback и алерт.

---

## 2. Управление секретами: пароли, ключи, сертификаты

**Ключевая мысль:** Секреты в Kafka — это не только TLS-сертификаты. Полная картина включает: SASL-пароли (особенно PLAIN — plaintext в JAAS!), SCRAM credentials, OAUTHBEARER-токены, Delegation Tokens, API-ключи для Schema Registry/Connect, пароли keystore/truststore. Каждый тип секрета требует своего подхода к хранению и ротации.

### 2.1 Категории секретов в экосистеме Kafka

| Тип секрета | Где хранится по умолчанию | Риск | Правильное хранение |
|-------------|---------------------------|------|---------------------|
| SASL/PLAIN пароли | JAAS config file (plaintext!) | CRITICAL — plaintext пароль в файловой системе | Vault KV Secret Engine |
| SASL/SCRAM credentials | ZooKeeper (ZK-based) / KRaft metadata log | HIGH — доступны любому с доступом к ZK/metadata | ZK/KRaft + ACL, пароли вне конфигов |
| TLS private keys | keystore JKS/PKCS12 (зашифрованы паролем) | MEDIUM — если пароль keystore слабый | HSM / Vault PKI |
| Keystore/truststore пароли | server.properties (plaintext!) | CRITICAL | Vault Agent inject env var |
| OAUTHBEARER JWT signing keys | Файловая система IdP | HIGH — компрометация = выпуск любых токенов | Vault Transit / HSM |
| Delegation Tokens | Генерируются брокером, кэшируются клиентом | MEDIUM — lifetime hours, replay possible | mTLS + короткий TTL |
| Connect/Schema Registry API keys | Файл конфигурации connect-distributed.properties | HIGH | Vault / External Secrets Operator |
| JMX credentials | jmxremote.password (plaintext!) | MEDIUM | Зашифрованный файл или отключить JMX network |
| KRaft metadata encryption key (KIP-965) | Зашифрован мастер-ключом в файловой системе | HIGH — доступ к данным metadata | KMS / Vault Transit |

**Аналогия:** Представьте, что все пароли и ключи вашей Kafka — это ключи от сейфов в банке. Если вы храните их под ковриком у входа (JAAS plaintext), злоумышленнику не нужно взламывать сейф — достаточно открыть дверь и забрать ключи.

### 2.2 Хранение SASL-паролей (SCRAM) и их ротация

**2.2.1 Где Kafka хранит SCRAM-credentials:**

- **ZooKeeper mode:** `/config/users/<user>` (Base64-encoded SCRAM credential)
- **KRaft mode:** в metadata log (доступ через `kafka-configs.sh`)

Сами credentials — salted, iterated hash (SCRAM-SHA-256 = 4096 итераций, SCRAM-SHA-512 = 4096). Это защищает от rainbow table attacks, но НЕ от перебора, если злоумышленник скопирует ZK/KRaft metadata.

**2.2.2 Безопасное создание SCRAM-пользователей:**

```bash
# ❌ НЕПРАВИЛЬНО: передача пароля в командной строке (виден в history/ps)
kafka-configs.sh --alter --add-config 'SCRAM-SHA-512=[password=MyP@ss]'

# ✅ ПРАВИЛЬНО: интерактивный ввод или из Vault
VAULT_PASS=$(vault kv get -field=scram-password secret/kafka/users/app-service)
echo "SCRAM-SHA-512=[password=$VAULT_PASS]" > /tmp/scram-config.tmp
kafka-configs.sh --alter --add-config-file /tmp/scram-config.tmp
shred -u /tmp/scram-config.tmp
```

**2.2.3 Ротация SCRAM-паролей (с нулевым downtime):**

```bash
#!/bin/bash
# rotate-scram.sh — безопасная ротация SCRAM-пароля для сервиса
USER=$1
OLD_PASS=$2
NEW_PASS=$3

# Шаг 1: Генерировать новый пароль в Vault
vault write secret/kafka/users/$USER \
    scram-password=$NEW_PASS \
    scram-password-previous=$OLD_PASS \
    rotated_at=$(date -Iseconds)

# Шаг 2: Обновить SCRAM-credential в Kafka (старый пароль ещё валиден для клиентов)
kafka-configs.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --entity-type users --entity-name $USER \
  --alter --add-config "SCRAM-SHA-512=[password=$NEW_PASS]"

# Шаг 3: Дождаться propagation по кластеру (обычно секунды)
sleep 10

# Шаг 4: Поочерёдно перезапустить инстансы сервиса с новым паролем
# (через CI/CD или rolling restart)

# Шаг 5: Через 24 часа, когда все инстансы обновлены,
# запланировать удаление старого пароля из Vault
echo "vault write secret/kafka/users/$USER scram-password-previous=''" | \
  at now + 24 hours
```

### 2.3 API-ключи и токены — OAUTHBEARER, Delegation Tokens

**2.3.1 OAUTHBEARER — централизованное управление:**

В enterprise-среде все токены OAuth управляются через Identity Provider (IdP) — Keycloak, Azure AD, Okta. Kafka не хранит токены — только валидирует их через JWKS (JSON Web Key Set).

```properties
# server.properties — OAUTHBEARER конфигурация
sasl.enabled.mechanisms=OAUTHBEARER
listener.name.sasl_ssl.oauthbearer.sasl.jaas.config= \
  org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  jwksEndpointUri="https://keycloak.internal/realms/kafka/protocol/openid-connect/certs" \
  validIssuerUri="https://keycloak.internal/realms/kafka" \
  expectedAudience="kafka-broker";
```

**Ротация OAuth-ключей на стороне IdP — автоматическая, Kafka получает новые ключи через JWKS endpoint без перезапуска.**

**2.3.2 Delegation Tokens — особенности:**

Delegation Tokens — механизм делегирования аутентификации без раскрытия основных credentials. Полезен для long-running jobs (Spark, Flink).

```bash
# Создать delegation token (от имени admin)
kafka-delegation-tokens.sh --bootstrap-server broker1:9093 \
  --command-config /etc/kafka/admin-client.conf \
  --create --renewer-principal User:admin --max-life-time-period 86400000
```

**Риски Delegation Tokens:**
- Токены валидны до истечения lifetime (обычно 24 часа)
- Не могут быть отозваны досрочно (в отличие от OAuth)
- Если токен украден — злоумышленник имеет доступ до истечения

**Рекомендация:** Используйте минимальный lifetime. Для Kafka 4.0+ используйте OAuth вместо Delegation Tokens.

### 2.4 Управление TLS-сертификатами: Vault PKI и cert-manager

**2.4.1 Vault PKI — architecture overview:**

- Vault PKI Secrets Engine действует как Certificate Authority (CA)
- Vault Agent на каждом брокере автоматически запрашивает и обновляет сертификаты
- Vault Agent template переписывает keystore и автоматически перезапускает Kafka

**2.4.2 Vault Agent конфигурация для Kafka-брокера:**

```hcl
# vault-agent-kafka.hcl
pid_file = "/var/run/vault-agent.pid"

auto_auth {
  method {
    type = "approle"
    config = {
      role_id_file_path   = "/etc/vault/role-id"
      secret_id_file_path = "/etc/vault/secret-id"
    }
  }
}

# Шаблон для генерации сертификата и ключа
template {
  source      = "/etc/vault/templates/kafka-cert.tmpl"
  destination = "/etc/kafka/secrets/kafka-keystore.p12"
  command     = "systemctl restart kafka"
  perms       = 0600
}

# Cache для снижения нагрузки на Vault
cache {
  use_auto_auth_token = true
}

listener "unix" {
  address     = "/var/run/vault-agent.sock"
  tls_disable = true
}
```

**Template для keystore (kafka-cert.tmpl):**

```
{{- with pkiCert "pki/issue/kafka-broker"
    "common_name=kafka-broker-1.internal"
    "alt_names=kafka-broker-1,kafka-cluster.internal"
    "ttl=720h"
    "exclude_cn_from_sans=true" -}}
{{ .Data.private_key }}
{{ .Data.certificate }}
{{ .Data.ca_chain }}
{{- end }}
```

**Ключевые параметры PKI:**
- TTL сертификатов: 30-90 дней (баланс безопасности и нагрузки на Vault)
- Grace period: 30% от TTL (обновление начинается за 9 дней до истечения для 30d TTL)
- Vault Agent проверяет срок каждые 60 секунд
- Automatic restart Kafka после обновления keystore

**2.4.3 cert-manager для Kubernetes:**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-brokers-tls
  namespace: kafka
spec:
  secretName: kafka-brokers-tls
  duration: 2160h    # 90 дней
  renewBefore: 720h  # renew за 30 дней до истечения
  privateKey:
    algorithm: RSA
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - "*.kafka-brokers.kafka.svc.cluster.local"
    - "kafka-bootstrap.kafka.svc"
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
```

### 2.5 Keystore/Truststore пароли — безопасное хранение

**Критическая проблема:** По умолчанию пароли keystore/truststore хранятся в PLAINTEXT в `server.properties`:

```properties
# ❌ Так делать НЕЛЬЗЯ:
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.password=changeit
```

**Решение 1 — переменные окружения (для bare metal):**

```properties
# server.properties — ссылаемся на переменные окружения
ssl.keystore.password=${KAFKA_KEYSTORE_PASSWORD}
ssl.key.password=${KAFKA_KEY_PASSWORD}
ssl.truststore.password=${KAFKA_TRUSTSTORE_PASSWORD}
```

Systemd unit запускает Kafka с переменными, которые инжектит Vault Agent:

```ini
[Service]
EnvironmentFile=-/etc/kafka/secrets/env    # Vault Agent перезаписывает этот файл
ExecStartPre=/usr/local/bin/vault-agent-template  # Генерирует env файл из Vault
ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/server.properties
```

**Решение 2 — External Secrets Operator (Kubernetes):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: kafka-keystore-passwords
  namespace: kafka
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: kafka-keystore-secrets
  data:
    - secretKey: keystore-password
      remoteRef:
        key: kafka/tls
        property: keystore-password
    - secretKey: key-password
      remoteRef:
        key: kafka/tls
        property: key-password
    - secretKey: truststore-password
      remoteRef:
        key: kafka/tls
        property: truststore-password
```

### 2.6 Полная автоматизация через Vault Agent

**Итоговая архитектура управления секретами Kafka с Vault:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Vault       │     │  Vault Agent │     │  Kafka        │
│  Server      │────▶│  (на брокере)│────▶│  Broker       │
│              │     │              │     │              │
│  • PKI CA    │     │  • AppRole   │     │  • keystore  │
│  • KV Store  │     │  • Templates │     │  • truststore│
│  • Transit   │     │  • Auto-rest.│     │  • env vars  │
└──────────────┘     └──────────────┘     └──────────────┘
```

Все секреты живут в Vault. Vault Agent на брокере:
1. Запрашивает TLS-сертификат из PKI Engine (если срок истекает)
2. Извлекает keystore/truststore пароли из KV Engine
3. Генерирует environment file и keystore
4. Рестартует Kafka

Ни один секрет не хранится в файловой системе брокера дольше времени генерации template.

### 2.7 Ротация секретов: чеклист и процедуры

| Секрет | Периодичность ротации | Метод | Downtime |
|--------|----------------------|-------|----------|
| TLS-сертификаты брокеров | 30-90 дней | Vault PKI + Vault Agent → auto-restart Kafka | Нет (rolling restart) |
| Keystore/Truststore пароли | 90 дней | Vault KV → Vault Agent env inject | Нет (при rolling restart сертификатов) |
| SCRAM-пароли сервисов | 90 дней | kafka-configs.sh через скрипт | Нет (grace period для клиентов) |
| OAUTHBEARER signing keys | 30 дней | IdP автоматически, Kafka подхватывает через JWKS | Нет |
| Delegation Tokens | Максимум 24 часа | Автоматически истекают | Нет |
| Connect/SR API keys | 90 дней | Vault → External Secrets Operator | Нет (K8s Secret refresh) |
| Vault AppRole Secret ID | 7-30 дней | Vault Agent cache refresh | Нет (Vault Agent обрабатывает) |

**Кейс Zendesk:** Команда автоматизировала ротацию Root CA через Vault PKI — устранили риск истечения сертификатов для multi-region Kafka-кластеров (Zendesk Engineering, 2025).

---

## 3. Аудит и мониторинг безопасности

**Ключевая мысль:** Без SIEM-интеграции администратор Kafka слеп к событиям безопасности. Нужно собирать три уровня: (1) Kafka authorizer-логи (ACL-решения), (2) JMX-метрики безопасности, (3) OS-level audit-логи. Всё — централизованно, с алертами на аномалии.

### 3.1 SIEM-интеграция — какие события Kafka отправлять

Authorizer-logger — встроенный механизм Kafka, фиксирующий каждое ACL-решение.

**Настройка отдельного appender (log4j2):**

```properties
# log4j2.properties
appender.authorizer.type = RollingFile
appender.authorizer.name = authorizerAppender
appender.authorizer.fileName = /var/log/kafka/kafka-authorizer.log
appender.authorizer.filePattern = /var/log/kafka/kafka-authorizer.%d{yyyy-MM-dd-HH}.log
appender.authorizer.layout.type = PatternLayout
appender.authorizer.layout.pattern = [%d] %p %m (%c)%n

logger.authorizer.name = kafka.authorizer.logger
logger.authorizer.level = DEBUG
logger.authorizer.appenderRef.authorizer.ref = authorizerAppender
logger.authorizer.additivity = false
```

**Trade-off:** На кластере с 10 000 запросов/сек DEBUG authorizer-logger генерирует гигабайты в час. Для таких нагрузок рассмотрите INFO уровень + JMX-счётчики вместо DEBUG.

### 3.2 Категории событий для SIEM и пороги алертов

| Категория | Событие | Источник | Приоритет | Порог алерта |
|-----------|---------|----------|-----------|-------------|
| Аутентификация | Неудачные SASL-рукопожатия | `kafka.network` | **CRITICAL** | >10/мин с одного IP |
| Аутентификация | Успешный вход с admin principal с нового IP | `kafka.authorizer.logger` | **CRITICAL** | Любое событие |
| Авторизация | ACL DENIED на sensitive topics | `kafka.authorizer.logger` | **HIGH** | Любое DENIED на PII/Payments |
| Авторизация | Массовые DENIED (>5/мин от principal) | `kafka.authorizer.logger` | **HIGH** | >5/мин |
| Административные | Создание/удаление топика | `kafka.controller` | **HIGH** | >5/мин |
| Административные | Изменение ACL | `kafka.authorizer.logger` | **CRITICAL** | Любое изменение не от admin |
| Административные | Изменение server.properties/broker configs | `kafka.server` | **HIGH** | Любое изменение |
| Кластер | ISR shrink (партиция теряет реплики) | JMX | **MEDIUM** | <min.insync.replicas |
| Кластер | Under-replicated partitions > 0 >5 мин | JMX | **MEDIUM** | >0 дольше 5 мин |

### 3.3 Мониторинг security-метрик через JMX/Prometheus

Ключевые JMX-метрики для безопасности:

```yaml
# prometheus-alerts-kafka-security.yaml
groups:
  - name: kafka_security
    rules:
      - alert: KafkaAuthenticationFailures
        expr: rate(kafka_network_authentication_failures_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Высокая частота отказов аутентификации Kafka"
          description: "Брокер {{ $labels.instance }}: {{ $value }}/s отказов SASL/TLS"

      - alert: KafkaUnauthorizedAccess
        expr: rate(kafka_authorizer_denied_total[5m]) > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Частые ACL DENIED в Kafka"
          description: "Principal {{ $labels.principal }}: {{ $value }}/s отказов авторизации"

      - alert: KafkaACLChanged
        expr: increase(kafka_authorizer_acl_changes_total[10m]) > 0
        labels:
          severity: info
        annotations:
          summary: "ACL Kafka были изменены"
```

### 3.4 Filebeat → Elasticsearch → SIEM: сквозной пример

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/kafka/kafka-authorizer.log
      - /var/log/kafka/server.log
    fields:
      log_type: kafka_security
      cluster: production-us-east
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after

  # OS-level audit логи
  - type: log
    enabled: true
    paths:
      - /var/log/audit/audit.log
    fields:
      log_type: os_audit

output.elasticsearch:
  hosts: ["https://elasticsearch.internal:9200"]
  index: "kafka-security-%{+yyyy.MM.dd}"
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
```

---

## 4. Реагирование на инциденты: playbook и сценарии

**Ключевая мысль:** Компрометация Kafka-кластера — не гипотетический сценарий. Наличие playbook сокращает время реакции с часов до минут. Ниже — 4 детальных сценария с хронометражом.

### 4.1 Четыре фазы реагирования

**Фаза 1 — Подготовка (до инцидента):**
- Назначить IRT: security analyst + Kafka administrator + SRE
- Подготовить каналы связи: Slack #kafka-incidents, PagerDuty
- Провести tabletop-учения (TTX) минимум раз в квартал
- Централизовать все журналы, обеспечить доступ IRT (не только админам)
- Задокументировать baseline: профиль трафика, известные consumer groups, ожидаемые ACL

**Фаза 2 — Обнаружение и анализ:**
- Подтвердить через корреляцию SIEM + JMX + алерты приложений
- Scope: все брокеры или конкретные? Только read или write и админ?
- Классификация: P1 (data exfiltration, total outage) / P2 (suspicious) / P3 (audit finding)

**Фаза 3 — Сдерживание, устранение, восстановление:**
- Сдерживание: сетевая изоляция ПЕРЕД остановкой процессов (сохранить forensic-данные!)
- Устранение: отозвать credentials, пропатчить, перестроить образ
- Восстановление: чистый образ, проверка целостности, переключение трафика

**Фаза 4 — Post-mortem:**
- Timeline с точностью до минут, root cause, обновить playbook/политики/мониторинг

### 4.2 Сценарий 1: компрометация брокера

**Симптомы:** SIEM-алерт — SSH/RDP с нестандартного IP, процесс Kafka от нестандартного пользователя, аномальный трафик на внешний адрес :9092.

**Runbook (T+0 — T+20 мин):**

1. **T+0:** Объявить инцидент. War room.
2. **T+3:** Сетевая изоляция скомпрометированного брокера:
   ```bash
   iptables -A OUTPUT -p tcp --dport 9092 -j DROP  # Блокировать внешний Kafka
   iptables -A OUTPUT -p tcp --dport 9093 -j DROP
   iptables-save > /tmp/forensic/iptables-$(date +%s).txt  # Сохранить для forensics
   ```
3. **T+5:** Дамп процессов и соединений:
   ```bash
   netstat -tulpn > /tmp/forensic/netstat.txt
   ps auxf > /tmp/forensic/ps.txt
   lsof -p $(pgrep -f kafka) > /tmp/forensic/lsof-kafka.txt
   ```
4. **T+10:** Если брокер — лидер: Kafka автоматически выберет нового из ISR.
5. **T+12:** Снять образ диска для offline-анализа (dd / volume snapshot на WORM-носитель).
6. **T+15:** Уничтожить compromised instance. Развернуть из golden image.
7. **T+18:** Проверить, что новый брокер вошёл в кластер и догнал репликацию.
8. **T+20:** Проверить ACL — не добавлены ли вредоносные правила.

### 4.3 Сценарий 2: утечка данных через несанкционированное чтение

**Симптомы:** authorizer.log показывает READ от principal без ACL, нестандартная consumer group читает sensitive topic.

**Runbook (T+0 — T+20 мин):**

1. **T+0:** Подтвердить:
   ```bash
   grep "Principal = User:suspicious.*Operation = Read.*resource = Topic:LITERAL:payments" \
     /var/log/kafka/kafka-authorizer.log | tail -50
   ```
2. **T+3:** Заблокировать principal: отозвать SASL-credentials или сетевой блок.
3. **T+5:** Определить объём: какие партиции, с какого offset, как долго.
4. **T+10:** Root cause: почему ACL пропустил? Wildcard? Унаследованное правило?
5. **T+15:** Исправить ACL, аудит всех ACL на sensitive topics.
6. **T+20:** Задокументировать для compliance officer / DPO.

### 4.4 Сценарий 3: ransomware / шифрование data-директорий

**Симптомы:** Массовый consumer lag > порога, алерты от приложений-потребителей, брокер не может прочитать сегменты лога (IOException в server.log).

**Особенность Kafka:** Data-директории — основной вектор для ransomware. Злоумышленник, получивший доступ к хосту брокера, может зашифровать `/var/lib/kafka/data/`, парализовав кластер.

**Runbook:**

1. **T+0:** Изолировать поражённый брокер.
2. **T+5:** Проверить replication factor: если RF=3 и min.insync.replicas=2, потеря одного брокера не критична.
3. **T+10:** Если поражены несколько брокеров → проверка, что ISR не опустели полностью. Если ISR=0 для каких-либо партиций — DATA LOSS.
4. **T+15:** Восстановление: из бэкапа (tiered storage KIP-405) или репликация с уцелевших реплик.
5. **T+30:** Post-mortem: как злоумышленник получил доступ к хосту? Обновить OS hardening.

### 4.5 Сценарий 4: компрометация CI/CD pipeline → вредоносный deployment

**Симптомы:** ArgoCD/Flux синхронизирует неавторизованное изменение конфигурации Kafka (например, отключение TLS или добавление внешнего листенера).

**Runbook:**

1. **T+0:** Блокировать GitOps-синхронизацию: ArgoCD → pause auto-sync.
2. **T+5:** Rollback к последней известной хорошей конфигурации.
3. **T+10:** Аудит: чей GPG/SSH ключ подписал вредоносный коммит? Скомпрометирован ли ключ?
4. **T+15:** Revoke скомпрометированный ключ. Обновить required signers в ArgoCD.
5. **T+30:** Обновить pipeline: добавить проверку OPA/Conftest на отключение TLS.

### 4.6 Коммуникации и эскалация

| Роль | Когда | Канал |
|------|-------|-------|
| Kafka Admin on-call | T+0 немедленно | PagerDuty |
| Security IRT Lead | T+5 | Slack + phone |
| CISO / DPO | При подтверждённой утечке | Email + phone |
| Affected data owners | После сдерживания | Email |
| Регулятор (РКН) | По 152-ФЗ — 24-72 ч | Официальное уведомление |

---

## 5. Forensic-сбор данных в Kafka

**Ключевая мысль:** Первое правило forensics — не уничтожить улики. Второе — собирать в правильном порядке: от наиболее волатильных (память, процессы, сетевые соединения) к наименее волатильным (дисковые образы, бэкапы).

### 5.1 Порядок сбора улик — RFC 3227

Order of Volatility (от нестабильных к стабильным):
1. Регистры CPU, кэши → (неприменимо к админам Kafka без JTAG)
2. Память (RAM) — `/proc/kcore`, `fmem`, LiME
3. Состояние сети — `netstat`, `ss`, ARP cache, conntrack table
4. Процессы — `ps auxf`, `/proc/<pid>/`
5. Дисковая активность — Kafka log segments, файловая система
6. Физические носители — disk image (dd), volume snapshot

### 5.2 Скрипт автоматического сбора forensic-данных

```bash
#!/bin/bash
# kafka-forensic-collect.sh — полный сбор
FORENSIC_DIR="/mnt/forensic/$(hostname)-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$FORENSIC_DIR"

echo "[*] Collecting forensic data to $FORENSIC_DIR"

# 1. Память (если доступно)
cat /proc/kcore > "$FORENSIC_DIR/memory.dump" 2>/dev/null || \
  echo "   ⚠ Memory dump requires root" >> "$FORENSIC_DIR/collection.log"

# 2. Сеть
netstat -tulpn > "$FORENSIC_DIR/netstat.txt"
ss -tulpn > "$FORENSIC_DIR/ss.txt"
conntrack -L > "$FORENSIC_DIR/conntrack.txt" 2>/dev/null || true

# 3. Процессы
ps auxf > "$FORENSIC_DIR/ps.txt"
# Информация о процессе Kafka
PID=$(pgrep -f kafka.Kafka)
if [ -n "$PID" ]; then
    lsof -p "$PID" > "$FORENSIC_DIR/lsof-kafka.txt"
    cat /proc/$PID/cmdline | tr '\0' ' ' > "$FORENSIC_DIR/cmdline.txt"
    cat /proc/$PID/environ | tr '\0' '\n' > "$FORENSIC_DIR/environ.txt"  # осторожно: могут быть секреты!
    ls -la /proc/$PID/fd > "$FORENSIC_DIR/fd-list.txt"
fi

# 4. Kafka-специфичное
kafka-acls.sh --bootstrap-server localhost:9093 --list > "$FORENSIC_DIR/acls.txt" 2>/dev/null
kafka-topics.sh --bootstrap-server localhost:9093 --describe > "$FORENSIC_DIR/topics.txt" 2>/dev/null
kafka-consumer-groups.sh --bootstrap-server localhost:9093 --list > "$FORENSIC_DIR/consumer-groups.txt" 2>/dev/null
kafka-metadata-quorum.sh --bootstrap-server localhost:9093 --describe > "$FORENSIC_DIR/metadata.txt" 2>/dev/null

# 5. Логи
cp -r /var/log/kafka/ "$FORENSIC_DIR/logs/"
cp /etc/kafka/server.properties "$FORENSIC_DIR/"
cp -r /etc/kafka/ "$FORENSIC_DIR/kafka-config/"
ls -laR /var/lib/kafka/data/ > "$FORENSIC_DIR/data-dir-listing.txt"

# 6. Системные логи
cp /var/log/syslog "$FORENSIC_DIR/" 2>/dev/null || true
cp /var/log/auth.log "$FORENSIC_DIR/" 2>/dev/null || true
cp /var/log/audit/audit.log "$FORENSIC_DIR/" 2>/dev/null || true

# 7. Хэши для chain-of-custody
find "$FORENSIC_DIR" -type f -exec sha256sum {} \; > "$FORENSIC_DIR/hashes.txt"

echo "[+] Complete. Archive size: $(du -sh $FORENSIC_DIR | cut -f1)"
```

### 5.3 Анализ логов авторизации после инцидента

```bash
# Кто обращался к конкретному sensitive topic?
grep "resource = Topic:LITERAL:payments" kafka-authorizer.log | \
  grep -oP "Principal = \K[^,]+" | sort | uniq -c | sort -rn

# Какие операции выполнял подозрительный principal?
grep "Principal = User:suspicious-app" kafka-authorizer.log | \
  grep -oP "Operation = \K\w+" | sort | uniq -c

# Хронология действий principal (timeline reconstruction):
grep "Principal = User:suspicious-app" kafka-authorizer.log | \
  grep -oP '^\[\K[^\]]+' | sort -n | head -20

# С каких IP подключался principal:
grep "Principal = User:suspicious-app" kafka-authorizer.log | \
  grep -oP "host = \K[0-9.]+" | sort | uniq -c
```

### 5.4 Целостность улик и chain-of-custody

- Записать всё на **WORM**-носитель (Write Once Read Many)
- SHA-256 всех файлов сразу после сбора
- Хранить минимум 90 дней после закрытия инцидента
- Для SOC2/HIPAA: 1 год
- Для 152-ФЗ: 3 года (журналы событий безопасности)

---

## 6. Compliance-автоматизация: CIS Benchmarks, OpenSCAP, автоматические проверки

**Ключевая мысль:** Ручной compliance-аудит Kafka раз в год не работает. К моменту аудита данные устаревают на недели, а инженеры тратят дни на сбор evidence. Непрерывная compliance-валидация через CIS Benchmarks + OpenSCAP + KafkaGuard + OPA/Conftest превращает compliance из квартального стресса в ежедневную рутину.

**Аналогия:** Годовой compliance-аудит — это как техосмотр автомобиля раз в год. Вы можете ездить с лысой резиной 364 дня в году, но в день проверки всё должно быть идеально. Непрерывная compliance-валидация — это датчики давления в шинах в реальном времени + автоматическая диагностика каждый раз при запуске двигателя.

### 6.1 Что такое CIS Apache Kafka Benchmark

CIS Apache Kafka Benchmark v1.0.0 — это 41 рекомендация по security hardening, распределённые по 6 разделам (BlackLabs, 2025):

| Раздел CIS | Кол-во рекомендаций | Темы |
|-----------|---------------------|------|
| 1. Network Configuration | 8 | Отключение PLAINTEXT, ограничение портов, TLS-only |
| 2. Authentication | 7 | SASL mechanisms, inter-broker auth, minimum SASL level |
| 3. Authorization | 9 | ACL enforcement, deny-by-default, super.users audit |
| 4. Encryption | 6 | TLS versions, cipher suites, endpoint identification |
| 5. Logging & Monitoring | 6 | Authorizer logging, audit trails, SIEM integration |
| 6. OS & Platform | 5 | File permissions, systemd hardening, non-root execution |

**Примеры CIS-рекомендаций для Kafka:**

- **CIS-KAFKA-1.1:** Убедиться, что PLAINTEXT listener отключён → `listeners` не содержит `PLAINTEXT://`
- **CIS-KAFKA-2.2:** Использовать SASL/SCRAM или mTLS, не SASL/PLAIN → `sasl.enabled.mechanisms` = SCRAM-SHA-256/512
- **CIS-KAFKA-3.1:** Включить ACL authorizer → `authorizer.class.name` not empty
- **CIS-KAFKA-4.1:** Минимальная версия TLS 1.2 → `ssl.enabled.protocols` не содержит TLSv1, TLSv1.1

### 6.2 Автоматизация CIS-проверок через OpenSCAP

OpenSCAP — open-source фреймворк для автоматизации compliance-проверок на основе SCAP (Security Content Automation Protocol). Напрямую для Kafka нет готового SCAP-контента (как для RHEL или Ubuntu), но вы можете создать custom OVAL-дефиниции или использовать OpenSCAP для проверки OS-level CIS требований к хостам Kafka.

**Проверка OS-level CIS для хостов Kafka через OpenSCAP:**

```bash
# Установка OpenSCAP
yum install -y openscap-scanner scap-security-guide  # RHEL/CentOS
# или
apt install -y libopenscap8 ssg-debian               # Debian/Ubuntu

# Сканирование хоста брокера на CIS Benchmark для ОС
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --results /tmp/kafka-broker-cis-results.xml \
  --report /tmp/kafka-broker-cis-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# Просмотр результатов
oscap xccdf generate report /tmp/kafka-broker-cis-results.xml > /tmp/cis-report.html
```

**Custom OpenSCAP-проверки для Kafka (через OVAL+script):**

```bash
# kafka_cis_check.sh — проверяет ключевые Kafka CIS требования
#!/bin/bash
FAILS=0
echo "=== Kafka CIS Check ==="

# CIS-KAFKA-1.1: PLAINTEXT listener
if grep -q "PLAINTEXT" /etc/kafka/server.properties; then
    echo "❌ CIS-KAFKA-1.1 FAIL: PLAINTEXT listener enabled"
    ((FAILS++))
else
    echo "✅ CIS-KAFKA-1.1 PASS"
fi

# CIS-KAFKA-2.2: SCRAM механизм
if grep -q "SCRAM-SHA" /etc/kafka/server.properties; then
    echo "✅ CIS-KAFKA-2.2 PASS: SCRAM enabled"
else
    echo "❌ CIS-KAFKA-2.2 FAIL: SCRAM not enabled"
    ((FAILS++))
fi

# CIS-KAFKA-3.1: Authorizer class
if grep -q "authorizer.class.name" /etc/kafka/server.properties; then
    echo "✅ CIS-KAFKA-3.1 PASS: Authorizer configured"
else
    echo "❌ CIS-KAFKA-3.1 FAIL: No authorizer"
    ((FAILS++))
fi

# CIS-KAFKA-4.1: Минимальный TLS 1.2
if grep "ssl.enabled.protocols" /etc/kafka/server.properties | grep -vq "TLSv1\b"; then
    echo "✅ CIS-KAFKA-4.1 PASS: TLS 1.0/1.1 disabled"
else
    echo "❌ CIS-KAFKA-4.1 FAIL: Old TLS versions enabled"
    ((FAILS++))
fi

echo "=== Result: $FAILS failed ==="
exit $FAILS
```

### 6.3 KafkaGuard — специализированный сканер безопасности Kafka

KafkaGuard — open-source инструмент для автоматизированного compliance-сканирования кластеров Kafka. В отличие от OpenSCAP, он проверяет runtime-состояние кластера (не только конфиг-файлы).

```bash
# Установка KafkaGuard
curl -LO https://github.com/KafkaGuard/kafkaguard-releases/releases/latest/download/kafkaguard-linux-amd64
chmod +x kafkaguard-linux-amd64
mv kafkaguard-linux-amd64 /usr/local/bin/kafkaguard

# Сканирование кластера
kafkaguard scan \
  --bootstrap-servers broker1:9093,kafka2:9093,kafka3:9093 \
  --sasl-mechanism SCRAM-SHA-512 \
  --sasl-username admin \
  --sasl-password AdminP@ss! \
  --profile cis \
  --output json > kafka-scan-results.json

# Проверка выводов (exit code = количество failed)
kafkaguard scan ... --fail-on medium  # упасть при medium+ severity
```

**Что проверяет KafkaGuard:**
- Открытые порты и используемые протоколы на каждом брокере
- Наличие аутентификации на каждом listener
- ACL — wildcard patterns, overly permissive grants
- TLS — версии, cipher suites, срок истечения сертификатов
- Метрики и логирование: включён ли authorizer.logger
- Соответствие CIS Kafka Benchmark v1.0.0

### 6.4 Checkov и KICS — проверка Kafka-конфигураций в IaC

**Checkov** (by Bridgecrew/Palo Alto) — сканер Infrastructure as Code, который проверяет Terraform, Helm, Kubernetes манифесты на security misconfigurations:

```bash
# Проверка Terraform/IaC для Kafka
checkov -d ./terraform/kafka/ \
  --check CKV2_K8S_1 \   # NetworkPolicy exists
  --check CKV_K8S_21 \    # default ServiceAccount not used
  --check CKV_K8S_28 \    # non-root container
  --check CKV_K8S_43 \    # read-only root filesystem
  --soft-fail
```

**KICS** (Keeping Infrastructure as Code Secure) — ещё один статический анализатор IaC для проверки security misconfigurations в Docker, Kubernetes, Terraform и Helm:

```bash
# Сканирование Docker-образов Kafka
kics scan -p ./docker/kafka/ -o ./results/ --ci

# Сканирование Helm-чартов Strimzi
kics scan -p ./helm/kafka/ -o ./results/
```

### 6.5 Conftest / OPA — policy-as-code валидация до деплоя

Проверка конфигураций Kafka ДО применения — в CI/CD пайплайне:

```bash
# Проверка server.properties через OPA Conftest
cat server.properties | conftest test \
  --policy kafka-security.rego -

# Проверка Strimzi Kafka CR
conftest test kafka-cluster.yaml \
  --policy kafka-policies/ \
  --all-namespaces
```

**Пример OPA-политики для server.properties:**

```rego
# kafka-security.rego
package kafka.security

deny[msg] {
    contains(input, "PLAINTEXT")
    msg := "PLAINTEXT listener detected — must be SASL_SSL"
}

deny[msg] {
    contains(input, "TLSv1")
    msg := "Old TLS version (1.0/1.1) detected"
}

deny[msg] {
    not contains(input, "authorizer.class.name")
    msg := "Authorization not configured"
}
```

### 6.6 Conduktor-подход: непрерывная compliance-валидация

Conduktor (commercial tool, но подход универсальный) предлагает модель **Continuous Compliance**:

Вместо "раз в год готовим evidence для аудитора" — ежедневные автоматические проверки:
- Все production-топики имеют RF ≥ 3 ✓
- Все топики с PII зашифрованы ✓
- Все service accounts прошли access review ≤ 90 дней ✓
- Все ACL-изменения имеют approval в audit trail ✓

```bash
# Концептуальный пример: daily compliance check
# (можно реализовать через bash + Kafka CLI)

#!/bin/bash
# daily-compliance-check.sh
echo "=== Daily Kafka Compliance Check ==="

# Проверка RF для production-топиков
LOW_RF_TOPICS=$(kafka-topics.sh --bootstrap-server broker1:9093 \
  --describe | grep "ReplicationFactor: [12]\b" | wc -l)
if [ "$LOW_RF_TOPICS" -gt 0 ]; then
    echo "❌ $LOW_RF_TOPICS topics with RF < 3 — violating policy"
else
    echo "✅ All topics have RF ≥ 3"
fi

# Проверка retention для PII-топиков
# (логика зависит от naming convention)

# Проверка, что authorizer работает
if kafka-acls.sh --bootstrap-server broker1:9093 --list &>/dev/null; then
    echo "✅ Authorizer active"
else
    echo "❌ Authorizer not responding"
fi

# Отправить результаты в SIEM / compliance dashboard
```

### 6.7 Автоматизированный compliance-пайплайн для Kafka

**Итоговая архитектура compliance automation:**

```
┌──────────┐     ┌───────────┐     ┌──────────────┐     ┌──────────┐
│   Git    │────▶│  CI/CD    │────▶│  K8s / Host  │────▶│  Kafka   │
│  (IaC)   │     │ Pipeline  │     │              │     │  Cluster │
└──────────┘     └───────────┘     └──────────────┘     └──────────┘
     │                │                                      │
     ▼                ▼                                      ▼
┌──────────────────────────────────────────────────────┐
│  Compliance Validation Layers:                       │
│  ① Conftest/OPA — проверка конфигов в CI (до deploy)│
│  ② Checkov/KICS — проверка IaC (до deploy)          │
│  ③ OS-level OpenSCAP — проверка хостов              │
│  ④ KafkaGuard — runtime-проверка кластера           │
│  ⑤ Daily compliance script — непрерывная валидация  │
└──────────────────────────────────────────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Compliance       │
                  │ Dashboard / SIEM │
                  │ (Evidence +      │
                  │  Alerts)         │
                  └─────────────────┘
```

---

## 7. Enterprise best practices для безопасности Kafka

**Ключевая мысль:** Best practices — это не "делайте TLS" (это base requirement). Enterprise best practices — это архитектурные решения, которые делают безопасность Kafka воспроизводимой, доказуемой и автоматизированной в масштабе организации.

### 7.1 Zero Trust архитектура для Kafka

**Принципы Zero Trust применительно к Kafka:**

1. **Never trust, always verify:** Каждый запрос к брокеру аутентифицируется, даже если он из того же namespace/VPC. Никаких implicit trust boundaries.

2. **Micro-segmentation:** Каждый компонент Kafka-экосистемы — в отдельном network segment:
   - Брокеры → broker subnet
   - Connect → connect subnet
   - Schema Registry → registry subnet
   - Клиентские приложения → app subnets (per team per environment!)

```yaml
# NetworkPolicy: Connect pods могут общаться ТОЛЬКО с брокерами
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: connect-to-brokers-only
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      app: kafka-connect
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              strimzi.io/component-type: kafka
      ports:
        - port: 9093
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

3. **mTLS everywhere:** Не просто TLS — mutual TLS между КАЖДОЙ парой компонентов. Брокер↔брокер, клиент↔брокер, Connect↔брокер, Schema Registry↔брокер.

4. **Per-component identities:** Каждый клиент, Connect task, Schema Registry instance имеет уникальный сертификат/SASL identity. Нельзя использовать shared credentials.

5. **Just-in-time access:** Для administrative операций — временное повышение привилегий через approval workflow, автоматически истекающее.

### 7.2 Defence in Depth для Kafka-экосистемы

| Слой | Механизм | Инструменты |
|------|----------|------------|
| **Периметр** | Firewall, WAF, DDoS protection | iptables/nftables, AWS Security Groups, Cloudflare |
| **Сеть** | Network segmentation, IDS/IPS | NetworkPolicy (K8s), VPC, Suricata/Snort |
| **Хост** | OS hardening, EDR, anti-malware | Systemd hardening, SELinux, auditd, CrowdStrike/SentinelOne |
| **Контейнер** | Non-root, read-only, seccomp, capabilities drop | Docker security opts, PodSecurityStandards |
| **Приложение (Kafka)** | SASL, TLS, ACL, audit logging | SCRAM-SHA-512, mTLS, StandardAuthorizer, authorizer.logger |
| **Данные** | Encryption at rest, key management, DLP | Disk encryption, Vault Transit, Vault PKI, tiered-storage encryption |
| **Операции** | Policy as Code, GitOps, SIEM, IR | OPA/Kyverno, ArgoCD (signed), Falco, Elasticsearch |

### 7.3 Segregation of duties и least privilege

**Разделение ролей в Kafka:**

| Роль | Обязанности | Типичный доступ |
|------|------------|-----------------|
| Platform Admin | Управление кластером, конфигурация брокеров, топики | Super users (полный) |
| Security Admin | Управление ACL, аудит, compliance | ACL Create/Delete/Alter |
| Application Developer | Разработка producer/consumer | Read/Write на конкретные топики |
| Data Owner | Владелец данных, approval доступа | Describe/Read на свои топики |
| SRE / Operations | Мониторинг, алертинг, DR | Read JMX метрики, Describe кластер |

**Least privilege enforcement через ACL для каждой роли — пример для Security Admin:**

```bash
# Security Admin: может менять ACL, но НЕ может читать/писать sensitive данные

# Разрешить управление ACL (Alter, Describe, Create, Delete для Cluster resource)
kafka-acls.sh --bootstrap-server broker1:9093 \
  --add --allow-principal User:security-admin \
  --operation Alter --operation Describe --operation Create --operation Delete \
  --cluster

# ❌ НЕ давать Read/Write на топики с данными
# (Security Admin не должен иметь доступа к содержимому сообщений)
```

### 7.4 Безопасность резервного копирования и DR

**Бэкап без шифрования — это резервная копия утечки.**

**Требования к безопасности бэкапов Kafka:**

| Требование | Реализация |
|-----------|-----------|
| Encryption at rest | AES-256-GCM, ключ в Vault Transit (не в том же хранилище) |
| Encryption in transit | TLS при передаче backup → S3/NFS |
| Key separation | Ключи шифрования бэкапов ≠ ключи production-кластера |
| Offline (air-gapped) копия | Минимум 1 копия на отключённом от сети носителе (защита от ransomware) |
| Immutable storage | WORM / object lock (AWS S3 Object Lock, Azure Immutable Blob) — защита от deletion |
| Integrity verification | SHA-256 после создания, периодическая проверка целостности |

**Шифрование Kafka-бэкапа перед отправкой в S3:**

```bash
#!/bin/bash
BACKUP_FILE="kafka-backup-$(date +%Y%m%d-%H%M%S).tar.gz"
BACKUP_DIR="/backup/kafka"

# Создать бэкап
tar -czf "$BACKUP_DIR/$BACKUP_FILE" /var/lib/kafka/data/

# Получить ключ из Vault
BACKUP_KEY=$(vault read -field=key transit/export/encryption-key/kafka-backup-key)

# Зашифровать (AES-256-GCM)
echo "$BACKUP_KEY" | openssl enc -aes-256-gcm -pass stdin \
  -in "$BACKUP_DIR/$BACKUP_FILE" \
  -out "$BACKUP_DIR/$BACKUP_FILE.enc"

# SHA-256 для целостности
sha256sum "$BACKUP_DIR/$BACKUP_FILE.enc" > "$BACKUP_DIR/$BACKUP_FILE.enc.sha256"

# Отправить зашифрованный в S3 (SSE-KMS дополнительно)
aws s3 cp "$BACKUP_DIR/$BACKUP_FILE.enc" "s3://kafka-backups/encrypted/" \
  --sse aws:kms --sse-kms-key-id alias/kafka-backup-key

# Безопасно удалить локальный незашифрованный бэкап
shred -u "$BACKUP_DIR/$BACKUP_FILE"
```

**Безопасность DR для Kafka:**

| Аспект | Требование |
|--------|-----------|
| Сетевая изоляция | DR-площадка в отдельном VPC/VLAN, трафик через IPSec tunnel |
| Аутентификация | Отдельные credentials для DR (≠ production!) |
| MirrorMaker | mTLS + ACL проверка на принимающей стороне |
| Failover | Документированный и ПРОТЕСТИРОВАННЫЙ runbook |
| Post-failover | Полный аудит ACL, consumer group authorisation |

### 7.5 Enterprise security governance для Kafka

**Владельцы безопасности:**
- **Kafka Platform Owner:** Head of Data Platform / Platform Engineering
- **Security Owner:** CISO / Head of Platform Security
- **Escalation:** Kafka Admin → Platform Security Lead → CISO → Board

**KPI безопасности Kafka:**

| KPI | Target | Измерение |
|-----|--------|-----------|
| MTTD (Mean Time To Detect) | < 5 мин | SIEM latency + IR response time |
| MTTR (Mean Time To Respond) | < 30 мин | От алерта до сдерживания |
| Audit finding closure rate | 100% за 30 дней | Compliance dashboard |
| Credential rotation compliance | 100% ротированы в срок | Vault audit log |
| Security patch latency | < 7 дней после CVE announcement | CI/CD pipeline |

**BIA (Business Impact Assessment) для Kafka:**

| Сценарий | RTO | RPO | Финансовый удар |
|----------|-----|-----|----------------|
| Kafka total outage | 2 часа | 0 | $120K/час (для e-commerce) |
| Data exfiltration из payments | 0 (немедленная реакция) | N/A | $500K+ (штрафы + репутация) |
| Компрометация админ-доступа | 1 час | 0 | $250K+ |

**Compliance-отчётность:** Ежеквартальный security posture report, включающий:
- Состояние hardening всех брокеров (automated scan results)
- Статус ротации сертификатов и credentials
- Инциденты безопасности (timeline, root cause, lessons learned)
- CIS Benchmark compliance score (% passed)

---

## 8. Три перспективы безопасности

### 8.1 Инфраструктурный безопасник

**Зона ответственности:** сеть, хосты, OS-level аудит, PKI-инфраструктура, сетевые IDS.

**Ключевые практики:**

- **auditd monitoring:** Правила отслеживают изменения server.properties, доступ к data/log/secrets директориям, запуск Kafka-бинарников с нестандартными параметрами.

- **OS-level firewall (iptables/nftables):** Whitelist-only политика. Клиентский трафик — только из approved подсетей. Межброкерный — только внутри кластерного VLAN.

- **Systemd hardening:** `PrivateTmp=true`, `ProtectSystem=strict`, `NoNewPrivileges=true`, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`, фильтрация system calls.

- **File access control:** server.properties → root:root 644. Secrets/keystores → kafka:kafka 600. Data/log dirs → kafka:kafka. Всё логируется auditd.

- **Network monitoring:** Регулярный nmap-скан брокеров (порты Kafka, ZK, JMX). Алерт на новые открытые порты и изменения протоколов.

- **SIEM-интеграция:** Filebeat → Elasticsearch. Алерты на: >10 SASL failures/мин, успешный админ-логин с нового IP, подозрительные сетевые соединения Kafka → external IP.

**Чеклист инфраструктурного безопасника:**
- [ ] PLAINTEXT listener (9092) ЗАКРЫТ на файрволе
- [ ] SASL_SSL listener (9093) — только approved подсети
- [ ] iptables политика DROP by default, whitelist
- [ ] auditd активен, правила kafka-specific загружены
- [ ] systemd hardening directives применены
- [ ] File permissions: 600 на secrets, 644 на server.properties
- [ ] Kafka процесс не от root
- [ ] SIEM получает authorizer.log + OS audit log

### 8.2 DevSecOps-инженер

**Зона ответственности:** CI/CD pipeline security, secrets management, container/K8s runtime security, supply chain, GitOps.

**Ключевые практики:**

- **Secrets management:** Vault (PKI для TLS-сертификатов, KV для SASL-паролей и keystore/truststore паролей). All credentials через Vault Agent или External Secrets Operator. Никаких plaintext-секретов в Git.

- **Container hardening:** Non-root (User 1000), read-only rootfs, cap-drop ALL (except NET_BIND_SERVICE), no-new-privileges, seccomp runtime default, resource limits (memory, CPU, pid limit).

- **Kubernetes security (Strimzi):** PodSecurityStandards restricted, NetworkPolicies deny-by-default + explicit allows (broker↔broker, clients↔broker, monitoring↔broker), OPA/Kyverno enforcement обязательного securityContext.

- **Pipeline security checks:**
  1. **Pre-commit:** Git hooks — no secrets in code (detect-secrets / gitleaks)
  2. **CI — config:** Conftest/OPA валидация server.properties и Strimzi CRs
  3. **CI — IaC:** Checkov/KICS — Terraform/Helm/Kubernetes misconfigs
  4. **CI — image:** Trivy/Grype scan образа Kafka
  5. **CI — sign:** GPG/SSH signed commit, cosign sign image

- **Runtime security:** Falco с custom Kafka-правилами (изменение server.properties, не-Kafka процесс читает data/, аномальное сетевое соединение на external IP).

- **GitOps security:** ArgoCD — signed commits only, RBAC least privilege (kafka-team: sync kafka/*, security-team: get kafka/*), External Secrets Operator для credentials.

**Чеклист DevSecOps:**
- [ ] Vault развёрнут, PKI + KV engines настроены для Kafka
- [ ] Vault Agent работает на всех брокерах, авто-ротация сертификатов
- [ ] Docker/K8s security context: non-root, read-only, seccomp, cap-drop
- [ ] OPA/Kyverno enforcement policy active в K8s
- [ ] CI пайплайн: Conftest → Checkov → Trivy → cosign
- [ ] Falco active с Kafka-rules
- [ ] ArgoCD: signed commits, RBAC
- [ ] Все secrets через External Secrets Operator / Vault CSI

### 8.3 Архитектор информационной безопасности

**Зона ответственности:** threat model, compliance, governance, стратегические решения, enterprise architecture, оценка рисков.

**Ключевые практики:**

- **Enterprise SSO for Kafka:** Federated authentication через OAuth 2.0 / OIDC. Единый IdP (Keycloak, Azure AD, Okta) — не Kafka-specific. RBAC на уровне организации с наследованием ролей.

- **Data classification в Kafka:**
  | Уровень | Топики | Controls |
  |---------|--------|----------|
  | Публичные | `*.logs`, `*.metrics` | Базовый ACL |
  | Внутренние | `orders`, `inventory` | ACL + audit log |
  | Конфиденциальные | `payments`, `phi-*` | ACL + audit + encryption + DLP |
  | КИИ / Гостайна | `kii-*` | ACL + air-gapped + СКЗИ |

- **Attribute-Based Access Control (ABAC):** Доступ на основе атрибутов principal (department, clearance level, data classification) — вместо ручного ACL-менеджмента. Integration с корпоративной IdM.

- **Compliance mapping, задокументированный для каждого framework:**
  - **152-ФЗ:** ПДн — классификация топиков, retention policies, audit trail (3 года хранения)
  - **GDPR:** Data inventory (какие топики содержат ПДн), consent tracking, right to erasure — automated deletion verification
  - **PCI-DSS:** Encryption at rest/transit (TLS 1.2+), quarterly access review, pen-test annual
  - **SOC 2:** Access control matrix, change management logs, incident response reports, availability metrics
  - **ISO 27001:** Asset inventory (Kafka-кластер как asset), risk assessment, incident response procedures

- **Data lifecycle policies:** Retention на основе классификации (6 мес — logs, 1 год — orders, 6 лет — HIPAA). Secure deletion — физическое затирание сегментов. Архивация — encrypted S3/tiered storage.

- **Pen-test scope (annual):** Unauthenticated Kafka access → ACL bypass → privilege escalation → data exfiltration → persistence → lateral movement to other Kafka-компоненты (Connect, Schema Registry).

- **Threat model (STRIDE) — updates:** После каждой major-версии Kafka (3.x → 4.x) пересмотр threat model — новые фичи = новые угрозы.

**Чеклист архитектора ИБ:**
- [ ] Data classification scheme задокументирована и применена к Kafka-топикам
- [ ] Enterprise SSO (OAuth/OIDC) интегрирован для administrative access
- [ ] Compliance mapping: 152-ФЗ, GDPR, PCI-DSS, SOC2, ISO 27001 — все документированы
- [ ] Data lifecycle policies определены (retention, archival, secure deletion)
- [ ] Pen-test scope определён, annual schedule установлен
- [ ] BIA: RTO/RPO/financial impact задокументированы
- [ ] Security governance: owners, escalation paths, quarterly reports

---

## 9. Итоговый чеклист административных практик

### Hardening
- [ ] Шаг 1: Сетевая изоляция — PLAINTEXT (9092) закрыт, whitelist для SASL_SSL (9093)
- [ ] Шаг 2: Аутентификация + шифрование — SASL_SSL listener, SCRAM-SHA-512, TLSv1.2+
- [ ] Шаг 3: ACL least privilege — authorizer enabled, deny-by-default, без wildcards
- [ ] Шаг 4: OS hardening — systemd directives, file permissions (600 secrets), auditd rules
- [ ] Шаг 5: Контейнеры — non-root, read-only, seccomp, capabilities drop (если Docker/K8s)
- [ ] Шаг 6: Валидация — automated hardening checks (скрипт), интеграция в CI/CD

### Secrets
- [ ] SASL/SCRAM пароли — ротируются каждые 90 дней (скрипт, zero downtime)
- [ ] OAUTHBEARER — через корпоративный IdP, keys через JWKS auto-refresh
- [ ] TLS-сертификаты — Vault PKI + Vault Agent auto-rotation (TTL ≤ 90 дней)
- [ ] Keystore/Truststore пароли — переменные окружения через Vault Agent, не в server.properties!
- [ ] Delegation Tokens — lifetime ≤ 24 часа, prefer OAuth
- [ ] Connect/Schema Registry API keys — External Secrets Operator

### Аудит и мониторинг
- [ ] authorizer.logger — DEBUG, отдельный appender, централизован (ELK/Splunk)
- [ ] SIEM алерты — auth failures, ACL DENIED, config changes, admin login from new IP
- [ ] JMX-метрики безопасности — Prometheus alerts для critical
- [ ] OS-level auditd — мониторинг Kafka sensitive files

### Incident Response
- [ ] IRT назначена, каналы связи готовы (Slack #kafka-incidents, PagerDuty)
- [ ] Tabletop-учения (TTX) минимум раз в квартал
- [ ] 4 playbook-сценария задокументированы: компрометация брокера, data exfiltration, ransomware, CI/CD compromise
- [ ] Forensic-скрипт сбора протестирован, chain-of-custody процедура задокументирована
- [ ] Baseline-норма трафика и ACL задокументирована

### Compliance automation
- [ ] CIS Kafka Benchmark v1.0.0 маппинг на controls
- [ ] OpenSCAP — OS-level CIS checks для всех Kafka-хостов (automated, weekly)
- [ ] KafkaGuard — automated scan кластера (daily or per deployment)
- [ ] Checkov/KICS — IaC security scanning в CI
- [ ] Conftest/OPA — policy-as-code валидация конфигов ДО деплоя
- [ ] Daily compliance checks — topics RF≥3, PII encrypted, access reviews

### Enterprise
- [ ] Zero Trust: mTLS everywhere, micro-segmentation (NetworkPolicies), unique per-component identities
- [ ] Segregation of duties: Platform Admin ≠ Security Admin ≠ Developer roles
- [ ] Бэкапы: encrypted AES-256-GCM, key separation, air-gapped копия, immutable storage
- [ ] DR: изолированная площадка, отдельные credentials, протестированный failover
- [ ] Security governance: owners defined, KPI tracked (MTTD/MTTR < targets), quarterly reports
- [ ] Compliance evidence: continuous generation, automated reporting, not manual audit prep

---

## 10. Источники

1. Apache Kafka Documentation — Security, [kafka.apache.org/documentation/#security](https://kafka.apache.org/documentation/#security)
2. CIS Apache Kafka Benchmark v1.0.0 — BlackLabs (2025), [cis.blacklabs.team/apache-kafka.html](https://cis.blacklabs.team/apache-kafka.html)
3. CodeStudy — "Kafka CIS Benchmark: A Comprehensive Guide" (2026), [codestudy.net/blog/kafka-cis-benchmark](https://www.codestudy.net/blog/kafka-cis-benchmark/)
4. Conduktor — "Kafka Security Best Practices: Enforcement Over Documentation" (2026), [conduktor.io/blog/kafka-security-best-practices](https://www.conduktor.io/blog/kafka-security-best-practices)
5. Conduktor — "Kafka Audit Automation: Continuous Compliance" (2026), [conduktor.io/blog/kafka-audit-automation](https://www.conduktor.io/blog/kafka-audit-automation)
6. Conduktor — "Audit Logging in Kafka: Who Did What and When" (2026), [conduktor.io/blog/kafka-audit-logging-compliance-forensics](https://www.conduktor.io/blog/kafka-audit-logging-compliance-forensics)
7. Confluent — "Building Real-Time Compliance and Audit Logging with Apache Kafka" (2025), [confluent.io/blog](https://www.confluent.io/blog/build-real-time-compliance-audit-logging-kafka/)
8. HashiCorp Developer — "Secure Kafka with Vault" (2026), [developer.hashicorp.com/validated-patterns/vault/vault-securing-kafka](https://developer.hashicorp.com/validated-patterns/vault/vault-securing-kafka)
9. Zendesk Engineering — "Kafka: Automating Root CA Rotation with Vault" (2025), [zendesk.engineering](https://zendesk.engineering/kafka-automating-root-ca-rotation-with-vault-9bbbe07c7c6e)
10. KLogic — "Kafka Security Best Practices 2026", [klogic.io/blog/kafka-security-best-practices](https://klogic.io/blog/kafka-security-best-practices/)
11. KLogic — "Kafka SSL Certificate Rotation — Zero-Downtime Guide" (2026), [klogic.io/guides/kafka-ssl-certificate-rotation](https://klogic.io/guides/kafka-ssl-certificate-rotation/)
12. Software Patterns Lexicon — "Security Incident Response for Apache Kafka" (2026), [softwarepatternslexicon.com/kafka/...](https://softwarepatternslexicon.com/kafka/security-data-governance-and-ethical-considerations/auditing-and-monitoring-security-events/security-incident-response/)
13. Software Patterns Lexicon — "Mastering Secrets Management with Vault for Apache Kafka" (2026), [softwarepatternslexicon.com/kafka/...](https://softwarepatternslexicon.com/kafka/security-data-governance-and-ethical-considerations/integration-with-external-security-tools/secrets-management-with-vault/)
14. Falco Security — Cloud Native Runtime Security, [falco.org](https://falco.org/)
15. ArgoCD Documentation — Security and RBAC, [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/release-1.8/operator-manual/security/)
16. Aquilax — "GitOps Security: ArgoCD and Flux Vulnerabilities" (2026), [aquilax.ai/blog/gitops-argocd-flux-security](https://aquilax.ai/blog/gitops-argocd-flux-security/)
17. KLogic — "Kafka Audit Logging & Compliance — GDPR, SOX, HIPAA & PCI DSS Guide" (2026), [klogic.io/guides/kafka-audit-logging-compliance](https://klogic.io/guides/kafka-audit-logging-compliance/)
18. Kai Waehner — "Apache Kafka in Cybersecurity for SIEM/SOAR Modernization" (2024), [kai-waehner.medium.com](https://kai-waehner.medium.com/apache-kafka-in-cybersecurity-for-siem-soar-modernization-ccefb44ddbe9)
19. Codestudy — "Kafka SSO: A Comprehensive Guide" (2026), [codestudy.net/blog/kafka-sso](https://www.codestudy.net/blog/kafka-sso/)
20. TuxCare — "2026 Apache Kafka Security: Best Practices & EOL Risks" (2025), [tuxcare.com/blog/apache-kafka-security](https://tuxcare.com/blog/apache-kafka-security/)
21. OneUptime — "How to Set Up Kafka Security (SASL/SSL)" (2026), [oneuptime.com](https://oneuptime.com/blog/post/2026-02-02-kafka-security-sasl-ssl/view)
22. OneUptime — "How to Run OpenSCAP Compliance Scans on Ubuntu" (2026), [oneuptime.com](https://oneuptime.com/blog/post/2026-01-15-run-openscap-compliance-scans-ubuntu/view)
23. AutoMQ — "Kafka Security: All You Need to Know & Best Practices" (2025), [automq.com/blog](https://www.automq.com/blog/kafka-security-all-you-need-to-know-and-best-practices)
24. ITNEXT — "Securing Kafka: Demystifying SASL, SSL, and Authentication Essentials" (2026), [itnext.io](https://itnext.io/securing-kafka-demystifying-sasl-ssl-and-authentication-essentials-01a9fb8092a3)

---

## 11. Связанные статьи

- [01-security-features.md](01-security-features.md) — Встроенные механизмы безопасности (TLS, SASL, ACL)
- [02-hardening.md](02-hardening.md) — Hardening: OS, Docker, Kubernetes, CIS Benchmark
- [03-attack-surface.md](03-attack-surface.md) — Поверхность атаки: CVE, Attack Tree, векторы атак
- [../07-operations/03-backup-recovery.md](../07-operations/03-backup-recovery.md) — Резервное копирование и аварийное восстановление
- [../07-operations/01-monitoring.md](../07-operations/01-monitoring.md) — Мониторинг Kafka (security-метрики)
- [../07-operations/05-automation.md](../07-operations/05-automation.md) — Автоматизация: IaC, GitOps
- [../07-operations/04-troubleshooting.md](../07-operations/04-troubleshooting.md) — Диагностика неполадок (Incident Response procedures)
