# Docker Bench Security

## Что такое?

| Параметр | Описание |
|----------|----------|
| CIS Benchmark | Правила безопасности |
| Docker Bench | Скрипт проверки |
| Проверки | 100+ тестов |

## Категории

| Категория | Проверки |
|-----------|----------|
| Host | Конфигурация хоста |
| Docker daemon | Настройки daemon |
| Docker files | Dockerfile |
| Images | Образы |
| Runtime | Запущенные контейнеры |
| Security operations | Дополнительно |

## Пример проверок

| Проверка | Результат |
|----------|-----------|
| Не используется root | PASS |
| Read-only filesystem | PASS |
| Dropped capabilities | PASS |
| No new privileges | PASS |
| Memory limit | WARN |

## Запуск

```bash
docker run -it --net host --pid host --userns host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/lib/systemd:/usr/lib/systemd \
  -v /etc:/etc --label docker_bench_security \
  docker/docker-bench-security
```

---
_Статья создана на основе анализа материалов Habr_
