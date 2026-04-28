# Безопасность Docker Compose

## Структура

```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    networks:
      - frontend
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend
networks:
  frontend:
  backend:
    internal: true
```

## Безопасность

| Параметр | Описание |
|----------|----------|
| user | Не root |
| read_only | Только чтение |
| cap_drop | Убрать capabilities |
| security_opt | Seccomp, AppArmor |
| networks | Сегментация |
| secrets | Docker secrets |

## Лучшие практики

| Практика | Описание |
|----------|----------|
| internal: true | Без доступа извне |
| depends_on | Порядок запуска |
| restart | Политика |
| healthcheck | Проверка здоровья |
| resources | Ограничения |

---
_Статья создана на основе анализа материалов Habr_
