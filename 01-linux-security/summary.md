# Linux Security — Выжимка знаний

## 1. Landlock

**Что это:** Файловая песочница без root (kernel 5.13+)

**Системные вызовы:**
- `landlock_create_ruleset()` — создание набора правил
- `landlock_add_rule()` — добавление правила
- `landlock_restrict_self()` — применение к текущему процессу

**Ключевые концепции:**
- Работает без root (capability CAP_SYS_ADMIN не нужен)
- Наследуется fork/exec
- Бинарный формат правил (нет JSON/XML)
- **Ловушка:** дескрипторы, открытые ДО restrict, остаются доступны

**Пример:**
```c
int abi = landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION);
// abi < 0 — ядро не поддерживает Landlock

struct landlock_ruleset_attr ruleset_attr = {
    .handled_access_fs = LANDLOCK_ACCESS_FS_READ_FILE | 
                         LANDLOCK_ACCESS_FS_WRITE_FILE,
};
int ruleset_fd = landlock_create_ruleset(&ruleset_attr, sizeof(ruleset_attr), 0);

// Добавляем правило для /tmp
struct landlock_path_beneath_attr path_attr = {
    .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE | 
                      LANDLOCK_ACCESS_FS_WRITE_FILE,
    .parent_fd = open("/tmp", O_PATH | O_CLOEXEC),
};
landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &path_attr, 0);

// Применяем
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
landlock_restrict_self(ruleset_fd, 0);
```

---

## 2. AppArmor

**Что это:** Profile-based MAC (проще SELinux)

**Режимы:**
- **Enforce** — блокирует запрещённое
- **Complain** — только логирует

**Ключевые команды:**
```bash
aa-autodep /path/to/app      # Создать шаблон
aa-genprof /path/to/app      # Интерактивное создание
aa-enforce /etc/apparmor.d/profile  # Включить enforce
aa-complain profile           # Включить complain
aa-status                     # Показать статус
```

**Пример профиля:**
```
#include <tunables/global>

/usr/bin/myapp {
  #include <abstractions/base>
  
  /usr/bin/myapp r,
  /etc/myapp/** r,
  /var/log/myapp.log w,
  /tmp/** rw,
  
  network inet stream,
  
  deny /etc/passwd r,
}
```

**Преимущества:**
- Проще SELinux в настройке
- Path-based (не нужны метки)
- Профили можно смешивать enforce/complain

**Недостатки:**
- Менее гибкий чем SELinux
- Path-based подход уязвим к symlink attacks

---

## 3. SELinux

**Что это:** Label-based MAC (самый мощный)

**Контекст безопасности:**
```
user:role:type:level
# Пример:
system_u:system_r:httpd_t:s0
```

**Режимы:**
- **Enforcing** — политика применяется
- **Permissive** — только логирует
- **Disabled** — отключён

**Ключевые команды:**
```bash
getenforce                    # Показать режим
setenforce 0|1               # Переключить
sestatus                     # Подробный статус
ls -Z                        # Показать контекст
chcon -t httpd_sys_content_t file  # Изменить тип
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"  # Постоянное правило
restorecon -R /web           # Применить контексты
```

**Почему симлинк не работает:**
Контекст безопасности привязан к inode, а не пути. Симлинк имеет свой контекст.

**MLS (Multi-Level Security):**
- Для защиты информации разных уровней секретности
- Пример: Confidential < Secret < Top Secret

---

## 4. Namespaces

**8 типов (Linux 6.12.1):**

| Namespace | Изолирует | Системный вызов |
|-----------|-----------|-----------------|
| **PID** | ID процессов | `unshare --pid` |
| **NET** | Сетевые интерфейсы | `unshare --net` |
| **MNT** | Точки монтирования | `unshare --mount` |
| **IPC** | IPC-ресурсы | `unshare --ipc` |
| **UTS** | Имя хоста | `unshare --uts` |
| **USER** | UID/GID | `unshare --user` |
| **CGROUP** | cgroup root | `unshare --cgroup` |
| **TIME** | Системное время | `unshare --time` |

**USER namespace — ключ к безопасности контейнеров:**
```bash
# UID 0 внутри контейнера ≠ UID 0 на хосте
unshare --user --pid --fork --mount-proc
# Теперь внутри "root" = обычный пользователь на хосте
```

---

## 5. Seccomp

**Что это:** Фильтрация системных вызовов

**Режимы:**
- **Strict** — только read/write/exit/sigreturn
- **Filter** — BPF-фильтр (гибкий)

**Docker default profile:**
- ~44 системных вызова заблокировано
- Список: `default.json`

**Пример:**
```c
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);
seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(open), 1,
    SCMP_A1(SCMP_CMP_MASKED_EQ, O_WRONLY, O_WRONLY));
seccomp_load(ctx);
```

**Seccomp + Namespaces = полноценная песочница:**
```bash
# Максимальная изоляция
unshare --user --pid --net --mount --fork --mount-proc
# + seccomp filter
# + capabilities drop
# + read-only rootfs
```

---

## 6. POSIX ACL

**Расширение стандартных прав:**
```bash
# Стандартные права
chmod 750 file

# ACL — добавляем конкретного пользователя
setfacl -m u:alice:rwx file
setfacl -m g:developers:rw file

# Проверка
getfacl file

# Наследование (default ACL)
setfacl -d -m u:alice:rwx dir/
```

---

## 7. Capabilities

**Fine-grained privileges вместо root:**

| Capability | Заменяет |
|------------|----------|
| `CAP_NET_ADMIN` | ifconfig, iptables |
| `CAP_SYS_ADMIN` | mount, swapon |
| `CAP_SETUID` | setuid() |
| `CAP_CHOWN` | chown() |
| `CAP_DAC_OVERRIDE` | bypass DAC |

```bash
# Просмотр capabilities процесса
cat /proc/$$/status | grep Cap

# Запуск с ограниченными capabilities
capsh --drop=cap_net_admin -- -c "iptables -L"  # Не сработает
```

---

## 8. PAM (Pluggable Authentication Modules)

**Структура:**
```
/etc/pam.d/service
```

**Типы модулей:**
| Тип | Описание |
|-----|----------|
| `auth` | Аутентификация |
| `account` | Авторизация (проверка доступа) |
| `password` | Изменение пароля |
| `session` | Сессия (логирование, переменные) |

**Пример:**
```
# /etc/pam.d/sshd
auth       required     pam_krb5.so
auth       required     pam_google_authenticator.so
account    required     pam_unix.so
password   required     pam_cracklib.so retry=3
session    required     pam_limits.so
```

---

## 9. chroot

**Ограничения chroot (почему не песочница):**
- Требует root для вызова
- Можно обойти из-под root (chroot escape)
- Не изолирует процессы, сеть, IPC

**Правильный подход:**
```bash
# НЕ chroot
chroot /jail /bin/bash

# ДА: namespaces + seccomp
docker run --rm -it --security-opt seccomp=profile.json \
  --cap-drop ALL --read-only alpine sh
```

---

## 10. PolicyKit

**Разница с sudo:**
- **sudo** — даёт root на всё
- **PolicyKit** — даёт права на конкретное действие

```bash
# Проверить действие
pkcheck --action-id org.freedesktop.NetworkManager.enable --process $$ -u

# Правило
# /usr/share/polkit-1/actions/com.example.myapp.policy
```

---

## Чек-лист для собеседования

- [ ] Landlock: 3 системных вызова, не требует root
- [ ] AppArmor: профили, enforce/complain, aa-genprof
- [ ] SELinux: контексты, домены, MLS, почему симлинк не работает
- [ ] Namespaces: 8 типов, USER namespace = безопасность
- [ ] Seccomp: BPF-фильтр, Docker default profile
- [ ] POSIX ACL: setfacl/getfacl, default ACL
- [ ] Capabilities: fine-grained privileges
- [ ] PAM: auth/account/password/session
- [ ] chroot: ограничения, почему не песочница
- [ ] PolicyKit: разница с sudo

---

_Выжимка создана на основе 15 статей Habr_
