# Platform Linux — Security Patterns
> Tags: linux, server, os
> Load when: développement ou audit d'une app desktop/serveur Linux (Tauri, Flatpak, container)

## CVEs Critiques Linux

| CVE | CVSS | Impact | Fix |
|-----|------|--------|-----|
| CVE-2025-21756 | 7.8 | vsock UAF "Attack of the Vsock" → root, PoC public | Patch requis |
| CVE-2025-38236 | — | UNIX sockets MSG_OOB UAF (Jann Horn) → Chrome sandbox escape → kernel control (6.9+) | Patch requis |
| CVE-2025-38617 | — | Packet socket race → privesc + container escape, CAP_NET_RAW (depuis Linux 2.6.12) | Patch requis |
| CVE-2025-38352 | — | POSIX CPU timers TOCTOU, CISA KEV sept 2025 | Patch requis |
| CVE-2024-1086 | — | netfilter nf_tables UAF, CISA KEV | Patché |
| CVE-2025-31133 | — | runc race /dev/null → symlink → core_pattern → RCE hôte | runc v1.2.8+ |
| CVE-2025-52565 | — | runc /dev/console bind-mount race → écriture procfs | runc v1.2.8+ |
| CVE-2025-52881 | — | Contourne tous LSMs → core_pattern/sysrq-trigger (affecte crun/youki) | runc v1.2.8+ |
| CVE-2024-21626 | — | "Leaky Vessels" fd leak → filesystem hôte (Metasploit dispo) | Patché |
| CVE-2025-0009 | — | eBPF verifier type confusion → OOB map → root | Patch requis |
| CVE-2025-21362 | — | eBPF bypass SELinux/AppArmor | Patch requis |

**5 530 CVEs kernel en 2025** (+28% vs 2024). 7 entrées CISA KEV kernel.

## Failles Spécifiques Linux

### MAC Layers — SELinux / AppArmor / seccomp-bpf

**SELinux** (Type Enforcement) :
- Booleans dangereux : `httpd_can_network_connect`, `selinuxuser_execmod` → désactiver
- openSUSE → SELinux par défaut fév 2025

**AppArmor** (path-based) :
- Hard links contournent AppArmor (mitigé par Yama sysctl `fs.protected_hardlinks=1`)
- Pas de filtrage réseau granulaire — compléter avec seccomp

**seccomp-bpf** :
```
SECCOMP_RET_KILL  // Kill process immédiatement
SECCOMP_RET_ERRNO // Retourner EPERM
// Docker bloque ~44 syscalls : bpf, userfaultfd, keyctl...
// Limite : ne peut pas inspecter les arguments pointeur (adresses mémoire)
```

### Landlock LSM (kernel 5.13+, ABI v7 mars 2025)
Auto-sandbox sans root. Couvre : filesystem + TCP + sockets UNIX + signals.
Hérité par les processus enfants, non relaxable après activation.
**Idéal pour Tauri/Electron post-init** : appeler après chargement des ressources.
```rust
// Rust : crate landlock
let ruleset = Ruleset::create()?
    .handle_accesses(AccessFs::from_all(ABI::V7))?
    .create()?
    .add_rules(rules)?
    .restrict_self()?;
```

### io_uring — Rootkit "Curing" (ARMO, avril 2025)
61 opérations io_uring **contournent la surveillance syscall traditionnelle**.
Evade : Falco, Microsoft Defender for Linux, outils eBPF standards.
Google a désactivé io_uring sur Android.
```
// Fix : KRSI (Kernel Runtime Security Instrumentation) + Tetragon configuré spécifiquement
// Ou : désactiver io_uring si non nécessaire
sysctl -w kernel.io_uring_disabled=1
```

### eBPF Verifier Bugs
Pattern : verifier exploité → OOB map → leak kernel pointers → overwrite credentials → root.
```
# CRITIQUE en production :
sysctl -w kernel.unprivileged_bpf_disabled=1
```

### Secret Storage Linux — Faille Critique
Fallback quand aucun keyring daemon actif : **Chromium/Electron chiffrent avec mot de passe hardcodé** → équivalent pas de chiffrement.
Tout process sur le session D-Bus peut lire les secrets (pas d'isolation par app).
```
// Fix : libsecret simple API (portail Flatpak automatique)
// Détecter si keyring daemon actif, avertir l'utilisateur sinon
// Secret Service API = interface D-Bus standard (GNOME Keyring / KWallet)
```

### X11 vs Wayland
**X11** : toute app connectée peut keylogger, capturer écran, injecter input.
CVE-2025-62229/62230/62231 : UAF/overflow dans Xorg (20+ ans d'existence).
Red Hat supprime Xorg de RHEL 10 en 2025.

**Wayland** : isolation client par design, capture écran = permission explicite via xdg-desktop-portal.
```
// Tauri : WebKitGTK natif Wayland — meilleure posture par défaut
// Electron : forcer Wayland via --ozone-platform=wayland
```

### Container Escapes Classiques
- cgroup v1 `release_agent` : mitigé par cgroup-v2 (forcer cgroup-v2)
- `core_pattern` write depuis container
- Exposition socket Docker sans protection

### polkit / D-Bus
CVE-2021-3560 (PwnKit), CVE-2021-4034 : polkit pkexec local privesc.
```
// Limiter les règles polkit aux seuls process légitimes
// D-Bus : policy files explicites — interdire accès non nécessaires
```

## Hardening Checklist Linux

- [ ] SELinux ou AppArmor activé en mode enforcing (pas permissive)
- [ ] seccomp-bpf : profil restrictif sur l'app — bloquer syscalls non nécessaires
- [ ] Landlock activé post-init dans l'app (filesystem + réseau + signals)
- [ ] `kernel.unprivileged_bpf_disabled=1` en production absolument
- [ ] `kernel.io_uring_disabled=1` si io_uring non utilisé par l'app
- [ ] `fs.protected_hardlinks=1` et `fs.protected_symlinks=1` (Yama)
- [ ] cgroup-v2 uniquement — supprimer cgroup-v1 `release_agent`
- [ ] runc v1.2.8+ si containers utilisés (CVE-2025-31133/52565/52881)
- [ ] Wayland uniquement — Xorg désactivé ou non installé
- [ ] Secret Service API : détecter keyring daemon actif, jamais de fallback hardcodé
- [ ] Flatpak : pas de `filesystem=home` dans les permissions manifest
- [ ] Tauri v2 : deny-by-default IPC, permissions par fenêtre/commande
- [ ] Processus non-root pour l'app — capabilities minimales, pas de `CAP_NET_RAW`
- [ ] Kernel à jour — patch prioritaire pour toute entrée CISA KEV
