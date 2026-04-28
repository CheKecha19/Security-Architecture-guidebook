# Лучшие практики AI Security

## Чек-лист

### Модель

- [ ] Проверенные данные для обучения
- [ ] Проверка на poisoning
- [ ] Подпись модели
- [ ] Регулярное обновление
- [ ] Monitoring на аномалии

### Инференс

- [ ] Input validation
- [ ] Output filtering
- [ ] Rate limiting
- [ ] Authentication
- [ ] Logging

### Данные

- [ ] Нет sensitive в обучении
- [ ] Differential privacy
- [ ] Data sanitization
- [ ] Audit доступа

### Prompt

- [ ] Защита от injection
- [ ] Jailbreaking detection
- [ ] Context isolation
- [ ] System prompt защита

### Governance

- [ ] AI Ethics board
- [ ] Impact assessment
- [ ] Compliance
- [ ] Documentation

## Архитектура

`
User → API Gateway → Auth → Input Filter → LLM → Output Filter → User
              ↓           ↓                ↓
         Logging    Monitoring      Rate Limit
`

## Итог

| Уровень | Защита |
|---------|--------|
| Данные | Проверенные, sanitized |
| Модель | Подпись, monitored |
| Инференс | Filtered, rate-limited |
| Prompt | Protected |
| Governance | Compliant |

---
_Статья создана на основе анализации материалов Habr_
