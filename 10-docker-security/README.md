# Docker Security — Безопасность контейнеров

## Содержание раздела

| # | Статья | Тема |
|---|--------|------|
| 01 | [Docker Architecture](01-docker-architecture.md) | Namespaces, cgroups, overlayfs, daemon |
| 02 | [Image Security](02-image-security.md) | Сканирование образов, минимальные base images, multi-stage |
| 03 | [Container Runtime](03-container-runtime.md) | Seccomp, AppArmor, capabilities, read-only rootfs |
| 04 | [Network Security](04-network-security.md) | Bridge, host, overlay, macvlan, микросегментация |
| 05 | [Registry Security](05-registry-security.md) | Harbor, подпись (Cosign/Notary), RBAC, репликация |
| 06 | [Orchestration Security](06-orchestration-security.md) | Docker Compose, Swarm, secrets, healthcheck |
| 07 | [Daemon Security](07-daemon-security.md) | Docker socket, TLS, rootless mode, daemon.json |
| 08 | [Host Security](08-host-security.md) | Container escape, CIS Benchmark, live patching |
| 09 | [Monitoring & Logging](09-monitoring-logging.md) | Prometheus, Grafana, Falco, ELK, Docker Events |
| 10 | [Overview](10-docker-security-overview.md) | Пирамида безопасности, Shift Left, Defense in Depth, Zero Trust |

## Ключевые темы

- **Изоляция**: namespace, cgroup, seccomp, AppArmor, capabilities
- **Supply Chain**: сканирование образов, подпись, SBOM, SLSA
- **Runtime**: container escape, privileged mode, read-only rootfs
- **Сеть**: bridge vs overlay vs macvlan, network policies, mTLS
- **Registry**: Harbor, Cosign, RBAC, репликация
- **Оркестрация**: Swarm, Compose, secrets management, healthcheck
- **Мониторинг**: Falco, Prometheus, ELK, Docker Events, alerting
- **Архитектура**: CIS Benchmark, Shift Left, Zero Trust, Defense in Depth

## Целевая аудитория

- DevOps-инженеры
- Security Architects
- Platform Engineers
- Специалисты по безопасности контейнеров

## Уровень

Intermediate — Advanced. Предполагается знакомство с Docker и Linux.
