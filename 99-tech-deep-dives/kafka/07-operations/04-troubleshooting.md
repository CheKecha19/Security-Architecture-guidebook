---
title: "Диагностика и устранение неисправностей Apache Kafka"
track: "07-operations"
article: "04"
topic: "kafka"
word-count: 8000
sources: 13
date: 2026-05-13
---

# Диагностика и устранение неисправностей Apache Kafka

**TL;DR:** Комплексное руководство по диагностике и устранению типичных проблем Apache Kafka в production-окружениях. Рассматриваются systematic approach к поиску причин, конкретные symptom→cause→solution-матрицы для каждой категории отказов, диагностические команды и инструменты, процедуры аварийного восстановления. Отдельный блок — устранение неисправностей безопасности (SASL, TLS, ACL) с тремя перспективами: инфраструктурный безопасник, DevSecOps, архитектор ИБ. Материал ориентирован на практическое применение — каждая проблема сопровождается командой для диагностики и конкретным планом действий. Расчётное время чтения: 35 минут.

---

## Содержание

- [1. Системный подход к диагностике](#1-системный-подход-к-диагностике)
- [2. Проблемы брокеров: диагностика и решение](#2-проблемы-брокеров-диагностика-и-решение)
  - [2.1 Брокер не запускается](#21-брокер-не-запускается)
  - [2.2 Брокер упал/перестал отвечать](#22-брокер-упалперестал-отвечать)
  - [2.3 Диск заполнен (No space left on device)](#23-диск-заполнен-no-space-left-on-device)
  - [2.4 Высокая загрузка CPU/памяти](#24-высокая-загрузка-cpuпамяти)
- [3. Проблемы репликации](#3-проблемы-репликации)
  - [3.1 Under-replicated partitions](#31-under-replicated-partitions)
  - [3.2 ISR churn (частые изменения in-sync replicas)](#32-isr-churn-частые-изменения-in-sync-replicas)
- [4. Проблемы consumer groups](#4-проблемы-consumer-groups)
  - [4.1 Consumer lag](#41-consumer-lag)
  - [4.2 Частые ребалансировки](#42-частые-ребалансировки)
  - [4.3 Consumers не получают сообщения](#43-consumers-не-получают-сообщения)
  - [4.4 Дубликаты сообщений](#44-дубликаты-сообщений)
- [5. Проблемы продюсеров](#5-проблемы-продюсеров)
- [6. Проблемы Kafka Connect](#6-проблемы-kafka-connect)
- [7. Проблемы KRaft (Kafka 4.x)](#7-проблемы-kraft-kafka-4x)
- [8. Диагностический инструментарий](#8-диагностический-инструментарий)
- [9. Процедуры аварийного реагирования](#9-процедуры-аварийного-реагирования)
- [10. Устранение неисправностей безопасности](#10-устранение-неисправностей-безопасности)
- [Итоги](#итоги)
- [Источники](#источники)
- [Связанные статьи](#связанные-статьи)

---

## 1. Системный подход к диагностике

**Ключевая мысль:** Хаотичный поиск причины в распределённой системе с десятками компонентов — прямой путь к многочасовому даунтайму. Диагностика должна следовать чёткому алгоритму: от общего к частному, от быстрых проверок к глубокому анализу.

**Аналогия:** Работа с Kafka-инцидентом похожа на приём у врача в отделении неотложной помощи. Сначала измеряют жизненные показатели (пульс, давление), потом задают вопросы о симптомах, затем назначают анализы. Никто не начинает с МРТ всего тела.

### Алгоритм диагностики из 5 шагов

```
Шаг 1: Определить масштаб проблемы
├─ Затронуты все брокеры или один?
├─ Продюсеры, консьюмеры или те и другие?
├─ Все топики или конкретные?
└─ Проблема возникла внезапно или нарастала?

Шаг 2: Проверить базовые показатели (5 минут)
├─ Все ли брокеры в кластере?
│  kafka-broker-api-versions.sh --bootstrap-server broker:9092
├─ Есть ли under-replicated partitions?
│  kafka-topics.sh --describe --under-replicated-partitions
├─ Состояние контроллера
│  Проверить лог controller.log
└─ Дисковое пространство на всех брокерах
   df -h /var/kafka/data

Шаг 3: Проанализировать логи (10 минут)
├─ server.log — последние ERROR/WARN
├─ state-change.log — переходы лидерства партиций
├─ controller.log — операции контроллера
└─ kafka-authorizer.log — ошибки авторизации (если есть)

Шаг 4: Собрать метрики (JMX/Prometheus)
├─ UnderReplicatedPartitions (на каждом брокере)
├─ ActiveControllerCount (должно быть = 1)
├─ BytesInPerSec / BytesOutPerSec
├─ consumer_lag по группам
└─ GC-метрики JVM

Шаг 5: Локализовать корневую причину
├─ Исключить системные проблемы (CPU, память, диск, сеть)
├─ Исключить проблемы конфигурации
├─ Исключить проблемы внешних зависимостей
└─ Перейти к матрице symptom→cause→solution
```

**Важное правило:** Никогда не начинайте с перезапуска брокера без снятия диагностической информации. После перезапуска состояние JVM, heap dump, thread dump будут потеряны. Сначала соберите улики.

---

## 2. Проблемы брокеров: диагностика и решение

### 2.1 Брокер не запускается

**Симптомы:**
- Процесс Kafka завершается сразу после старта
- В логах: `ERROR Exiting Kafka` или `Fatal error`
- `kafka-server-start.sh` завершается с ненулевым кодом возврата

**Матрица диагностики:**

| Причина | Диагностика | Решение |
|---------|-------------|---------|
| Занят порт | `netstat -tulnp \| grep 9092` | Освободить порт или изменить `listeners` в `server.properties` |
| Конфликт `broker.id` | Поиск `Duplicate broker.id` в логах | Назначить уникальный `broker.id` |
| Повреждённые индексные файлы | `ERROR Found a corrupted index file` в логах | Удалить повреждённые `.index`/`.timeindex` файлы — Kafka пересоздаст их автоматически |
| Ошибка формата `meta.properties` | Файл `meta.properties` в `log.dirs` повреждён | Проверить содержимое, восстановить broker.id и cluster.id |
| Неправильные права на `log.dirs` | `ls -la /var/kafka/data` | `chown -R kafka:kafka /var/kafka/data` |
| Недостаточно памяти для JVM heap | `OutOfMemoryError: Java heap space` при старте | Увеличить `KAFKA_HEAP_OPTS="-Xmx2G -Xms2G"` или уменьшить `-Xmx` |
| Пустые snapshot-файлы (KRaft) | `find /var/kafka/data -name "*.snapshot" -size 0` | Удалить пустые snapshot-файлы |

**Диагностические команды:**

```bash
# Проверка занятости порта
netstat -tulnp | grep :9092

# Проверка логов при запуске
tail -f /var/log/kafka/server.log

# Проверка повреждённых индексных файлов
find /var/kafka/data -name "*.index" -size 0 -delete
find /var/kafka/data -name "*.timeindex" -size 0 -delete

# Проверка meta.properties
cat /var/kafka/data/meta.properties
# Ожидаемое содержимое:
# version=0
# broker.id=1
# cluster.id=abc123def456

# Проверка прав на директории данных
ls -laR /var/kafka/data | head -20
```

### 2.2 Брокер упал/перестал отвечать

**Симптомы:**
- Брокер числится в кластере, но не отвечает на запросы
- `kafka-broker-api-versions.sh` показывает timeout для этого брокера
- Продюсеры получают `TimeoutException` или `NetworkException`
- В логах — внезапное прекращение записей

**Матрица диагностики:**

| Причина | Диагностика | Решение |
|---------|-------------|---------|
| OutOfMemoryError | `grep "OutOfMemoryError" server.log` | Увеличить heap (`-Xmx`), проанализировать heap dump через Eclipse MAT |
| Слишком много открытых файлов | `lsof -p PID \| wc -l`; сравнить с `ulimit -n` | Увеличить `ulimit -n 100000` и `fs.file-max` |
| OOM Killer (Linux) | `dmesg \| grep -i "killed process"` | Уменьшить heap, добавить памяти серверу |
| Долгие GC-паузы (>10 сек) | `jstat -gc PID 1000` — мониторинг GC | Переключиться на G1GC, настроить `-XX:MaxGCPauseMillis=200` |
| Аппаратный сбой (диск/память) | `dmesg` на предмет I/O errors; SMART-статус диска | Замена оборудования |
| Исчерпание дискового пространства | См. раздел 2.3 | См. раздел 2.3 |

**Диагностические команды:**

```bash
# Проверка процесса
ps aux | grep kafka | grep -v grep

# Проверка открытых файловых дескрипторов
lsof -p $(pgrep -f kafka) | wc -l
# Типично для production: 5000-50000

# Проверка лимитов
ulimit -n
# Должно быть минимум 100000

# GC-мониторинг в реальном времени
jstat -gc $(pgrep -f kafka) 1000

# Проверка аппаратных проблем
dmesg | grep -i -E "error|fail|corrupt" | tail -20

# Проверка SMART для диска
smartctl -a /dev/sda | grep -i -E "error|reallocated|pending"
```

**Аналогия:** Представьте кассира в супермаркете, которому дали слишком маленький стол (heap). Пока покупателей мало — справляется. Но в час пик (production load) начинает ронять товары (сообщения) и задерживать очередь. Решение — либо дать стол побольше, либо уменьшить количество товаров, которые он обрабатывает одновременно.

### 2.3 Диск заполнен (No space left on device)

**Симптомы:**
- Брокер аварийно завершается с `Exit.halt(1)`
- В логах: `java.io.IOException: No space left on device`
- `ERROR Failed to append to log segment`
- Продюсеры получают `RecordTooLargeException` или `KafkaStorageException`

**Немедленные действия (Runbook):**

```bash
# Шаг 1: Оценить масштаб
df -h /var/kafka/data
# /dev/sda1  500G  500G  0  100% /var/kafka

# Шаг 2: Определить, кто съел место
du -sh /var/kafka/data/* | sort -rh | head -10
# 180G  /var/kafka/data/high-volume-topic-0
# 120G  /var/kafka/data/high-volume-topic-1
#  50G  /var/kafka/data/another-topic-0

# Шаг 3: Проверить, может ли кластер жить без этого брокера
kafka-topics.sh --bootstrap-server other-broker:9092 \
  --describe --under-replicated-partitions
# Если пусто и replication factor > 1 — кластер жив
```

**Стратегии восстановления (от безопасной к рискованной):**

#### Стратегия A: Динамическое уменьшение retention (безопасно)

```bash
# Уменьшаем retention до 1 часа для проблемного топика
kafka-configs.sh --bootstrap-server healthy-broker:9092 \
  --alter --entity-type topics --entity-name high-volume-topic \
  --add-config retention.ms=3600000,retention.bytes=10737418240

# Ждать 5 минут — log cleaner отработает
# Вернуть настройки после освобождения места:
kafka-configs.sh --bootstrap-server healthy-broker:9092 \
  --alter --entity-type topics --entity-name high-volume-topic \
  --delete-config retention.ms,retention.bytes
```

#### Стратегия B: Удаление старых сегментов (остановка брокера обязательна)

```bash
# СТОП брокера!
kafka-server-stop.sh
# Убедиться, что процесс не висит:
ps aux | grep kafka

# Удалить сегменты старше 7 дней
# НИКОГДА не удаляйте активный сегмент (самый новый .log в каждой партиции)!
find /var/kafka/data -name "*.log" -mtime +7 -type f -delete
find /var/kafka/data -name "*.index" -mtime +7 -type f -delete
find /var/kafka/data -name "*.timeindex" -mtime +7 -type f -delete

# Запуск брокера
kafka-server-start.sh -daemon /etc/kafka/server.properties
tail -f /var/log/kafka/server.log
```

#### Стратегия C: Расширение диска (для облачных сред)

```bash
# AWS EBS
aws ec2 modify-volume --volume-id vol-xxxx --size 1000
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1

# Для JBOD — временно исключить заполненный диск
# В server.properties:
# Было: log.dirs=/data1/kafka,/data2/kafka,/data3/kafka
# Стало: log.dirs=/data1/kafka,/data2/kafka
# Партиции с отключённого диска станут under-replicated —
# перераспределить через kafka-reassign-partitions.sh
```

**Предотвращение рецидива:**

```properties
# server.properties — страховочные лимиты
log.retention.bytes=107374182400   # 100 GB на партицию
log.retention.check.interval.ms=60000    # Проверка каждую минуту

# Критические пороги мониторинга:
# Warning: диск > 70%
# Critical: диск > 85%
# OfflineLogDirectoryCount > 0 = немедленный алерт
```

### 2.4 Высокая загрузка CPU/памяти

**Симптомы:**
- `process.cpu.load` постоянно высокий
- GC-паузы растут
- Latency для продюсеров и консьюмеров увеличивается

**Матрица диагностики:**

| Причина | Диагностика | Решение |
|---------|-------------|---------|
| Слишком много партиций на брокере | Сравнить `partition count` между брокерами | Перераспределить партиции `kafka-reassign-partitions.sh` |
| Неэффективная компрессия | Проверить `compression.type` | `snappy` — быстрее, `lz4` — лучше баланс, `zstd` — сильнее сжатие, но дороже CPU |
| Слишком много сетевых подключений | `netstat -an \| grep 9092 \| wc -l` | Настроить `max.connections` |
| SSL-терминация нагружает CPU | Проверить, используется ли SSL | Рассмотреть выделенный SSL-терминатор или hardware acceleration |
| Утечка памяти в плагине/интерцепторе | `jmap -heap PID`; heap растёт монотонно | Отключить подозрительные плагины по одному |

---

## 3. Проблемы репликации

### 3.1 Under-replicated partitions

**Ключевая мысль:** `UnderReplicatedPartitions > 0` — это красный флаг. В норме этот показатель должен быть строго равен нулю. Любое ненулевое значение означает, что данные не реплицируются полностью и при отказе лидера возможна потеря.

**Симптомы:**
- JMX-метрика `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions > 0`
- `kafka-topics.sh --describe --under-replicated-partitions` возвращает непустой список
- Возможна потеря данных при отказе брокера-лидера

**Алгоритм диагностики:**

```
Есть under-replicated partitions?
│
├─ Все партиции на одном брокере?
│  └─ Проблема в этом конкретном брокере
│     ├─ Проверить: брокер жив?
│     ├─ Проверить: сеть до брокера?
│     ├─ Проверить: диск брокера не заполнен?
│     └─ Проверить: нагрузка на брокер (CPU/IO)?
│
├─ Число нестабильно (то растёт, то падает)?
│  └─ Performance-проблема в кластере
│     ├─ Проверить балансировку партиций
│     ├─ Проверить ресурсы брокеров (CPU, IO, сеть)
│     └─ Возможно, один брокер перегружен
│
└─ Число растёт монотонно?
   └─ Прогрессирующая деградация
      ├─ Проверить загрузку follower-брокеров
      ├─ Проверить межброкерную сеть (latency, packet loss)
      └─ Проверить дисковую подсистему на follower
```

**Диагностические команды:**

```bash
# Получить список under-replicated partitions
kafka-topics.sh --bootstrap-server broker:9092 \
  --describe --under-replicated-partitions

# Пример вывода (укажет на проблемный брокер):
# Topic: orders  Partition: 5  Leader: 1  Replicas: 1,2,3  Isr: 1,3
# Topic: orders  Partition: 12 Leader: 1  Replicas: 1,2,3  Isr: 1,3
# Тренд: во всех проблемных партициях отсутствует broker 2 в ISR

# Проверить JMX-метрику через jmxterm или Prometheus:
# kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

**Решение проблемы конкретного брокера:**

```bash
# 1. Проверить логи проблемного брокера
ssh broker-2
tail -200 /var/log/kafka/server.log | grep -E "ERROR|WARN"

# 2. Проверить дисковую подсистему
iostat -x 1 5
# Key metric: await (время ожидания IO) — должно быть < 20ms для SSD

# 3. Если брокер работает, но не реплицирует — проверить сеть
ping -c 100 broker-1  # latency до лидера
iperf -c broker-1     # пропускная способность

# 4. Проверить, не «завис» ли процесс репликации
# Перезапуск брокера как последнее средство (если репликация застыла)
```

### 3.2 ISR churn (частые изменения in-sync replicas)

**Симптомы:**
- Постоянные логи о добавлении/удалении брокеров из ISR
- Метрика ISR меняется каждые несколько секунд
- Увеличенная нагрузка на контроллер

**Причины и решения:**

| Причина | Решение |
|---------|---------|
| `replica.lag.time.max.ms` слишком мал | Увеличить — например, с 10с до 30с. Но осторожно: больше значение → дольше задержка перед признанием брокера мёртвым |
| Большие сообщения забивают канал репликации | Уменьшить `max.message.bytes`, использовать компрессию |
| Сетевые микро-разрывы между брокерами | Проверить сетевую инфраструктуру, переключить межброкерный трафик на выделенный интерфейс |
| Follower-брокер перегружен | Перераспределить партиции, добавить брокеры |
| Дисковая подсистема follower не справляется | Перейти на SSD, увеличить IOPS |

---

## 4. Проблемы consumer groups

### 4.1 Consumer lag

**Ключевая мысль:** Consumer lag — самый важной показатель здоровья консьюмера. Lag = разница между последним записанным сообщением и последним прочитанным. Растущий lag означает, что консьюмер отстаёт от продюсера, и разрыв увеличивается.

**Аналогия:** Consumer lag — это как очередь в банке. Если клерк (консьюмер) обслуживает медленнее, чем приходят новые клиенты (продюсеры), очередь растёт. Когда очередь достигает двери — проблемы начинаются и у входящих клиентов (disk full).

**Симптомы:**
- `consumer_lag` монотонно растёт
- Задержка обработки сообщений увеличивается
- Алёрты: `fetch.max.wait.ms exceeded`

**Диагностические команды:**

```bash
# Проверить lag по consumer group
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-consumer-group

# Пример вывода (обратить внимание на LAG):
# GROUP           TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# my-group        orders    0          15000           25000           10000
# my-group        orders    1          5000            25000           20000
#                 ↑ lag 20K — партиция 1 сильно отстаёт

# Мониторинг через JMX (на каждом брокере):
# kafka.server:type=FetcherLagMetrics,\
#   clientId=Replica,*\  name=MaxLag
```

**Матрица причин и решений:**

| Причина | Диагностика | Решение |
|---------|-------------|---------|
| Консьюмер обрабатывает слишком медленно | `max.poll.interval.ms` истекает | Оптимизировать логику обработки; вынести тяжёлые операции в отдельный тред; увеличить `max.poll.interval.ms` |
| Недостаточно консьюмеров | Lag > 0 для всех партиций | Масштабировать consumer instances до количества партиций |
| Дисбаланс назначения партиций | Lag только на некоторых партициях | Проверить, нет ли "горячей" партиции (skewed key distribution) |
| Ресурсные ограничения на хосте консьюмера | CPU/Mem/Network utilisation упирается в потолок | Увеличить ресурсы или оптимизировать код |
| Сетевые проблемы между консьюмером и брокером | `fetch-size` постоянно упирается в лимит | Увеличить `fetch.max.bytes`, `max.partition.fetch.bytes` |
| Слишком мало партиций в топике | Партиций меньше, чем консьюмеров | Увеличить `num.partitions` (внимание: перераспределение ключей!) |

**Пример настройки консьюмера для борьбы с lag:**

```java
// Java: настройка консьюмера для high-throughput сценария
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "high-throughput-group");
// Увеличиваем размер fetch — меньше round-trips
props.put("fetch.min.bytes", "1048576");        // 1 MB
props.put("fetch.max.wait.ms", "500");           // ждём 500ms перед fetch
props.put("max.partition.fetch.bytes", "10485760"); // 10 MB на партицию
// Увеличиваем интервал poll — даём консьюмеру время на обработку
props.put("max.poll.interval.ms", "600000");     // 10 минут
props.put("max.poll.records", "5000");           // до 5000 записей за poll
// Отключаем auto-commit — коммитим руками
props.put("enable.auto.commit", "false");
```

### 4.2 Частые ребалансировки

**Симптомы:**
- Логи показывают ребалансировку каждые 30-60 секунд
- Консьюмеры постоянно переключаются между партициями
- Обработка сообщений прерывается на время ребалансировки

**Матрица причин и решений:**

| Причина | Симптом в логах | Решение |
|---------|-----------------|---------|
| `session.timeout.ms` слишком мал | Heartbeat thread не успевает отправить heartbeat | Увеличить: `session.timeout.ms=30000`, `heartbeat.interval.ms=10000` |
| `max.poll.interval.ms` истекает | Consumer не успевает вызвать `poll()` | Увеличить `max.poll.interval.ms=600000` (10 мин) |
| Консьюмеры часто перезапускаются | Постоянные join/leave группы | Стабилизировать деплой, добавить graceful shutdown |
| Нестабильная сеть | Heartbeat теряются | Проверить сетевую инфраструктуру |
| Много консьюмеров стартуют одновременно | Массовый join вызывает каскад ребалансировок | Настроить `group.initial.rebalance.delay.ms=3000` |

**Рекомендованная конфигурация для стабильных consumer groups:**

```properties
# consumer.properties — стабильный consumer group
group.id=my-stable-group
session.timeout.ms=30000           # 30 секунд — достаточно для heartbeat
heartbeat.interval.ms=10000        # 1/3 от session.timeout
max.poll.interval.ms=600000        # 10 минут — для долгой обработки
group.initial.rebalance.delay.ms=3000  # задержка перед первой ребалансировкой
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
# ↑ Cooperative (Kafka 2.4+) — инкрементальная ребалансировка без полной остановки
```

**Аналогия:** Представьте детскую игру «музыкальные стулья». Когда музыка (обработка) останавливается, все бегут занимать стулья (партиции). Если музыка выключается слишком часто (`session.timeout.ms` мал), никто не успевает сесть и начать работать. Решение — дать больше времени на рассадку (увеличить таймауты).

### 4.3 Consumers не получают сообщения

**Симптомы:**
- Consumer работает, но не обрабатывает новые сообщения
- Lag не уменьшается
- `poll()` возвращает пустой `ConsumerRecords`

**Шаги диагностики:**

```bash
# 1. Проверить, что консьюмер в группе и ему назначены партиции
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-group

# 2. Проверить смещение (offset) — не «убежал» ли он вперёд?
# Если CURRENT-OFFSET == LOG-END-OFFSET → консьюмер всё прочитал
# Если CURRENT-OFFSET > LOG-END-OFFSET → ошибка offset management

# 3. Сбросить offset если он некорректен (ОСТОРОЖНО: потеря данных!)
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --group my-group --topic my-topic \
  --reset-offsets --to-earliest --execute

# Альтернативы сброса:
# --to-latest    → только новые сообщения
# --to-datetime  → на конкретное время
# --shift-by N   → сдвинуть на N сообщений (N может быть отрицательным)
# --to-offset X  → на конкретный offset
```

### 4.4 Дубликаты сообщений

**Симптомы:**
- Одно и то же сообщение обрабатывается дважды (или больше)
- Данные в downstream-системе дублируются

**Причины:**

| Причина | Когда происходит | Решение |
|---------|-----------------|---------|
| Консьюмер упал после обработки, но до коммита offset | Offset не закоммичен → при перезапуске повтор | Ручной коммит offset ПОСЛЕ обработки |
| `enable.auto.commit=true` + краш между авто-коммитами | Интервал между авто-коммитами — окно для дубликатов | Отключить auto-commit, коммитить руками |
| Ребалансировка во время обработки | Партиция переходит к другому консьюмеру | CooperativeStickyAssignor + Graceful shutdown |

**Паттерн идемпотентного консьюмера:**

```python
# Python: идемпотентный consumer через проверку message_id
from kafka import KafkaConsumer
import hashlib
import redis

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers='broker:9092',
    group_id='order-processor',
    enable_auto_commit=False,  # Ручной коммит
    auto_offset_reset='earliest'
)

# Redis как хранилище обработанных message_id
processed_ids = redis.Redis(host='localhost', port=6379, db=0)

MESSAGE_ID_HEADER = 'message_id'

for message in consumer:
    try:
        # Генерируем или извлекаем уникальный ID сообщения
        msg_id = None
        for header in message.headers:
            if header[0] == MESSAGE_ID_HEADER:
                msg_id = header[1].decode()
                break

        if msg_id is None:
            # Нет idempotency key — генерируем из содержимого
            msg_id = hashlib.sha256(message.value).hexdigest()

        # Проверяем, не обработано ли уже
        if processed_ids.exists(msg_id):
            print(f"Сообщение {msg_id} уже обработано — пропускаем")
            consumer.commit()  # Всё равно коммитим offset
            continue

        # --- Бизнес-логика обработки ---
        process_order(message.value)

        # Сохраняем факт обработки (с TTL = retention окно)
        processed_ids.setex(msg_id, 86400, '1')  # 24 часа

        # Коммитим offset ПОСЛЕ успешной обработки
        consumer.commit()

    except Exception as e:
        print(f"Ошибка обработки: {e}")
        # Offset не коммитим — сообщение будет переобработано
        # Это корректно, т.к. наша логика идемпотентна
```

---

## 5. Проблемы продюсеров

**Ключевая мысль:** Продюсер — входная точка данных в Kafka. Ошибки на этом уровне каскадно влияют на всё downstream. Большинство проблем продюсера решаются правильной конфигурацией `acks`, `retries` и `enable.idempotence`.

**Симптомы проблем продюсера:**
- `TimeoutException` при отправке
- `RecordTooLargeException`
- Дубликаты в топике (без идемпотентности)
- `NotEnoughReplicasException`

**Матрица symptom→cause→solution:**

| Симптом | Причина | Решение |
|---------|---------|---------|
| `TimeoutException` | Брокер не отвечает за `delivery.timeout.ms` | Увеличить `delivery.timeout.ms`, проверить сеть до брокера, проверить нагрузку брокера |
| `RecordTooLargeException` | Сообщение превышает `max.request.size` на продюсере или `message.max.bytes` на брокере | Уменьшить размер сообщения или увеличить лимиты на обеих сторонах |
| Дубликаты | `enable.idempotence=false` + retries | Включить `enable.idempotence=true` (доступно с acks=all) |
| `NotEnoughReplicasException` | `acks=all`, но недостаточно in-sync реплик | Увеличить `min.insync.replicas` на брокере; добавить брокеры; проверить ISR |
| Низкая пропускная способность | Маленький `batch.size`, нет компрессии | Увеличить `batch.size` до 64KB+, `linger.ms=5`, включить `compression.type=lz4` |

**Рекомендованная конфигурация продюсера для production:**

```properties
# producer.properties — production-grade
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# Надёжность (durability):
acks=all                           # ждём подтверждения от всех ISR
enable.idempotence=true            # исключает дубликаты при retries
max.in.flight.requests.per.connection=5  # 5 — максимум с идемпотентностью

# Производительность (performance):
compression.type=lz4               # хороший баланс CPU/compression ratio
batch.size=65536                   # 64 KB — больше = эффективнее
linger.ms=5                        # ждать 5ms перед отправкой батча
buffer.memory=33554432             # 32 MB буфер

# Таймауты:
delivery.timeout.ms=120000         # 2 минуты на доставку
request.timeout.ms=30000           # 30 секунд на один запрос
retries=2147483647                 # максимальное число повторных попыток
```

**Аналогия:** Продюсер — это курьер, доставляющий посылки на склад (брокер). `acks=all` означает, что курьер ждёт подписи от начальника смены, его зама и бухгалтера. `enable.idempotence=true` — это наклейка с уникальным штрих-кодом: если курьер сомневается, доставлена ли посылка, он пробует ещё раз, а склад говорит «спасибо, мы уже получили эту».

---

## 6. Проблемы Kafka Connect

**Ключевая мысль:** Kafka Connect — самый сложный для отладки компонент, потому что проблемы могут быть на любом из трёх уровней: сам Connect worker, конкретный коннектор, внешняя система. Отладка всегда начинается с REST API Connect.

**Диагностические команды:**

```bash
# Проверить статус всех коннекторов
curl -s http://connect-worker:8083/connectors | jq

# Статус конкретного коннектора и его задач
curl -s http://connect-worker:8083/connectors/my-connector/status | jq

# Пример ответа:
# {
#   "name": "my-connector",
#   "connector": {"state": "RUNNING", "worker_id": "worker1:8083"},
#   "tasks": [
#     {"id": 0, "state": "FAILED", "trace": "org.apache.kafka.connect.errors..."},
#     {"id": 1, "state": "RUNNING", "worker_id": "worker1:8083"}
#   ]
# }

# Получить stack trace упавшей задачи
curl -s http://connect-worker:8083/connectors/my-connector/tasks/0/status | jq

# Перезапустить упавшую задачу (не весь коннектор)
curl -X POST http://connect-worker:8083/connectors/my-connector/tasks/0/restart

# Приостановить/возобновить коннектор
curl -X PUT http://connect-worker:8083/connectors/my-connector/pause
curl -X PUT http://connect-worker:8083/connectors/my-connector/resume
```

**Матрица типовых проблем:**

| Симптом | Причина | Решение |
|---------|---------|---------|
| Коннектор RUNNING, но задача FAILED | Ошибка в одной задаче (e.g., bad record, schema mismatch) | Получить trace через REST API; исправить данные/схему; перезапустить задачу |
| «Task is being killed and will not recover until manually restarted» | Общая ошибка-симптом — искать глубже в trace | Читать полный stack trace в логах worker; это не первопричина |
| Sink connector: данные не пишутся | Проблема с downstream системой (DB, S3 и т.д.) | Проверить доступность downstream, креденшелы, схему таблиц |
| Source connector: данные не читаются | Проблема с upstream системой или конфигурацией | Проверить connectivity, polling interval, query |
| Connect worker не стартует | Конфликт `rest.advertised.host.name` или plugin path | Проверить конфигурацию worker, `plugin.path` |
| Ребалансировка убивает задачи | Слишком частые ребалансировки worker'ов (старый eager-протокол) | Перейти на incremental cooperative rebalancing (Kafka 2.3+) |
| Out of Memory на worker | Слишком много больших сообщений в памяти | Увеличить heap, уменьшить `consumer.max.poll.records` |

**Динамическое изменение уровня логирования (Kafka 2.4+):**

```bash
# Включить TRACE-логирование для конкретного коннектора БЕЗ перезапуска
curl -X PUT http://connect-worker:8083/admin/loggers/org.apache.kafka.connect.file \
  -H "Content-Type: application/json" \
  -d '{"level": "TRACE"}'

# Вернуть обратно
curl -X PUT http://connect-worker:8083/admin/loggers/org.apache.kafka.connect.file \
  -H "Content-Type: application/json" \
  -d '{"level": "INFO"}'

# Посмотреть текущие уровни
curl -s http://connect-worker:8083/admin/loggers | jq
```

**Аналогия:** Kafka Connect — это конвейерная лента на заводе. Коннектор = вся линия, задачи (tasks) = отдельные станки на линии. Если один станок (task) сломался — достаточно починить его, не останавливая всю линию. REST API = панель управления конвейером.

---

## 7. Проблемы KRaft (Kafka 4.x)

**Ключевая мысль:** С переходом на KRaft в Kafka 4.0 диагностика метаданных кластера переехала из ZooKeeper в сам Kafka. У этого есть преимущества (единый стек), но и новые классы проблем.

**Основные отличия диагностики KRaft vs ZooKeeper:**

| Аспект | ZooKeeper | KRaft |
|--------|-----------|-------|
| Где метаданные | Внешний ZK ensemble | Внутренний metadata log (__cluster_metadata) |
| Кто хранит | 3-5 ZK-нод | 3-5 контроллеров Kafka |
| Диагностика | `zkCli.sh`, 4-letter words | `kafka-metadata-quorum.sh`, controller.log |
| Состояние кворума | `mntr` → `zk_server_state` | `kafka-metadata-quorum.sh --describe` |

**Диагностические команды KRaft:**

```bash
# Состояние метадата-кворума
kafka-metadata-quorum.sh --bootstrap-server controller:9093 \
  --describe --status

# Пример вывода:
# NodeId  Host          Port  Status
# 1       controller1   9093  Leader
# 2       controller2   9093  Follower
# 3       controller3   9093  Follower

# Проверка replicas (аналог zkCli.sh ls /brokers/ids)
kafka-metadata-quorum.sh --bootstrap-server controller:9093 \
  --describe --replication

# Проверка состояния снапшотов
find /var/kafka/metadata -name "*.checkpoint" -ls
```

**Типовые проблемы KRaft:**

| Симптом | Причина | Решение |
|---------|---------|---------|
| Контроллер не может стать лидером | Потерян кворум (нет большинства контроллеров) | Восстановить минимум 2 из 3 контроллеров |
| `No leader for partition` | Metadata log повреждён или рассинхронизирован | Проверить `controller.log`; при необходимости восстановить из snapshot |
| Брокер не регистрируется в кластере | Не может связаться с активным контроллером | Проверить `controller.quorum.voters` в `server.properties` |
| Metadata disk full | Metadata log растёт без очистки | Проверить `metadata.log.dir`, snapshot retention |
| Диск контроллера отказал | KIP-856: Disk Failure Recovery | Заменить диск; контроллер восстановит метаданные из кворума |

**Сравнение: KRaft vs ZooKeeper troubleshooting effort — KRaft выигрывает по простоте:**
- Не нужно держать экспертизу в ZooKeeper
- Единый язык конфигурации
- Единая система логирования
- Меньше moving parts = меньше точек отказа

---

## 8. Диагностический инструментарий

### 8.1 CLI-инструменты (kafka-*.sh)

| Инструмент | Назначение | Ключевые команды |
|-----------|------------|-----------------|
| `kafka-topics.sh` | Управление топиками | `--describe`, `--list`, `--under-replicated-partitions` |
| `kafka-consumer-groups.sh` | Consumer groups | `--describe --group`, `--reset-offsets`, `--list` |
| `kafka-broker-api-versions.sh` | Проверка доступности брокеров | `--bootstrap-server` |
| `kafka-configs.sh` | Динамическая конфигурация | `--describe`, `--alter`, `--delete-config` |
| `kafka-reassign-partitions.sh` | Перераспределение партиций | `--generate`, `--execute`, `--verify` |
| `kafka-log-dirs.sh` | Информация о log-директориях | `--describe`, `--bootstrap-server` |
| `kafka-metadata-quorum.sh` | KRaft metadata quorum | `--describe --status` |
| `kafka-dump-log.sh` | Инспекция log-сегментов | `--files`, `--deep-iteration` |
| `kafka-delegation-tokens.sh` | Управление delegation tokens | `--describe`, `--renew`, `--expire` |
| `kafka-acls.sh` | Управление ACL | `--list`, `--add`, `--remove` |

### 8.2 JMX-метрики (must-have)

**Таблица критически важных JMX-метрик для production-мониторинга:**

| MBean | Метрика | Значение | Критический порог |
|-------|---------|----------|-------------------|
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Value | Кол-во under-replicated партиций | > 0 |
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | Value | Активных контроллеров | ≠ 1 |
| `kafka.controller:type=KafkaController,name=OfflinePartitionsCount` | Value | Офлайн-партиций | > 0 |
| `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` | Count | Сокращений ISR в секунду | > 0.1 |
| `kafka.network:type=RequestChannel,name=RequestQueueSize` | Value | Размер очереди запросов | > 500 |
| `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent` | Value | Простой network processor | < 0.2 (20%) |
| `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent` | Value | Простой request handler | < 0.3 (30%) |
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | OneMinuteRate | Входящие данные (байт/с) | Зависит от baseline |
| `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec` | OneMinuteRate | Исходящие данные (байт/с) | Зависит от baseline |
| `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec` | OneMinuteRate | Входящих сообщений/с | Зависит от baseline |
| `java.lang:type=GarbageCollector,name=G1 Young Generation` | LastGcInfo → duration | Длительность GC (ms) | > 1000ms |
| `java.lang:type=Memory` | HeapMemoryUsage → used | Использование heap | > 85% от max |

### 8.3 Системные утилиты

```bash
# CPU и память
top -p $(pgrep -f kafka) -bn1
# Load average, %CPU, %MEM

# Поиск утечек памяти
jstat -gcutil $(pgrep -f kafka) 1000 10
# S0/S1/E/O/M  — если Old Gen растёт монотонно = утечка

# Дамп потоков
jstack $(pgrep -f kafka) > thread_dump_$(date +%s).txt
# Для анализа зависаний, deadlocks

# Heap dump для анализа памяти
jmap -dump:format=b,file=heap_$(date +%s).hprof $(pgrep -f kafka)
# Анализ: Eclipse MAT

# Дисковый IO
iostat -x 1 10
# await (ms) — ключевая метрика задержки диска

# Инспекция log-сегмента (чтение содержимого)
kafka-dump-log.sh --files /var/kafka/data/my-topic-0/00000000000000000000.log \
  --print-data-log
# Показывает смещения, временные метки, ключи и значения записей

# Генератор тестовой нагрузки (оценка производительности)
kafka-producer-perf-test.sh --topic test-topic \
  --num-records 1000000 --record-size 1024 \
  --throughput -1 --producer-props bootstrap.servers=broker:9092 acks=1
```

### 8.4 Анализ логов

**Структура логов и что искать:**

```bash
# Основной лог брокера — все операции и ошибки
tail -500 /var/log/kafka/server.log | grep -E "ERROR|WARN"

# Лог переходов лидерства партиций
tail -100 /var/log/kafka/state-change.log
# Ключевые фразы: "Leader changed", "ISR changed", "Controller election"

# Лог контроллера — метаданные кластера
tail -100 /var/log/kafka/controller.log
# Ключевые фразы: "Controller elected", "Partition reassignment", "Topic deletion"

# Лог log cleaner — фоновые процессы очистки
tail -100 /var/log/kafka/log-cleaner.log
# Ключевые фразы: "Log cleaner completed", "Error cleaning log"

# Анализ времени в логах (паттерны периодичности)
grep "ERROR" /var/log/kafka/server.log \
  | awk '{print $1, $2}' \
  | uniq -c
# Показывает кол-во ошибок по временным интервалам
```

---

## 9. Процедуры аварийного реагирования

### 9.1 Падение брокера (Emergency Runbook)

```
Время: T+0 минут
┌─────────────────────────────────────────┐
│ Шаг 1: Оценка влияния                   │
├─────────────────────────────────────────┤
│ Проверить, жив ли кластер без брокера:  │
│ $ kafka-broker-api-versions.sh ...      │
│ $ kafka-topics.sh --under-replicated    │
│                                         │
│ Если replication factor >= 2 и другие   │
│ брокеры в порядке → кластер работает.    │
│ Можно без паники.                       │
└─────────────────────────────────────────┘

Время: T+5 минут
┌─────────────────────────────────────────┐
│ Шаг 2: Сбор диагностики на упавшем      │
│ брокере (SSH, пока не перезагружали)    │
├─────────────────────────────────────────┤
│ $ dmesg | tail -50                      │
│ $ tail -200 server.log | grep ERROR     │
│ $ jstack PID                            │
│ $ jmap -heap PID                        │
│ $ df -h                                 │
│ $ free -h                               │
└─────────────────────────────────────────┘

Время: T+10 минут
┌─────────────────────────────────────────┐
│ Шаг 3: Попытка восстановления           │
├─────────────────────────────────────────┤
│ Если OOM → увеличить heap, перезапустить│
│ Если disk full → раздел 2.3             │
│ Если log corruption → удалить .index,   │
│    перезапустить                        │
│ Если неясно → перезапустить брокер      │
│    $ kafka-server-start.sh -daemon ...  │
│    $ tail -f server.log                 │
└─────────────────────────────────────────┘

Время: T+15 минут
┌─────────────────────────────────────────┐
│ Шаг 4: Верификация                      │
├─────────────────────────────────────────┤
│ $ kafka-topics.sh --under-replicated    │
│   (ждём, пока опустеет)                │
│ $ kafka-consumer-groups.sh --describe   │
│   (lag не растёт)                       │
│ Проверить продюсеров/консьюмеров       │
└─────────────────────────────────────────┘
```

### 9.2 Предотвращение потери данных

**Чеклист ДО перезапуска проблемного брокера:**

- [ ] Сделать бэкап `server.properties`
- [ ] Сделать бэкап JAAS-конфигурации (если есть SASL)
- [ ] Записать текущие ошибки из логов
- [ ] Если возможно — скопировать `log.dirs` на резервное хранилище
- [ ] Проверить статус репликации критических топиков

**Чеклист ПОСЛЕ восстановления:**

- [ ] `UnderReplicatedPartitions == 0` на всех брокерах
- [ ] `ActiveControllerCount == 1`
- [ ] ISR в норме для всех критических топиков
- [ ] Consumer lag не выше baseline
- [ ] Продюсеры и консьюмеры восстановили работу
- [ ] Нет новых ERROR в логах

---

## 10. Устранение неисправностей безопасности

### 10.1 С точки зрения инфраструктурного безопасника

**Фокус:** сетевые порты, межброкерная аутентификация, TLS-сертификаты, файрволлы.

**Проблема: Клиент не может подключиться к брокеру по TLS**

```bash
# Диагностика TLS handshake
openssl s_client -connect broker:9093 -tls1_2 -showcerts \
  -CAfile /path/to/ca-cert.pem

# Типичные ошибки и решения:
# "verify error:num=21:unable to verify the first certificate"
#   → отсутствует корневой CA-сертификат в truststore клиента
# "ssl handshake failure: certificate expired"
#   → проверить срок действия сертификата:
#     openssl x509 -in broker-cert.pem -noout -enddate

# Проверка доступности портов
nmap -p 9092,9093 broker-hostname
```

**Проблема: Межброкерная аутентификация не работает**

```bash
# Проверить конфигурацию listeners в server.properties
grep -E "listeners|advertised|inter.broker" /etc/kafka/server.properties

# Должно быть примерно так:
# listeners=SASL_SSL://:9093,PLAINTEXT://:9092
# advertised.listeners=SASL_SSL://broker1.example.com:9093,PLAINTEXT://broker1.example.com:9092
# inter.broker.listener.name=SASL_SSL
# security.inter.broker.protocol=SASL_SSL

# Проверить JAAS-конфигурацию
cat /etc/kafka/kafka_server_jaas.conf
# Секция KafkaServer должна присутствовать и содержать корректный keytab/principal
```

**Проблема: Истекающие TLS-сертификаты**

```bash
# Скрипт мониторинга истечения сертификатов
#!/bin/bash
CERT_DIR="/etc/kafka/certs"
DAYS_THRESHOLD=30

for cert in $CERT_DIR/*.pem; do
    expiry=$(openssl x509 -in "$cert" -noout -enddate | cut -d= -f2)
    expiry_epoch=$(date -d "$expiry" +%s)
    now_epoch=$(date +%s)
    days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

    if [ $days_left -lt $DAYS_THRESHOLD ]; then
        echo "⚠️  $cert: истекает через $days_left дней ($expiry)"
    fi
done
```

**Инфраструктурный чеклист безопасности при инциденте:**

- [ ] Проверить, не изменились ли правила файрволла на портах Kafka (9092/9093/9094)
- [ ] Проверить, доступны ли все порты listeners с клиентских машин
- [ ] Проверить валидность всех сертификатов в цепочке доверия
- [ ] Проверить, что `ssl.client.auth` настроен корректно (required/requested/none)
- [ ] Проверить логи аудита ОС на предмет подозрительной активности вокруг Kafka-процессов

### 10.2 С точки зрения DevSecOps

**Фокус:** CI/CD pipeline, secrets management, контейнеры, сканирование конфигураций.

**Проблема: SASL-аутентификация падает после деплоя**

```bash
# CI pipeline — проверка конфигурации ДО деплоя
kafka-configs.sh --bootstrap-server staging-broker:9093 \
  --describe --entity-type users --entity-name my-service-user \
  --command-config /etc/kafka/admin-client.conf

# Валидация, что секреты загружены корректно
echo "mechanics" | kcat -L -b broker:9093 \
  -X security.protocol=SASL_SSL \
  -X sasl.mechanism=SCRAM-SHA-512 \
  -X sasl.username=my-service-user \
  -X sasl.password=$KAFKA_PASSWORD

# Ожидаемый результат: metadata о топиках и брокерах, а не auth error
```

**Проблема: Несоответствие конфигурации security protocol между средами**

```bash
# Валидация в CI — сравнение prod и staging
# Проверить, что security.protocol одинаковый
diff <(grep "security.protocol" staging.properties | sort) \
     <(grep "security.protocol" prod.properties | sort)

# Проверить, что SASL mechanism одинаковый
diff <(grep "sasl.mechanism" staging.properties | sort) \
     <(grep "sasl.mechanism" prod.properties | sort)
```

**Проблема: Утечка секретов в CI-логах**

```bash
# GitGuardian / truffleHog — сканирование коммитов на секреты
trufflehog git file://. --since-commit HEAD~10 --json

# Проверить, не выводится ли пароль в CI log
# Использовать маскированные переменные
# GitHub Actions example:
# env:
#   KAFKA_PASSWORD: ${{ secrets.KAFKA_PASSWORD }}
# run: |
#   echo "Password length: ${#KAFKA_PASSWORD}"  # OK — не выводим значение
#   kcat -b broker:9093 -X sasl.password="$KAFKA_PASSWORD" ...
```

**Сканирование Kafka-образов на уязвимости:**

```bash
# Trivy-сканирование базового образа Kafka
trivy image --severity HIGH,CRITICAL confluentinc/cp-kafka:7.8.0

# Проверка Dockerfile на best practices
hadolint Dockerfile.kafka
```

**DevSecOps-ранбук при security-инциденте:**

- [ ] Ревокнуть скомпрометированные креденшелы через `kafka-configs.sh --alter --delete-config`
- [ ] Перегенерировать SCRAM-пароли или keytab-файлы
- [ ] Проверить pipeline audit log — кто и когда вносил изменения в security-конфигурацию
- [ ] Запустить сканирование репозитория на предмет утечек секретов
- [ ] Убедиться, что ротация секретов отработала во всех средах (dev → staging → prod)

### 10.3 С точки зрения архитектора ИБ

**Фокус:** threat model, compliance, расследование инцидентов, governance.

**Методика расследования security-инцидента в Kafka:**

```
Инцидент: неавторизованный доступ к топику / подозрительная активность

Фаза 1: Обнаружение и изоляция (T+0 часов)
├─ Немедленно проверить authorizer.log
│  grep "DENIED\|ALLOWED" /var/log/kafka/kafka-authorizer.log | \
│    awk '{print $1,$2,$NF}' | uniq -c | sort -rn | head -20
├─ Проверить ACL для подозрительного топика
│  kafka-acls.sh --bootstrap-server broker:9093 \
│    --list --topic suspicious-topic \
│    --command-config admin-client.conf
├─ Изолировать скомпрометированный принципал
│  kafka-acls.sh --bootstrap-server broker:9093 \
│    --remove --allow-principal User:compromised-user --operation All \
│    --topic "*" --command-config admin-client.conf
└─ Зафиксировать evidence: дамп authorizer.log + ACL snapshot

Фаза 2: Расследование (T+24 часа)
├─ Проанализировать логи аудита на всех брокерах
├─ Определить blast radius: какие топики/данные затронуты
├─ Проверить audit trail: кто и когда создал/изменил скомпрометированные ACL
├─ Оценить, были ли данные эксфильтрованы (BytesOutPerSec аномалии)
└─ Проверить KRaft metadata log на предмет изменений

Фаза 3: Восстановление и hardening (T+72 часа)
├─ Восстановить корректные ACL
├─ Провести ретроспективу: почему стало возможно?
├─ Обновить threat model
├─ Добавить детектирующие контроли:
│  - Алерт на DENIED > N в минуту
│  - Алерт на изменение ACL не из CI
└─ Обновить процедуры ротации ключей
```

**Threat Model: проверка целостности данных**

```bash
# Сравнение текущего состояния с baseline
# Сохранить baseline ACL
kafka-acls.sh --bootstrap-server broker:9093 --list > acl_baseline.txt

# Периодически сверять с baseline (через cron или CI)
diff acl_baseline.txt <(kafka-acls.sh --bootstrap-server broker:9093 --list)
# Любое изменение, не прошедшее через CI = инцидент
```

**Compliance-матрица при security-расследовании:**

| Аспект | Что проверять | Инструмент | Срок хранения |
|--------|---------------|------------|--------------|
| Кто читал данные | `BytesOutPerSec` + consumer group offsets | JMX → Prometheus → Grafana | 90 дней |
| Кто изменял ACL | `kafka-acls.sh --list` diff + audit log | Git history + `authorizer.log` | 365 дней |
| Аутентификация | `server.log` → SASL handshake | ELK/Loki | 90 дней |
| Авторизация | `kafka-authorizer.log` → DENIED/ALLOWED | SIEM | 365 дней |
| Конфигурация | `server.properties` diff | Git (IaC) | Бессрочно |

---

## Итоги

**Ключевые выводы:**

1. **Системный подход критичен.** Бессистемная диагностика в распределённой системе — гарантированный долгий даунтайм. Следуйте 5-шаговому алгоритму: масштаб → базовые проверки → логи → метрики → корневая причина.

2. **Никогда не перезапускайте брокер без снятия диагностики.** Thread dump, heap dump, GC-логи — эти данные невосстановимы после перезапуска. Сначала соберите, потом перезапускайте.

3. **UnderReplicatedPartitions > 0 — всегда инцидент.** Это не «предупреждение», это прямой сигнал, что репликация нарушена и есть риск потери данных. Мониторинг этой метрики с алертом на любое ненулевое значение обязателен для production.

4. **Consumer lag — главный индикатор здоровья консьюмеров.** Растущий lag означает, что данные не обрабатываются. Настройте rate-based алертинг (lag растёт на N% в минуту), а не абсолютный (lag > X).

5. **Disk full — предотвратимая катастрофа.** Каждый инцидент с заполнением диска можно было предотвратить мониторингом на уровне 70%. Настройте алерты ЗАРАНЕЕ и включите `log.retention.bytes` как страховочный лимит.

6. **Безопасность — неотъемлемая часть troubleshooting.** Ошибки SASL/TLS/ACL — это такой же эксплуатационный инцидент, как и проблемы с репликацией. Держите JAAS-конфигурацию, сертификаты и ACL под контролем версий (IaC), мониторьте `kafka-authorizer.log` и expiration сертификатов.

7. **Runbook'и спасают.** Задокументированный порядок действий для disk full, broker failure и security incident сокращает время восстановления с часов до минут. Проводите drills ежеквартально.

**Что дальше:** В следующей статье серии «Operations & Observability» — [05-automation.md](./05-automation.md) — мы рассмотрим автоматизацию операций Kafka через Infrastructure as Code (Ansible, Terraform), GitOps-воркфлоу и automated provisioning.

---

## Источники

### Официальная документация
1. [Apache Kafka Documentation — Operations](https://kafka.apache.org/documentation/#operations) — официальное руководство по эксплуатации
2. [Apache Kafka Documentation — Security](https://kafka.apache.org/documentation/#security) — официальное руководство по безопасности
3. [Confluent Documentation — Monitoring Kafka](https://docs.confluent.io/platform/current/kafka/monitoring.html) — мониторинг Kafka от Confluent
4. [KIP-449: Add connector contexts to Connect logs](https://cwiki.apache.org/confluence/display/KAFKA/KIP-449%3A+Add+connector+contexts+to+Connect+logs) — улучшенное логирование Connect
5. [KIP-856: KRaft Disk Failure Recovery](https://cwiki.apache.org/confluence/display/KAFKA/KIP-856%3A+KRaft+Disk+Failure+Recovery) — восстановление дисковых отказов в KRaft
6. [KIP-858: Handle JBOD broker disk failure in KRaft](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=217392620) — обработка отказов JBOD дисков в KRaft

### Технические блоги и руководства
7. [Confluent — Apache Kafka Issues in Production: How to Diagnose and Prevent Failures](https://www.confluent.io/learn/kafka-issues-production/) — обзор типовых production-проблем
8. [Conduktor — Disk Full: Emergency Recovery When Kafka Runs Out of Space](https://www.conduktor.io/blog/disk-full-emergency-recovery) — emergency recovery при заполнении диска
9. [SkillCaptain — Troubleshooting Under-Replicated Kafka Partitions](https://skillcaptain.substack.com/p/troubleshooting-under-replicated) — детальный разбор under-replicated partitions
10. [Confluent Developer — Debugging and Troubleshooting Kafka Connect](https://developer.confluent.io/courses/kafka-connect/troubleshooting-kafka-connect/) — отладка Kafka Connect
11. [KLogic — Kafka Troubleshooting Guide](https://klogic.io/blog/kafka-troubleshooting/) — comprehensive troubleshooting guide
12. [DevOps AIBIT — Troubleshooting Kafka Broker Failures and Recovery Strategies](https://devops.aibit.im/article/troubleshooting-kafka-broker-failures) — отказы брокеров и стратегии восстановления
13. [DevOps AIBIT — Troubleshooting Common Kafka Consumer Group Issues](https://devops.aibit.im/article/troubleshooting-kafka-consumer-groups) — проблемы consumer groups

### Общепризнанные знания / Verified Knowledge
- «Kafka: The Definitive Guide» (O'Reilly, 2nd Edition) — главы по monitoring, operations и security
- Apache Kafka source code (GitHub) — реализация ReplicaManager, KafkaController, GroupCoordinator
- Confluent Community Slack / Kafka Summit talks — production experience, архитектурные решения
- Apache Kafka Improvement Proposals (KIP) — KIP-405 (tiered storage), KIP-500 (KRaft), KIP-928 (resilience to full logs)

---

## Связанные статьи

- [01-monitoring.md](./01-monitoring.md) — Мониторинг Kafka: метрики, дашборды, алерты
- [02-logging.md](./02-logging.md) — Логирование Kafka: форматы, агрегация, аудит
- [03-backup-recovery.md](./03-backup-recovery.md) — Резервное копирование и аварийное восстановление
- [05-automation.md](./05-automation.md) — Автоматизация операций Kafka (следующая статья)
- [../../08-security/01-security-features.md](../08-security/01-security-features.md) — Встроенные механизмы безопасности Kafka
- [../../06-performance/04-bottlenecks.md](../06-performance/04-bottlenecks.md) — Узкие места производительности Kafka
