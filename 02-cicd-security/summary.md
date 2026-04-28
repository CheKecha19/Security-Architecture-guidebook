# CI/CD Security — Выжимка знаний

## 1. SAST (Static Application Security Testing)

**Что это:** Анализ исходного кода без запуска

**Инструменты:**
| Инструмент | Языки | Особенности |
|------------|-------|-------------|
| SonarQube | Мульти | Правила quality gate |
| Semgrep | Мульти | Легковесный, кастомные правила |
| Checkmarx | Мульти | Enterprise |
| CodeQL | Мульти | GitHub-native |
| Bandit | Python | Специфичен для Python |

**Когда запускать:**
```yaml
# GitLab CI
sast:
  stage: test
  script:
    - semgrep --config=auto --json --output=report.json
  artifacts:
    reports:
      sast: report.json
```

**False positives:**
- SAST часто даёт ложные срабатывания
- Нужно настраивать suppressions
- Важен triage процесс

---

## 2. DAST (Dynamic Application Security Testing)

**Что это:** Анализ запущенного приложения

**Инструменты:**
| Инструмент | Тип | Особенности |
|------------|-----|-------------|
| OWASP ZAP | Открытый | Прокси, активное/пассивное сканирование |
| Burp Suite | Коммерческий | Professional для пентестеров |
| Acunetix | Коммерческий | Автоматизация |

**Когда запускать:**
```yaml
# После deploy в staging
dast:
  stage: security
  needs: [deploy_staging]
  script:
    - docker run -t owasp/zap2docker-stable zap-baseline.py \
        -t https://staging.pulse.ru
```

**Разница SAST vs DAST:**
| Критерий | SAST | DAST |
|----------|------|------|
| Когда | До деплоя | После деплоя |
| Что видит | Исходный код | HTTP-запросы/ответы |
| False positives | Много | Меньше |
| Покрытие | Весь код | Только выполненные пути |

---

## 3. SCA (Software Composition Analysis)

**Что это:** Анализ сторонних зависимостей

**Зачем:**
- 80% кода — сторонние библиотеки
- Log4Shell, Heartbleed — уязвимости в зависимостях

**Инструменты:**
| Инструмент | Экосистема |
|------------|-----------|
| Snyk | Мульти |
| OWASP Dependency-Check | Java, .NET, Python |
| GitHub Dependabot | GitHub-native |
| Mend (WhiteSource) | Enterprise |

**Пример:**
```bash
# OWASP Dependency-Check
./dependency-check.sh --project pulse --scan . --format JSON

# Snyk
snyk test --json > snyk-report.json
```

**SBOM (Software Bill of Materials):**
```json
{
  "bomFormat": "CycloneDX",
  "components": [
    {
      "name": "log4j-core",
      "version": "2.14.1",
      "purl": "pkg:maven/org.apache.logging.log4j/log4j-core@2.14.1",
      "vulnerabilities": ["CVE-2021-44228"]
    }
  ]
}
```

---

## 4. IAST (Interactive Application Security Testing)

**Что это:** Агент внутри приложения, анализирует во время выполнения

**Преимущества:**
- Нет false positives (видит реальное выполнение)
- Показывает точный путь от source к sink
- Минимальное влияние на производительность

**Инструменты:**
- Contrast Security
- Hdiv
- Seeker (Synopsys)

---

## 5. RASP (Runtime Application Self-Protection)

**Что это:** Защита внутри приложения во время выполнения

**Как работает:**
```java
// RASP агент перехватывает вызовы
// Пример: блокировка SQLi на уровне JDBC
Connection conn = dataSource.getConnection();
// RASP перехватывает и анализирует query
PreparedStatement stmt = conn.prepareStatement(userInput); 
// → БЛОКИРОВКА если обнаружена SQLi
```

**Преимущества:**
- Защита от zero-day (не нужна сигнатура)
- Видит контекст (различает легитимный и вредоносный input)

---

## 6. Secrets Management

**Проблема:**
```bash
# ПЛОХО:
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# git log --all --source --full-history -S 'AKIAIOSFODNN7EXAMPLE'
# → Ключ найден в истории!
```

**Решения:**
| Уровень | Инструмент |
|---------|-----------|
| Файл | .env (только dev) |
| Репозиторий | Git-crypt, SOPS |
| CI/CD | GitHub Secrets, GitLab CI/CD Variables |
| Инфраструктура | HashiCorp Vault, AWS Secrets Manager |
| Kubernetes | External Secrets Operator |

**Vault (HashiCorp):**
```bash
# Политика
path "secret/data/pulse/*" {
  capabilities = ["read"]
}

# Использование в приложении
vault kv get secret/pulse/database
```

**GitHub Actions + OIDC (лучшая практика):**
```yaml
# НЕ используем статические ключи!
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
    aws-region: us-east-1
# OIDC token — временный, автоматически rotated
```

---

## 7. Supply Chain Security

**Угрозы:**
| Угроза | Описание | Пример |
|--------|----------|--------|
| **Typosquatting** | Поддельный пакет с похожим именем | `reqeusts` вместо `requests` |
| **Dependency Confusion** | Приватный пакет в public repo | `pulse-internal` в npm |
| **Compromised Maintainer** | Учётка мейнтейнера взломана | event-stream incident |
| **Malicious Code** | Вредоносный код в легитимном пакете | ua-parser-js |

**SLSA (Supply-chain Levels for Software Artifacts):**
| Level | Требование | Что нужно |
|-------|-----------|-----------|
| 1 | Provenance | SBOM + откуда собрано |
| 2 | Signed provenance | Криптографическая подпись |
| 3 | Hardened build | Изолированная среда сборки |
| 4 | Two-person review | Все изменения ревьюятся |

**Sigstore/Cosign:**
```bash
# Подпись образа
cosign sign --key cosign.key pulse:latest

# Верификация
cosign verify --key cosign.pub pulse:latest
```

---

## 8. Pipeline Security

**Безопасность CI/CD pipeline:**

| Угроза | Защита |
|--------|--------|
| **Poisoned Pipeline** | Изоляция runner'ов, ephemeral environments |
| **Secret Leakage** | Secret scanning, pre-commit hooks |
| **Privileged Execution** | Least privilege, OIDC tokens |
| **Artifact Tampering** | Подпись артефактов, SLSA |
| **Dependency Confusion** | Private registry, lock files |

**Best Practices:**
```yaml
# .github/workflows/secure-pipeline.yml
name: Secure CI/CD

permissions:
  contents: read  # Минимальные права

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false  # Не сохранять токен
      
      - name: Run SAST
        uses: returntocorp/semgrep-action@v1
      
      - name: Run SCA
        uses: snyk/actions/node@master
      
      - name: Build and sign
        run: |
          docker build -t pulse:${{ github.sha }} .
          cosign sign --key env://COSIGN_PRIVATE_KEY pulse:${{ github.sha }}
```

---

## 9. DevSecOps

**Shift-Left Security:**
```
Традиционно:        DevSecOps:
Dev → Test → Sec    DevSec → Test → Deploy
         ↑           (security встроена)
         |
    (security здесь)
```

**Культура:**
- Security как shared responsibility
- Автоматизация > ручные проверки
- Быстрая обратная связь (fail fast)
- Обучение разработчиков (secure coding)

**Метрики:**
| Метрика | Цель |
|---------|------|
| Mean Time To Patch (MTTP) | < 24 часов для критических |
| % кода покрытого SAST | > 90% |
| % зависимостей без уязвимостей | > 95% |
| False Positive Rate | < 20% |

---

## 10. Artifact Security

**Подпись артефактов:**
```bash
# GPG подпись
gpg --detach-sign --armor artifact.jar

# Cosign (Sigstore)
cosign sign artifact.tar.gz

# Sigstore рекорд (без ключей)
cosign sign --oidc-issuer https://accounts.google.com artifact.tar.gz
```

**Верификация:**
```bash
# В CI/CD перед deploy
cosign verify --certificate-identity-regexp '.*@pulse.ru' \
  --certificate-oidc-issuer https://accounts.google.com \
  artifact.tar.gz
```

---

## Чек-лист для собеседования

- [ ] Разница SAST/DAST/SCA/IAST/RASP
- [ ] Когда что использовать
- [ ] Supply chain threats (typosquatting, dependency confusion)
- [ ] SLSA уровни
- [ ] Secrets management (Vault, OIDC)
- [ ] Pipeline hardening
- [ ] Shift-left security
- [ ] SBOM (CycloneDX/SPDX)
- [ ] Sigstore/Cosign
- [ ] DevSecOps метрики

---

_Выжимка создана на основе 15 статей Habr_
