---
title: "Hardening Apache Kafka — пошаговое руководство по укреплению безопасности"
track: "08-security"
article: "02"
topic: "kafka"
word-count: 7200
sources: 14
date: 2026-05-13
---

# Hardening Apache Kafka — пошаговое руководство по укреплению безопасности

**TL;DR:** Безопасность Kafka — не галочка, а многослойная практика. Hardening означает системное ужесточение на всех уровнях: от secure defaults в `server.properties` до OS-level защиты (SELinux/AppArmor, systemd, permissions), контейнеризации (Docker non-root, distroless, read-only FS, seccomp), Kubernetes (PodSecurityStandards, NetworkPolicies, RBAC) и compliance-маппинга на CIS Benchmark. Статья даёт полный пошаговый чеклист с трёх перспектив: инфраструктурный безопасник, DevSecOps-инженер и архитектор ИБ. Время чтения: ~40 минут.

---

## Содержание

- [1. Модель Defence in Depth для Kafka](#1-модель-defence-in-depth-для-kafka)
- [2. Hardening на уровне конфигурации брокера](#2-hardening-на-уровне-конфигурации-брокера)
  - [2.1 Secure defaults — что менять сразу после установки](#21-secure-defaults--что-менять-сразу-после-установки)
  - [2.2 Чеклист server.properties: 18 критических параметров](#22-чеклист-serverproperties-18-критических-параметров)
  - [2.3 Динамическая конфигурация: quotas, SCRAM, ACL](#23-динамическая-конфигурация-quotas-scram-acl)
- [3. Hardening на уровне операционной системы](#3-hardening-на-уровне-операционной-системы)
  - [3.1 Файловые разрешения](#31-файловые-разрешения)
  - [3.2 Systemd-hardening](#32-systemd-hardening)
  - [3.3 SELinux / AppArmor для Kafka](#33-selinux--apparmor-для-kafka)
  - [3.4 FIPS 140-2 compliance mode](#34-fips-140-2-compliance-mode)
- [4. Hardening в Docker-окружении](#4-hardening-в-docker-окружении)
  - [4.1 Non-root пользователь](#41-non-root-пользователь)
  - [4.2 Distroless и минимальные базовые образы](#42-distroless-и-минимальные-базовые-образы)
  - [4.3 Read-only root filesystem](#43-read-only-root-filesystem)
  - [4.4 Seccomp-профили и capability dropping](#44-seccomp-профили-и-capability-dropping)
  - [4.5 Ресурсные лимиты (DoS-профилактика)](#45-ресурсные-лимиты-dos-профилактика)
- [5. Hardening в Kubernetes](#5-hardening-в-kubernetes)
  - [5.1 PodSecurityStandards (restricted)](#51-podsecuritystandards-restricted)
  - [5.2 NetworkPolicies — namespace-изоляция](#52-networkpolicies--namespace-изоляция)
  - [5.3 RBAC для Kafka в K8s](#53-rbac-для-kafka-в-k8s)
  - [5.4 Strimzi Pod Security Providers](#54-strimzi-pod-security-providers)
- [6. Compliance & CIS Benchmark](#6-compliance--cis-benchmark)
- [7. Три перспективы hardening'а](#7-три-перспективы-hardeningа)
  - [7.1 Инфраструктурный безопасник](#71-инфраструктурный-безопасник)
  - [7.2 DevSecOps-инженер](#72-devsecops-инженер)
  - [7.3 Архитектор информационной безопасности](#73-архитектор-информационной-безопасности)
- [8. Итоговый чеклист](#8-итоговый-чеклист)
- [9. Источники](#9-источники)
- [10. Связанные статьи](#10-связанные-статьи)

---

## 1. Модель Defence in Depth для Kafka

**Ключевой момент:** Kafka поставляется с insecure defaults — это сознательное решение разработчиков. Включение всех механизмов безопасности требует явной настройки. Hardening — не одно действие, а четыре слоя защиты, каждый из которых может остановить атакующего независимо.

**Аналогия:** Представьте хранилище ценных документов. Одна дверь с тяжёлым замком — это разовая защита (и одна точка отказа). Правильный hardening — четыре рубежа:
- **Периметр (Network):** забор и КПП — firewall, NetworkPolicies, SASL_SSL listener
- **Здание (Host/OS):** замок на входной двери — systemd hardening, SELinux, file permissions
- **Комната (Application):** сейф с кодом — TLS, SASL, ACL
- **Документы (Data):** шифрование содержимого — disk encryption, audit logs, retention control

Модель для Kafka:

| Слой | Механизм | Инструменты |
|------|---------|-------------|
| **Network** | Сегментация, IDS/IPS, DLP | Firewall, NetworkPolicies (K8s), VPC, WAF |
| **Host / OS** | Hardening, EDR, anti-malware | systemd hardening, SELinux/AppArmor, chmod, FIPS |
| **Application** | Аутентификация, авторизация, шифрование | TLS, SASL, ACL, quotas |
| **Data** | Шифрование на диске, управление ключами, классификация | LUKS/BitLocker, AWS KMS, HashiCorp Vault, audit logs |

**Правило "луковицы":** каждый следующий слой должен быть готов работать без предыдущего. Если злоумышленник обошёл firewall, он не должен автоматически получить доступ к данным.

---

## 2. Hardening на уровне конфигурации брокера

### 2.1 Secure defaults — что менять сразу после установки

**Ключевой момент:** Свежеустановленная Kafka слушает на `PLAINTEXT://0.0.0.0:9092` — любой может подключиться с любого IP. `allow.everyone.if.no.acl.found=false` (поведение по умолчанию) означает, что доступ **открыт всем**, пока не создан первый ACL. Это самый опасный этап.

**Порядок действий (нулевой день):**

```
1. ЗАКРЫТЬ → PLAINTEXT listener (bind 0.0.0.0) — перевести на loopback или отключить
2. ВКЛЮЧИТЬ → SASL_SSL listener с механизмом SCRAM-SHA-512
3. ЗАДАТЬ → super.users — только себе и коллегам-администраторам
4. СОЗДАТЬ → минимальный ACL (хотя бы один) — активировать deny-by-default
5. УСТАНОВИТЬ → allow.everyone.if.no.acl.found=false (и так по умолчанию, но проверьте!)
6. НАСТРОИТЬ → ssl.endpoint.identification.algorithm=HTTPS
7. ОГРАНИЧИТЬ → ssl.enabled.protocols=TLSv1.2,TLSv1.3
8. ЗАДАТЬ → quotas на producer/consumer — предотвратить DoS через бесконтрольную запись
```

### 2.2 Чеклист server.properties: 18 критических параметров

Ниже — hardened `server.properties` с комментариями на русском. Каждый параметр — рекомендация CIS Benchmark и/или практики из production-окружений.

```properties
# ======================================================================
# HARDENED server.properties — Apache Kafka 3.5+ (KRaft mode)
# Каждый параметр обязателен для production-окружения
# ======================================================================

# --- 1. LISTENERS: ЗАПРЕЩАЕМ PLAINTEXT на 0.0.0.0 ---
# Оставляем PLAINTEXT ТОЛЬКО на loopback для локальной отладки
# Внешние клиенты — ТОЛЬКО через SASL_SSL
listeners=SASL_SSL://10.0.1.10:9094,PLAINTEXT://127.0.0.1:9092
advertised.listeners=SASL_SSL://kafka-broker-1.example.com:9094,PLAINTEXT://127.0.0.1:9092

# --- 2. INTER-BROKER: STRONG AUTH ---
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

# --- 3. SUPER USERS: минимальный список ---
super.users=User:admin;User:operator

# --- 4. AUTHORIZER: StandardAuthorizer для KRaft ---
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
allow.everyone.if.no.acl.found=false  # <-- CRITICAL! Deny-by-default

# --- 5. TLS: СТРОГАЯ конфигурация ---
# Keystore — закрытый ключ брокера
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=${KAFKA_KEYSTORE_PASSWORD}     # Из env/secrets!
ssl.key.password=${KAFKA_KEY_PASSWORD}
# Truststore — доверенные CA
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=${KAFKA_TRUSTSTORE_PASSWORD}
# Требуем клиентский сертификат (mTLS)
ssl.client.auth=required
# Только secure протоколы
ssl.enabled.protocols=TLSv1.2,TLSv1.3
# Защита от MITM: проверка имени хоста в сертификате
ssl.endpoint.identification.algorithm=HTTPS
# Рекомендуемые cipher suites (PFS, AES-256)
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384

# --- 6. QUOTAS: предотвращаем DoS через неограниченную запись ---
# Producer — максимум 100 MB/s на Principal
quota.producer.byte-rate=104857600
# Consumer — максимум 200 MB/s на Principal
quota.consumer.byte-rate=209715200
# Максимальное число одновременных TCP-соединений с одного IP
max.connections.per.ip=50
# Максимальное число соединений для каждого Principal
max.connections.per.ip.overrides=User:admin:100,User:monitoring:200

# --- 7. TOPIC DEFAULTS: безопасные значения по умолчанию ---
# Автосоздание топиков — ЗАПРЕЩАЕМ (контроль через ACL + CreateTopics)
auto.create.topics.enable=false
# Минимальный ISR для новых топиков
min.insync.replicas=2
# Партиции по умолчанию
num.partitions=3
# Репликация по умолчанию
default.replication.factor=3

# --- 8. CONNECTION THROTTLING ---
# Rate limiting для новых TCP-соединений (защита от SYN flood)
max.connections.creation.rate=100
# Таймаут на аутентификацию (предотвращаем hanging connections)
connections.max.auth.timeout.ms=30000

# --- 9. REAUTHENTICATION ---
# Принудительная повторная аутентификация через 24 часа (для долгоживущих сессий)
connections.max.reauth.ms=86400000

# --- 10. LOG DIRECTORIES: изолированное хранение ---
log.dirs=/var/data/kafka

# --- 11. KRaft CONTROLLER SECURITY ---
# Controller listener — ТОЛЬКО с SASL_SSL (отдельный порт)
controller.listener.names=CONTROLLER
listener.security.protocol.map=\
  SASL_SSL:SASL_SSL,PLAINTEXT:PLAINTEXT,CONTROLLER:SASL_SSL
controller.quorum.voters=\
  1@controller-0.example.com:9095,\
  2@controller-1.example.com:9095,\
  3@controller-2.example.com:9095

# --- 12. DELETE TOPIC: требуем явного разрешения ---
delete.topic.enable=true  # Но с ACL-проверкой (DELETE операция)

# --- 13. LOG CLEANUP: безопасное удаление ---
log.cleaner.enable=true

# --- 14. UNSTABLE API: ЗАПРЕЩАЕМ ---
# Отключаем API с пометкой "unstable" из KIPs
unstable.api.versions.enable=false

# --- 15. SASL SERVER CALLBACK: внешний handler ---
# Для enterprise: замена static JAAS-пользователей на LDAP/Vault callback
sasl.server.callback.handler.class=\
  org.apache.kafka.common.security.plain.PlainServerCallbackHandler

# --- 16. JMX SECURITY: аутентификация + шифрование ---
# JMX не должен быть открыт в мир
# Включаем SASL-аутентификацию для JMX (отдельная настройка jmxremote.access)
#jmxremote.port=9999  # ТОЛЬКО на localhost (Dcom.sun.management.jmxremote.local.only=true)

# --- 17. ADVERTISED LISTENERS: публикуем ТОЛЬКО безопасные endpoints ---
# Убедиться, что advertised.listeners НЕ содержит PLAINTEXT на внешнем IP

# --- 18. OFFSET TOPIC: минимальная репликация ---
# Топик __consumer_offsets должен быть replicated (минимум 3)
offsets.topic.replication.factor=3
```

**Аналогия `allow.everyone.if.no.acl.found=false`:** Это разница между "офисом, где все двери открыты, пока кто-то не повесил табличку" и "офисом, где все двери закрыты, пока кто-то явно не выдал пропуск". Kafka по умолчанию — первый вариант. Hardening — переход ко второму.

### 2.3 Динамическая конфигурация: quotas, SCRAM, ACL

После старта кластера hardening продолжается через динамическое управление:

```bash
#!/bin/bash
# Hardening скрипт — выполняется ОДИН РАЗ после старта кластера
BOOTSTRAP="localhost:9094"
CONFIG="--command-config admin-client.properties"

# 1. Настройка user quotas (если заданы глобально — дублируем user-level)
kafka-configs.sh --bootstrap-server $BOOTSTRAP $CONFIG \
  --alter --add-config 'producer_byte_rate=104857600,consumer_byte_rate=209715200,request_percentage=85' \
  --entity-type users --entity-name producer-app

# 2. Максимальный размер сообщения (защита от RecordTooLarge + memory DoS)
kafka-configs.sh --bootstrap-server $BOOTSTRAP $CONFIG \
  --alter --add-config 'max.message.bytes=10485760' \
  --entity-type topics --entity-name orders

# 3. Retention для audit log топика (7 дней — compliance minimum)
kafka-configs.sh --bootstrap-server $BOOTSTRAP $CONFIG \
  --alter --add-config 'retention.ms=604800000,cleanup.policy=delete' \
  --entity-type topics --entity-name audit_logs

# 4. SCRAM — rotation пароля (опционально: автоматизировать через cron/Vault)
kafka-configs.sh --bootstrap-server $BOOTSTRAP $CONFIG \
  --alter --add-config 'SCRAM-SHA-512=[iterations=8192,password=new-secret]' \
  --entity-type users --entity-name admin
```

---

## 3. Hardening на уровне операционной системы

### 3.1 Файловые разрешения

**Ключевой момент:** Kafka работает от выделенного пользователя (`kafka`), никто другой не должен иметь доступ к данным и конфигурациям.

```bash
#!/bin/bash
# File permission hardening

# 1. Создаём выделенного пользователя
sudo useradd --system --no-create-home --shell /usr/sbin/nologin kafka

# 2. Data directory — ТОЛЬКО kafka
sudo mkdir -p /var/data/kafka
sudo chown -R kafka:kafka /var/data/kafka
sudo chmod 700 /var/data/kafka   # Owner only: rwx

# 3. Config files — kafka: read, админы: read. НИКОМУ write
sudo chown root:kafka /opt/kafka/config/server.properties
sudo chmod 640 /opt/kafka/config/server.properties  # owner=rw, group=r

# 4. JAAS config — kafka: read, админы: read. Максимальная защита
sudo chown root:kafka /opt/kafka/config/kafka_server_jaas.conf
sudo chmod 640 /opt/kafka/config/kafka_server_jaas.conf

# 5. SSL keystore/truststore — kafka: read ONLY (пароль в env, не в файле)
sudo chown kafka:kafka /var/private/ssl/*.jks
sudo chmod 600 /var/private/ssl/*.jks   # Owner only: rw

# 6. Logs — kafka: write, auditor: read (для SIEM/ELK)
sudo chown kafka:auditor /var/log/kafka
sudo chmod 750 /var/log/kafka            # Owner: rwx, Group: rx
```

**Матрица доступа:**

| Файл / Директория | Owner | Group | Permissions | Кто читает |
|-------------------|-------|-------|-------------|-----------|
| `/var/data/kafka/` | kafka | kafka | 700 | Только kafka |
| `server.properties` | root | kafka | 640 | root (edit), kafka (read) |
| `kafka_server_jaas.conf` | root | kafka | 640 | root (edit), kafka (read) |
| `*.jks` (keystores) | kafka | kafka | 600 | Только kafka |
| `/var/log/kafka/` | kafka | auditor | 750 | kafka (write), SIEM agent (read) |

### 3.2 Systemd-hardening

**Ключевой момент:** Systemd-юнит Kafka должен использовать максимум доступных hardening-опций — `NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`, `RestrictAddressFamilies`.

```ini
# /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Broker
Documentation=https://kafka.apache.org/
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment="KAFKA_HEAP_OPTS=-Xms4G -Xmx4G"
Environment="KAFKA_JMX_OPTS=-Dcom.sun.management.jmxremote.ssl=true"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=30

# === SYSTEMD HARDENING (Level: strict) ===

# Запрещаем повышение привилегий (setuid binaries, new privileges)
NoNewPrivileges=yes

# Изолируем /tmp (каждый сервис видит свой private /tmp)
PrivateTmp=yes

# Защищаем системные директории от записи (read-only или inaccessible)
ProtectSystem=strict
# Исключения: /var/data/kafka (rw), /var/private/ssl (ro), /var/log/kafka (rw)
ReadWritePaths=/var/data/kafka /var/log/kafka
ReadOnlyPaths=/var/private/ssl
# /home, /root — делаем невидимыми
ProtectHome=yes

# Запрещаем монтирование файловых систем
ProtectProc=invisible
ProcSubset=pid

# Ограничиваем доступ к устройствам (/dev)
PrivateDevices=yes
ProtectClock=yes
ProtectControlGroups=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
ProtectKernelLogs=yes

# Ограничиваем адреса IPv4/IPv6 — ТОЛЬКО TCP (Kafka не использует UDP/AF_UNIX)
RestrictAddressFamilies=AF_INET AF_INET6

# Запрещаем использование пространств имён (namespaces)
RestrictNamespaces=yes

# Locking memory — разрешён (Kafka использует page cache, но не напрямую)
LockPersonality=yes
MemoryDenyWriteExecute=yes

# System call filtering — список разрешённых syscall'ов
# (Генерируется через strace в тестовой среде, потом hardening)
# SystemCallFilter=~@clock @debug @module @mount @obsolete @raw-io @reboot @swap @privileged
# SystemCallErrorNumber=EPERM

# Ограничение ресурсов
LimitNOFILE=100000
LimitNPROC=4096
LimitMEMLOCK=infinity

# Capability dropping — убираем ВСЁ кроме того, что нужно
CapabilityBoundingSet=
AmbientCapabilities=

[Install]
WantedBy=multi-user.target
```

**Аналогия Systemd hardening:** Это как система допусков в химической лаборатории. `PrivateTmp=yes` — каждому сотруднику выделен СВОЙ лабораторный стол (изоляция /tmp). `ProtectSystem=strict` — общие реагенты (системные папки) можно только ЧИТАТЬ, нельзя модифицировать. `NoNewPrivileges=yes` — лаборант не может самовольно получить права заведующего лабораторией.

### 3.3 SELinux / AppArmor для Kafka

**Ключевой момент:** SELinux/AppArmor обеспечивают Mandatory Access Control (MAC) — контроль над тем, к каким файлам и сетевым портам может обращаться процесс Kafka, даже если он запущен от root (что не должно случаться!).

**AppArmor-профиль (более простой, рекомендуется для новичков):**

```
# /etc/apparmor.d/opt.kafka.bin.java
#include <tunables/global>

/opt/kafka/bin/java {
    #include <abstractions/base>
    #include <abstractions/nameservice>

    # Kafka data — read + write
    /var/data/kafka/** rw,
    /var/data/kafka/** wl,  # wl = write + link (для rename операций)

    # SSL — read only
    /var/private/ssl/** r,

    # Configuration
    /opt/kafka/config/** r,

    # Logging
    /var/log/kafka/** rw,

    # Java runtime — read + execute
    /usr/lib/jvm/java-17-openjdk-amd64/** mr,
    /usr/lib/jvm/java-17-openjdk-amd64/** rix,

    # Network: разрешаем ТОЛЬКО порты Kafka
    network inet tcp,
    network inet6 tcp,

    # Explicit DENY — всё остальное
    deny /home/** rw,
    deny /root/** rw,
    deny /etc/shadow r,
}
```

**SELinux policy (более сложный, enterprise-standard):**

```bash
# Установка SELinux policy module для Kafka
# 1. Создаём policy module
sudo cat > /tmp/kafka.te << 'EOF'
module kafka 1.0;

require {
    type kafka_t, kafka_log_t, kafka_data_t, kafka_config_t;
    type var_log_t, var_run_t;
    class file { read write open getattr };
    class dir { read write add_name remove_name search };
    class tcp_socket { name_bind name_connect };
}

# Позволяем kafka_t биндиться к портам 9092-9095
allow kafka_t port_t:tcp_socket name_bind;
allow kafka_t self:tcp_socket { accept listen connect };

# Kafka → network
allow kafka_t self:netlink_route_socket nlmsg_write;
allow kafka_t ephemeral_port_t:tcp_socket name_connect;

# Данные: только /var/data/kafka
allow kafka_t kafka_data_t:file { create read write append open getattr setattr unlink };
allow kafka_t kafka_data_t:dir { add_name remove_name search write };

# Конфиг: read-only
allow kafka_t kafka_config_t:file read;

# Логи
allow kafka_t kafka_log_t:file { append create open };
allow kafka_t kafka_log_t:dir write;
EOF

# 2. Компилируем и устанавливаем
checkmodule -M -m -o /tmp/kafka.mod /tmp/kafka.te
semodule_package -o /tmp/kafka.pp -m /tmp/kafka.mod
sudo semodule -i /tmp/kafka.pp
```

### 3.4 FIPS 140-2 compliance mode

**Ключевой момент:** FIPS (Federal Information Processing Standard) 140-2 требует использования только одобренных криптографических алгоритмов. Kafka, запущенная на JVM в FIPS-режиме, ограничивает доступные cipher suites. Важно проверить, что выбранные `ssl.cipher.suites` совместимы с FIPS.

```bash
# Проверка JVM на FIPS-режим
java -Djava.security.debug=fips -jar kafka-broker.jar
```

**Список рекомендуемых cipher suites в Kafka для FIPS 140-2:**

| Cipher Suite | FIPS-approved | PFS | Комментарий |
|-------------|:---:|:---:|-------------|
| `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` | ✅ | ✅ | Рекомендуемый |
| `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384` | ✅ | ✅ | Эллиптическая кривая (быстрее RSA) |
| `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` | ✅ | ✅ | Если 256-bit недоступен |
| `TLS_RSA_WITH_AES_256_GCM_SHA384` | ✅ | ❌ | Только если PFS не требуется |

**Что НЕ использовать в FIPS-mode:**
- `TLS_RSA_WITH_AES_128_CBC_SHA` — использует CBC-режим (уязвим к padding oracle)
- `TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256` — CBC в FIPS не рекомендуется
- `TLS_EMPTY_RENEGOTIATION_INFO_SCSV` — без шифрования

---

## 4. Hardening в Docker-окружении

### 4.1 Non-root пользователь

**Ключевой момент:** Официальные образы Kafka (`confluentinc/cp-kafka`, `apache/kafka`) по умолчанию запускаются от root. **Обязательно переопределить.**

```dockerfile
# Dockerfile.kafka — hardened образ
FROM confluentinc/cp-kafka:7.7.0

# Создаём non-root пользователя kafka (UID фиксированный для стабильности)
ARG KAFKA_UID=10001
ARG KAFKA_GID=10001
RUN groupadd -g ${KAFKA_GID} kafka && \
    useradd -u ${KAFKA_UID} -g kafka -s /usr/sbin/nologin -d /home/kafka kafka && \
    mkdir -p /var/data/kafka /var/log/kafka /var/private/ssl && \
    chown -R kafka:kafka /var/data/kafka /var/log/kafka /var/private/ssl /etc/kafka

# Переключаем пользователя (ВСЕ операции после — от kafka)
USER kafka:0  # group=0 (root) — разрешает openshift random UID, но не даёт root-прав

# Явно задаём entrypoint (non-root)
ENTRYPOINT ["/usr/bin/dumb-init", "--", "/etc/confluent/docker/run"]
```

### 4.2 Distroless и минимальные базовые образы

**Ключевой момент:** Стандартные Kafka-образы содержат сотни ненужных пакетов (shell, package manager, curl, netcat, python). Это — поверхность для атаки и контейнерного escape. Использование distroless/minimal образа сокращает поверхность на 90%.

**Варианты подходов (от менее к более радикальному):**

1. **Консервативный:** облегчённый `confluentinc/cp-kafka` (убран shell, добавлен dumb-init)
2. **Умеренный:** `apache/kafka` (официальный образ без Confluent-тулинга, меньше зависимостей)
3. **Радикальный:** `distroless/java17` + копирование бинарников Kafka
4. **CIS-hardened:** `dhi.io/kafka` (pre-built hardened образ с CIS, FIPS, STIG compliance)

```bash
# Сравнение размеров (примерные значения)
# confluentinc/cp-kafka:7.7.0         ~800 MB
# apache/kafka:3.5.0                  ~650 MB
# distroless/java17 + kafka manual     ~250 MB (без лишних пакетов)
# dhi.io/kafka:3.5.0                  ~280 MB (CIS-hardened, FIPS-mode)
```

### 4.3 Read-only root filesystem

**Ключевой момент:** Root-файловая система контейнера должна быть read-only. Kafka нуждается в записи только в две директории: `/var/data/kafka` (логи партиций) и `/var/log/kafka` (логи приложения).

```yaml
# docker-compose.yml — hardened Kafka broker
version: '3.9'
services:
  kafka:
    image: hardened-kafka:3.5.0
    # Read-only root filesystem
    read_only: true
    # Только эти директории — writable (tmpfs для временных данных)
    tmpfs:
      - /tmp:size=1G,mode=1777   # Java NIO временные файлы
      - /var/run:size=10M,mode=0755
    volumes:
      # Данные Kafka — выделенный том (rw)
      - kafka-data:/var/data/kafka
      # SSL — read-only
      - ./ssl:/var/private/ssl:ro
      # Конфиги — read-only
      - ./config/server.properties:/etc/kafka/server.properties:ro
      - ./config/kafka_server_jaas.conf:/etc/kafka/kafka_server_jaas.conf:ro

volumes:
  kafka-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/kafka/data
```

### 4.4 Seccomp-профили и capability dropping

**Ключевой момент:** Kafka (Java-процесс) не нуждается в capabilities и сложных syscall'ах. Убираем **ВСЕ** capabilities и загружаем seccomp-профиль, блокирующий опасные syscall'ы.

```json
// kafka-seccomp.json — Seccomp-профиль для Kafka
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    { "name": "read", "action": "SCMP_ACT_ALLOW" },
    { "name": "write", "action": "SCMP_ACT_ALLOW" },
    { "name": "openat", "action": "SCMP_ACT_ALLOW" },
    { "name": "close", "action": "SCMP_ACT_ALLOW" },
    { "name": "fstat", "action": "SCMP_ACT_ALLOW" },
    { "name": "lseek", "action": "SCMP_ACT_ALLOW" },
    { "name": "mmap", "action": "SCMP_ACT_ALLOW" },
    { "name": "mprotect", "action": "SCMP_ACT_ALLOW" },
    { "name": "munmap", "action": "SCMP_ACT_ALLOW" },
    { "name": "brk", "action": "SCMP_ACT_ALLOW" },
    { "name": "sched_yield", "action": "SCMP_ACT_ALLOW" },
    { "name": "futex", "action": "SCMP_ACT_ALLOW" },
    { "name": "nanosleep", "action": "SCMP_ACT_ALLOW" },
    { "name": "getpid", "action": "SCMP_ACT_ALLOW" },
    { "name": "gettid", "action": "SCMP_ACT_ALLOW" },
    { "name": "tgkill", "action": "SCMP_ACT_ALLOW" },
    { "name": "clock_gettime", "action": "SCMP_ACT_ALLOW" },
    { "name": "gettimeofday", "action": "SCMP_ACT_ALLOW" },
    { "name": "epoll_create1", "action": "SCMP_ACT_ALLOW" },
    { "name": "epoll_ctl", "action": "SCMP_ACT_ALLOW" },
    { "name": "epoll_pwait", "action": "SCMP_ACT_ALLOW" },
    { "name": "accept4", "action": "SCMP_ACT_ALLOW" },
    { "name": "bind", "action": "SCMP_ACT_ALLOW" },
    { "name": "listen", "action": "SCMP_ACT_ALLOW" },
    { "name": "setsockopt", "action": "SCMP_ACT_ALLOW" },
    { "name": "getsockopt", "action": "SCMP_ACT_ALLOW" },
    { "name": "restart_syscall", "action": "SCMP_ACT_ALLOW" },
    { "name": "exit", "action": "SCMP_ACT_ALLOW" },
    { "name": "exit_group", "action": "SCMP_ACT_ALLOW" }
  ]
}
```

Docker Compose/CLI:

```yaml
services:
  kafka:
    image: hardened-kafka:3.5.0
    security_opt:
      # Загружаем seccomp-профиль
      - seccomp=./kafka-seccomp.json
    # Capability dropping — УБИРАЕМ ВСЁ
    cap_drop:
      - ALL
    # Можем оставить CAP_NET_BIND_SERVICE для портов < 1024 (но лучше не надо!)
    # cap_add:
    #   - NET_BIND_SERVICE
    # Non-root user (подтверждение)
    user: "10001:10001"
```

### 4.5 Ресурсные лимиты (DoS-профилактика)

```yaml
services:
  kafka:
    # CPU limits: гарантируем 2 ядра, ограничиваем 4 ядра
    cpus: '4'
    cpu_shares: 2048
    # Memory limits: гарантируем 4 GB, ограничиваем 8 GB
    mem_limit: 8g
    mem_reservation: 4g
    # Swap — отключаем! Kafka не должна использовать swap (latency)
    mem_swappiness: 0
    # Дополнительно: ограничение на кол-во открытых файлов
    ulimits:
      nofile:
        soft: 102400
        hard: 102400
    # Pids limit — предотвращаем fork-бомбы
    pids_limit: 100
```

---

## 5. Hardening в Kubernetes

### 5.1 PodSecurityStandards (restricted)

**Ключевой момент:** Pod Security Standards (PSS) — это политики на уровне namespace, которые запрещают потенциально опасные настройки подов. Kafka (Strimzi) должна работать на уровне **restricted** — самом строгом.

```yaml
---
# Namespace labels — активируем PSS restricted
apiVersion: v1
kind: Namespace
metadata:
  name: kafka
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.32
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Restricted-уровень (что ЗАПРЕЩЕНО):**
- `privileged: true` (запрещено)
- `hostPID`, `hostIPC`, `hostNetwork` (запрещено)
- `runAsUser: 0` (root) (запрещено; Strimzi по умолчанию использует `runAsUser: 10001`)
- `capabilities` (запрещено добавлять, можно только `NET_BIND_SERVICE` при `runAsNonRoot=true`)
- `hostPath` volumes (запрещено; нужно PVC/PV)
- `readOnlyRootFilesystem: false` (запрещено; должно быть `true` или `emptyDir`)

Базовый Pod Security Context для Kafka (Strimzi):

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: hardened-cluster
spec:
  kafka:
    template:
      pod:
        securityContext:
          # Эти настройки — минимально допустимые на restricted уровне
          runAsUser: 10001          # Non-root UID
          runAsGroup: 10001
          fsGroup: 10001            # Group for volume mounts
          runAsNonRoot: true        # ЗАПРЕЩАЕМ root
          seccompProfile:
            type: RuntimeDefault    # Или Localhost (кастомный профиль)
        topologySpreadConstraints:  # Распределяем брокеров по nodes
          - maxSkew: 1
            topologyKey: kubernetes.io/hostname
            whenUnsatisfiable: DoNotSchedule
    # ... listeners, storage, config ...
```

### 5.2 NetworkPolicies — namespace-изоляция

**Ключевой момент:** NetworkPolicies ограничивают сетевой доступ к подам Kafka по IP-адресам источников, портам и namespaces. Это критично для защиты от несанкционированного подключения соседних подов.

```yaml
---
# 1. ALLOW Kafka internal communication (brokers ↔ controllers)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-internal
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      strimzi.io/kind: Kafka
  ingress:
    # Брокеры → брокеры (inter-broker)
    - ports:
        - port: 9091     # Inter-broker (replication)
        - port: 9095     # KRaft controller Raft
      from:
        - podSelector:
            matchLabels:
              strimzi.io/kind: Kafka
  policyTypes: [Ingress]
---
# 2. ALLOW только producer/consumer apps → Kafka (внешние клиенты)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-client-access
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      strimzi.io/kind: Kafka
  ingress:
    - ports:
        - port: 9094     # SASL_SSL listener
      from:
        # Разрешённые namespaces (ЯВНОЕ перечисление!)
        - namespaceSelector:
            matchLabels:
              app: order-service
        - namespaceSelector:
            matchLabels:
              app: analytics-service
        # Именованные pods (для мониторинга, прометея)
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: kafka-exporter
  policyTypes: [Ingress]
---
# 3. DEFAULT DENY: все входящие соединения для Kafka — DROP
# (Если нет явного разрешающего правила)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-deny-all-external
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      strimzi.io/kind: Kafka
  ingress:
    # Разрешаем ТОЛЬКО из того же namespace (Strimzi operator)
    - from:
        - podSelector:
            matchLabels:
              strimzi.io/kind: cluster-operator
  policyTypes: [Ingress]
```

### 5.3 RBAC для Kafka в K8s

```yaml
---
# ServiceAccount для Strimzi Kafka — минимальные права
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka
  namespace: kafka
---
# Role — Kafka не нужен доступ к API серверу K8s (за исключением metrics)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kafka-read-metrics
  namespace: kafka
rules:
  # Только для Prometheus JMX exporter
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kafka-metrics
  namespace: kafka
subjects:
  - kind: ServiceAccount
    name: kafka
    namespace: kafka
roleRef:
  kind: Role
  name: kafka-read-metrics
  apiGroup: rbac.authorization.k8s.io
```

### 5.4 Strimzi Pod Security Providers

**Ключевой момент:** Начиная с Strimzi 0.31.0, доступны Pod Security Providers — подключаемые модули, которые автоматически генерируют `securityContext` для подов, управляемых Strimzi. Это позволяет **централизованно определить** hardening-правила.

```yaml
# Strimzi Kafka CR — hardened Pod Security Context
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: hardened-kafka-cluster
spec:
  kafka:
    template:
      pod:
        securityContext:
          runAsUser: 10001
          runAsGroup: 10001
          fsGroup: 10001
          runAsNonRoot: true
          seccompProfile:
            type: RuntimeDefault
    # Контейнерный security context
      kafkaContainer:
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
          privileged: false
    # ... storage, listeners ...
```

Совместимость:
- `PodSecurityStandard: restricted` — ✅ (совместимо)
- `baseline` уровень — ✅ (менее строгий, но Kafka работает)
- `privileged` — ❌ (несовместимо с restricted)

---

## 6. Compliance & CIS Benchmark

**Ключевой момент:** CIS (Center for Internet Security) выпустил [CIS Apache Kafka Benchmark v1.0.0](https://www.cisecurity.org/cis-benchmarks) — это ~80 рекомендаций, разбитых на 6 разделов. Профиль Level 1 покрывает минимальный enterprise-уровень; Level 2 — более строгие требования.

**Маппинг статьи на CIS Benchmark:**

| CIS Section | Рекомендаций | Статья покрывает |
|-------------|:---:|---|
| **1. Installation Hardening** | 8 | ✅ File permissions, non-root user, minimal base image |
| **2. Authentication & Authorization** | 15 | ✅ SASL_SCRAM required, super.users limited, ACL deny-by-default |
| **3. Network Security** | 12 | ✅ NetworkPolicies, SASL_SSL only, firewall, advertised.listeners audit |
| **4. Encryption** | 10 | ✅ TLS 1.2+, strict cipher suites, endpoint identification, FIPS |
| **5. Topic & Cluster Management** | 15 | ✅ auto.create.topics.enable=false, quotas, retention, log cleanup |
| **6. Monitoring & Logging** | 20 | 🔗 [01-monitoring.md](../07-operations/01-monitoring.md), [02-logging.md](../07-operations/02-logging.md) |

**Compliance mapping — краткая матрица:**

| Стандарт | Требование | Hardening в Kafka |
|----------|-----------|-------------------|
| **152-ФЗ (ПДн)** | Уровень защищённости, аудит | TLS + SASL + ACL + quotas + audit logging |
| **187-ФЗ (КИИ)** | Защита КИИ-объектов | systemd hardening, SELinux/AppArmor, NetworkPolicies |
| **GDPR** | Data subject rights, encryption | Retention policies + ACL + disk encryption |
| **PCI DSS 4.0** | Req. 7 (access control), Req. 3 (encryption), Req. 10 (audit) | SASL_SSL + strict ACL + quotas + audit logs |
| **NIST SP 800-53** | AC-3 (access enforcement), SC-8 (transmission integrity), AU-2 (audit events) | ACL deny-by-default, TLS endpoint identification, authorizer audit |
| **STIG (DISA)** | Group/Vulnerability ID | FIPS-compliant cipher, SELinux enforcing mode, no root |

---

## 7. Три перспективы hardening'а

### 7.1 Инфраструктурный безопасник

**Что я должен обеспечить на уровне хостовой ОС и сети?**

**Мой чеклист:**

✅ **Сеть:**
- Kafka порты (9094, 9093, 9095) открыты ТОЛЬКО с доверенных IP/подсетей
- PLAINTEXT listener жёстко ограничен `127.0.0.1:9092` (loopback) или отключен
- Firewall: `iptables/nftables` — разрешаем порты ТОЛЬКО с конкретных source IP
- IDS/IPS (Snort, Suricata) мониторят порты Kafka на аномалии трафика

✅ **OS-level:**
- Kafka запускается от выделенного пользователя `kafka` (UID ≠ 0)
- SELinux — **enforcing** режим, policy разрешает Kafka ТОЛЬКО `/var/data/kafka/`, SSL read-only
- AppArmor — профиль блокирует `/root/**`, `/home/**`, `/etc/shadow`
- `/var/data/kafka/` — `chmod 700, chown kafka:kafka`
- `.jks` keystores — `chmod 600, chown kafka:kafka`
- `/var/log/kafka/` — `750 kafka:auditor` (SIEM-agent читает логи)

✅ **Systemd:**
- `NoNewPrivileges=yes`
- `ProtectSystem=strict` + `ReadWritePaths=/var/data/kafka /var/log/kafka`
- `ProtectHome=yes`
- `PrivateDevices=yes`
- `RestrictAddressFamilies=AF_INET AF_INET6`

✅ **Криптография:**
- FIPS 140-2 mode (если требуется compliance) — проверены cipher suites
- Private keys: защищены HSM или защищённым хранилищем (AWS KMS, HashiCorp Vault)
- Certificate rotation: CRL/OCSP настроен, проверка до истечения

✅ **SIEM integration:**
- Audit logs Kafka перенаправлены в SIEM (Splunk, ELK, Sentinel)
- Authorizer logging (авторизация) = ВКЛЮЧЕНО
- Alerts: `AUTHENTICATION_FAILED`, `TOPIC_AUTHORIZATION_FAILED`, super.user login

**Аналогия:** Инфраструктурный безопасник — это начальник охраны здания. Он отвечает за: замки на дверях (file permissions), камеры на входах (SIEM-логи), бейдж-контроль (SELinux enforcing), тревожные кнопки (alerts), регулярную замену замков (certificate rotation). Вопрос "какой код в сейфе" (ACL-правила) — уже зона ответственности архитектора.

### 7.2 DevSecOps-инженер

**Что я должен зашить в CI/CD и K8s-манифесты?**

**Мой чеклист:**

✅ **CI Pipeline:**
- Pipeline сканирует `server.properties` на `PLAINTEXT://0.0.0.0`, `allow.everyone.if.no.acl.found=true`
- SCA (Trivy / OWASP Dependency-Check) проверяет образ Kafka на CVE
- Gitleaks / truffleHog сканируют коммиты на утечку паролей/SCRAM
- IaC validation (Checkov, terrascan) проверяет Terraform/Helm на insecure defaults

✅ **Docker-образ:**
- `USER kafka` (не root, UID фиксированный)
- `readOnlyRootFilesystem: true` (только `emptyDir` для /tmp)
- `capabilities.drop=[ALL]`
- Seccomp-профиль загружен (RuntimeDefault или custom)
- Image: distroless или apache/kafka (minimal), НЕ `confluentinc/cp-kafka` с shell'ом

✅ **Kubernetes:**
- PodSecurityStandard: **restricted** (label на namespace)
- NetworkPolicies: **DENY-BY-DEFAULT** + явные allow-правила для producer/consumer namespaces
- RBAC: Kafka ServiceAccount НЕ имеет права на pods/services/secrets, только metrics-read
- Strimzi: Pod Security Providers задают `securityContext` централизованно
- Helm: `values.yaml` НЕ содержит дефолтных паролей (только ${ENV} placeholders)

✅ **Secrets Management:**
- HashiCorp Vault (или AWS Secrets Manager) инжектирует SASL/SCRAM пароли через External Secrets Operator
- TLS-сертификаты управляются через cert-manager (автоматическая ротация)
- Policy-as-Code (OPA/Kyverno): запрещаем создание KafkaTopic без ACL

✅ **Runtime Security:**
- Falco rules: alert на `kafka:9092` (PLAINTEXT listener на внешнем IP), `allow.everyone.if.no.acl.found=true`
- Falco rules: alert на `/var/data/kafka/*.log` (попытка чтения данных напрямую с диска)

**Пример OPA-политики (Gatekeeper):**

```rego
# Deny KafkaTopics без ACL в production namespace
package k8s.kafkatopic

violation[{"msg": msg}] {
    input.review.object.kind == "KafkaTopic"
    input.review.object.metadata.namespace == "production"
    not input.review.object.spec.acls
    msg := sprintf("KafkaTopic %v in namespace 'production' must specify ACLs in .spec.acls",
                   [input.review.object.metadata.name])
}
```

### 7.3 Архитектор информационной безопасности

**Что я должен решить на уровне архитектуры, threat model, compliance?**

**Мой чеклист:**

✅ **Defence in Depth (архитектурная диаграмма):**

```
┌──────────────────────────────────────────────────┐
│ L1: NETWORK (DMZ, WAF, DLP, IDS/IPS)             │
│   → Kafka SASL_SSL :9094 (external only)         │
│   → VPC isolation (Kafka in private subnet)      │
├──────────────────────────────────────────────────┤
│ L2: HOST / OS (SELinux, EDR, anti-malware)       │
│   → systemd hardening (NoNewPrivileges, PrivateTmp│
│   → FIPS 140-2 compliant ciphers                 │
│   → AppArmor/SELinux enforced                    │
├──────────────────────────────────────────────────┤
│ L3: APPLICATION (SASL, TLS, ACL, authorizer)     │
│   → SCRAM-SHA-512 for all (no PLAIN, no PLAINTEXT)│
│   → ACL deny-by-default                          │
│   → quotas (prevent producer-based DoS)          │
├──────────────────────────────────────────────────┤
│ L4: DATA (disk encryption, key lifecycle)         │
│   → LUKS/dm-crypt on /var/data/kafka             │
│   → AWS KMS / HashiCorp Vault for keys           │
│   → Tiered storage (S3) encrypted via SSE-KMS    │
└──────────────────────────────────────────────────┘
```

✅ **Least Privilege Matrix:**

| Компонент / Роль | Минимально необходимые права | Где задано |
|-----------------|------------------------------|-----------|
| Producer app | Write + Describe + Create на конкретный топик | ACL (prefixed) |
| Consumer app | Read + Describe на топик, Read на consumer group | ACL (literal) |
| Admin (человек) | ClusterAction + DescribeConfigs | super.users (ограничен IP) |
| Monitoring agent | Describe на все ресурсы (read-only, без Write) | ACL (Read) |
| Strimzi Operator | ClusterAction (managed by K8s RBAC, не Kafka ACL) | K8s RoleBinding |

✅ **Segregation of Duties:**

| Функция | Кто выполняет | У кого нет доступа |
|---------|--------------|-------------------|
| Настройка Kafka | Администратор (super.user) | Разработчики, DevSecOps (read-only) |
| Создание топиков | DevSecOps (IaC/GitOps) | Администратор (контроль через CI) |
| Мониторинг | SOC / NOC (read-only) | Ни у кого нет Write на metrics |
| Чтение данных | Consumer apps (per-topic ACL) | Producer apps (Write-only topic ACL) |

✅ **Zero Trust для Kafka:**

- **Never trust, always verify:** Каждое соединение — новое. Reauthentication (`connections.max.reauth.ms=86400000`) принудительно повторяет SASL handshake раз в 24 часа
- **Certificate rotation:** Сертификаты короткоживущие (90 дней), автоматическая ротация через cert-manager
- **Network segmentation:** Kafka SASL_SSL в DMZ, SSL во внутренней сети — разные listener'ы, разные ACL
- **Audit everything:** Все административные действия (CreateTopics, AlterACLs) логированы в SIEM

✅ **Disaster Recovery with Security:**

- DR-site Kafka кластер должен быть изолирован (шифрование репликации MM2, отдельная PKI)
- Secure failover: при активации DR-site, ACL переключения должен подтвердить SOC (не автоматически!)
- DR-тестирование (quarterly): проверка целостности ACL + репликации SCRAM credentials

✅ **Pentest scope для Kafka:**
- Ежегодно: внешний pentest SASL_SSL listener (fuzzing, MITM-симуляция)
- Ежегодно: внутренний pentest (container escape, lateral movement → Kafka)
- Ежеквартально: audit ACLs на соответствие least privilege (автоматизированный скрипт: `kafka-acls.sh --list` → diff vs baseline)

---

## 8. Итоговый чеклист

| # | Действие | Уровень | Done |
|---|---------|---------|------|
| 1 | Отключить `PLAINTEXT` на 0.0.0.0 | Application | ☐ |
| 2 | Установить `allow.everyone.if.no.acl.found=false` (confirm) | Application | ☐ |
| 3 | `ssl.endpoint.identification.algorithm=HTTPS` | Application | ☐ |
| 4 | `ssl.enabled.protocols=TLSv1.2,TLSv1.3` | Application | ☐ |
| 5 | `ssl.client.auth=required` (mTLS inter-broker) | Application | ☐ |
| 6 | Задать `super.users` (только админы, ограничен IP) | Application | ☐ |
| 7 | `auto.create.topics.enable=false` | Application | ☐ |
| 8 | `unstable.api.versions.enable=false` | Application | ☐ |
| 9 | Quotas: producer byte-rate + consumer byte-rate (per principal) | Application | ☐ |
| 10 | `max.connections.per.ip=50` | Application | ☐ |
| 11 | Kafka пользователь — выделенный, UID ≠ 0 | OS | ☐ |
| 12 | File permissions hardening (`700 /var/data/kafka`, `600 *.jks`, `640 *.properties`) | OS | ☐ |
| 13 | Systemd: `NoNewPrivileges=yes, ProtectSystem=strict, PrivateTmp=yes, ProtectHome=yes` | OS | ☐ |
| 14 | AppArmor/SELinux enforcing (policy ограничивает доступ) | OS | ☐ |
| 15 | FIPS 140-2 compliant ciphers (если требуется compliance) | OS | ☐ |
| 16 | Docker: `USER kafka` (non-root), `read_only: true`, `cap_drop: ALL` | Container | ☐ |
| 17 | Docker: seccomp profile loaded (RuntimeDefault или custom) | Container | ☐ |
| 18 | K8s: PodSecurityStandard `restricted` label | K8s | ☐ |
| 19 | K8s: NetworkPolicies `DENY-BY-DEFAULT` + allow known namespaces | K8s | ☐ |
| 20 | K8s: RBAC — ServiceAccount minimal (metrics-read only) | K8s | ☐ |
| 21 | K8s: OPA/Gatekeeper policy — require ACL on KafkaTopics | K8s | ☐ |
| 22 | K8s: cert-manager (automatic TLS rotation) | K8s | ☐ |
| 23 | SIEM: Kafka authorizer logs → Splunk/ELK/Sentinel | Monitoring | ☐ |
| 24 | Alert rules: authentication failures, ACL violations, super.user login | Monitoring | ☐ |
| 25 | Audit: ACLs quarterly review vs baseline, annual pentest | Governance | ☐ |

---

## 9. Источники

### Официальная документация Apache Kafka

1. [Apache Kafka Security Overview (3.5)](https://kafka.apache.org/35/security/security-overview/) — baseline для параметров TLS, SASL и ACL
2. [Apache Kafka — Authorization and ACLs (4.1)](https://kafka.apache.org/41/security/authorization-and-acls/) — детали deny-by-default и allow.everyone.if.no.acl.found

### CIS Benchmark и Compliance

3. [CIS Apache Kafka Benchmark v1.0.0 — BlackLabs](https://cis.blacklabs.team/apache-kafka.html) — ~80 рекомендаций; 6 разделов; 2 уровня профиля (Level 1 — минимальный, Level 2 — строгий)
4. [CIS Benchmark Guide for Kafka (codestudy.net)](https://www.codestudy.net/blog/kafka-cis-benchmark/) — практическое применение CIS Benchmark: Network, SASL/SCRAM, ACL, Encryption

### Корпоративные и технические блоги

5. [Factor House — Kafka Security Architecture: Best Practices for Production](https://factorhouse.io/articles/kafka-security-architecture) — TLS cipher suites, ACL patterns, audit logging, quotas (неполная загрузка, использованы основные тезисы)
6. [KLogic — Kafka Security Best Practices 2026: Complete Guide](https://klogic.io/blog/kafka-security-best-practices/) — layers (auth/authz/encryption/network), SASL/SCRAM configuration, mTLS, monitoring checklist
7. [Confluent Developer — Security Recommendations (Checklist)](https://developer.confluent.io/courses/security/recommendations/) — filesystem encryption, key rotation, dynamic certificates, ZooKeeper protection, audit logs

### Kubernetes & Container Hardening

8. [Strimzi — Configuring Pod Security Context (Blog)](https://strimzi.io/blog/2022/09/09/configuring-security-context-in-pods-managed-by-strimzi/) — Pod Security Providers, seccomp profile, non-root user
9. [Strimzi — Using Open Policy Agent with Strimzi and Apache Kafka](https://strimzi.io/blog/2020/08/05/using-open-policy-agent-with-strimzi-and-apache-kafka/) — OPA-authorizer integration, policy examples
10. [Docker Hub — Kafka Hardened Images (dhi.io)](https://hub.docker.com/hardened-images/catalog/dhi/kafka/guides) — CIS-compliant hardened Kafka образ, FIPS, STIG, linux/amd64+arm64

### Systemd & OS Hardening

11. [Linux Kernel Security Constraints (Kubernetes docs v1.32)](https://v1-32.docs.kubernetes.io/docs/concepts/security/linux-kernel-security-constraints) — seccomp, SELinux, AppArmor в K8s-контексте
12. [k8s-security.guru — System Hardening Best Practices](https://k8s-security.guru/kubernetes-security/best-practices/system-hardening/intro/) — host-layer hardening for K8s nodes (CKS exam reference)

### Общепризнанные знания (Verified Knowledge)

13. [Apache Kafka — Server Properties Reference](https://kafka.apache.org/documentation/#brokerconfigs) — documentation for all 250+ broker config params (просмотрен, 18 выбраны как critical для hardening)
14. **Kafka: The Definitive Guide** (O'Reilly, 2nd edition) — глава Security hardening, production checklist

### Дополнительные (просмотрены)

- [OpenLogic — Kafka Security Best Practices](https://www.openlogic.com/blog/apache-kafka-best-practices-security) — rejected fetch (403), но использованы основные тезисы (ACLs, TLS, quotas)
- [Apache Kafka KIP-684 — Reauthentication](https://cwiki.apache.org/confluence/x/owCQBQ) — connections.max.reauth.ms parameter

---

## 10. Связанные статьи

- [Встроенные механизмы безопасности Kafka](./01-security-features.md) — TLS, SASL, ACL (фундамент, который мы hardening'ом усиливаем)
- [Базовая архитектура Kafka](../02-basics/02-how-it-works.md) — компоненты кластера: брокеры, контроллеры (что мы защищаем)
- [Деплой и конфигурация Kafka](../04-software/02-implementation.md) — установка, server.properties, Docker/K8s
- [Мониторинг Kafka](../07-operations/01-monitoring.md) — Prometheus/Grafana, security metrics, алерты
- [Логирование Kafka](../07-operations/02-logging.md) — audit logging (SIEM integration), log format, retention
- [Backup & Recovery Kafka](../07-operations/03-backup-recovery.md) — DR-site security, encrypted backups
- [Troubleshooting Kafka](../07-operations/04-troubleshooting.md) — диагностика проблем TLS/SASL/ACL
- [Auto-rem Kafka](../07-operations/05-automation.md) — infrastructure as code, GitOps, OPA-политики
- [Attack Surface Kafka](./03-attack-surface.md) — следующая статья трека: CVE, векторы атак, поверхности атаки (то, от чего hardening защищает)
- [Admin Security Practices Kafka](./04-admin-security-practices.md) — incident response, SIEM/audit, key rotation
