# Rootless Docker

## Проблема root

| Проблема | Описание |
|----------|----------|
| Docker daemon | Работает от root |
| Container escape | Доступ к хосту |
| Privileged | Полный контроль |
| Socket | /var/run/docker.sock |

## Rootless

| Параметр | Описание |
|----------|----------|
| User namespace | Пользователь root внутри — обычный снаружи |
| Daemon | Работает от пользователя |
| Контейнеры | Без root на хосте |
| Ограничения | Нет некоторых функций |

## Настройка

```bash
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix:///run/user/1000/docker.sock
```

## Лучшие практики

| Практика | Описание |
|----------|----------|
| Rootless | По возможности |
| User namespace | Всегда |
| Не privileged | Никогда |
| Socket | Ограничить доступ |

---
_Статья создана на основе анализа материалов Habr_
