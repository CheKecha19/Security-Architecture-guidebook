# Linux Security — Чек-лист

## Базовый уровень

- [ ] Landlock: 3 системных вызова
- [ ] AppArmor: профили, режимы enforce/complain
- [ ] SELinux: контексты, домены, MLS
- [ ] Namespaces: 8 типов
- [ ] Seccomp: BPF-фильтр
- [ ] POSIX ACL: setfacl/getfacl
- [ ] Capabilities: fine-grained privileges
- [ ] PAM: auth/account/password/session
- [ ] chroot: ограничения
- [ ] PolicyKit: разница с sudo

## Продвинутый уровень

- [ ] Landlock ABI v2/v3/v4: новые возможности
- [ ] AppArmor: flags (Cx, Cix, Pix, Pix)
- [ ] SELinux: MLS/MCS, политики targeted/strict
- [ ] Namespaces: TIME namespace, иерархия cgroup
- [ ] Seccomp: user notification, addfd
- [ ] POSIX ACL: default ACL, наследование
- [ ] Capabilities: ambient capabilities, bounding set
- [ ] PAM: stack модулей, module arguments
- [ ] chroot: pivot_root, mount namespace
- [ ] PolicyKit: pkexec, .policy файлы

## Архитектурный уровень

- [ ] Как комбинировать Landlock + seccomp + namespaces
- [ ] Когда AppArmor лучше SELinux (и наоборот)
- [ ] Security-optimized Docker image: best practices
- [ ] Defense in depth: несколько слоёв защиты
- [ ] Linux hardening: CIS Benchmarks
- [ ] Container escape: виды и защита
- [ ] Privilege escalation: CVE анализ

---

_Для самопроверки перед собеседованием_
