# Лучшие практики Docker Security

## Чек-лист

### Образы

- [ ] Минимальный базовый образ (Alpine, distroless)
- [ ] Многоэтапная сборка
- [ ] Не root
- [ ] Обновление зависимостей
- [ ] Сканирование перед деплоем
- [ ] Подпись образов
- [ ] Не секреты в Dockerfile

### Runtime

- [ ] --cap-drop=ALL
- [ ] --read-only
- [ ] --security-opt
- [ ] --user
- [ ] --memory, --cpus
- [ ] --pids-limit
- [ ] --no-new-privileges

### Сеть

- [ ] Не --network host
- [ ] Ограничение портов
- [ ] Internal networks
- [ ] Firewall

### Secrets

- [ ] Docker secrets или Vault
- [ ] Не ENV для паролей
- [ ] Ротация

### Registry

- [ ] Приватный registry
- [ ] Сканирование
- [ ] Подпись
- [ ] RBAC

### Мониторинг

- [ ] Audit logs
- [ ] Container runtime monitoring
- [ ] Network monitoring
- [ ] Image vulnerability scanning

## Архитектура

```
Developer → Dockerfile → Build → Scan → Sign → Push → Registry
                                                    ↓
Runtime → Pull → Verify → Run (seccomp, AppArmor, caps drop)
```

## Итог

| Уровень | Защита |
|---------|--------|
| Образ | Минимальный, scanned, signed |
| Runtime | Seccomp, AppArmor, no root |
| Сеть | Segmented, limited |
| Secrets | Vault, rotation |
| Мониторинг | Audit, alerts |

---
_Статья создана на основе анализа материалов Habr_
