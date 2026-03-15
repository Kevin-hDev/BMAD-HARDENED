# Platform macOS — Security Patterns
> Tags: macos, desktop, os
> Load when: développement ou audit d'une app desktop macOS (Tauri, Electron, natif Swift/ObjC)

## CVEs Critiques macOS

| CVE | CVSS | Impact | Fix |
|-----|------|--------|-----|
| CVE-2025-43530 | — | VoiceOver TCC bypass → micro/cam/docs silencieux, PoC public | Patch requis |
| CVE-2025-24204 | 9.8 | gcore → lecture mémoire tout process même SIP → dump Keychain master key | Patch requis |
| CVE-2025-8672/9190/53811/53813 | — | Python bundlé ou Electron RunAsNode → hérite TCC parent → RCE | Critique Electron/Tauri |
| CVE-2025-8597/8700 | — | `get-task-allow` en prod → tout process local injecte + hérite TCC | NE JAMAIS shipper |
| CVE-2025-24118 | 9.8 | XNU race condition credentials (zalloc_ro_mut) → privesc kernel | Patch requis |
| CVE-2025-43529 | — | WebKit UAF → RCE arbitraire, attaques "extremely sophisticated" (Google TAG) | Patch requis |
| CVE-2025-14174 | — | WebKit/ANGLE memory corruption → impacte Electron ET Tauri simultanément | Patch requis |
| CVE-2026-20700 | — | dyld memory corruption 0-day (fév 2026), premier 0-day 2026, toutes apps macOS | Patch requis |
| CVE-2024-44243 | — | storagekitd SIP bypass → rootkit/TCC bypass | Patch déc 2024 |
| CVE-2024-54498 | 8.8 | sharedfilelistd path traversal → sandbox escape, PoC public | Patch requis |
| CVE-2025-31191 | — | Security-scoped bookmarks → accès fichier arbitraire depuis toute app sandboxée | Patch requis |
| CVE-2025-24277 | — | osanalyticshelperd crash report → symlink race → fichiers root | Patch requis |
| CVE-2025-10015/10016 | — | Sparkle : pas de validation client XPC → trigger update arbitraire → LPE root | Patch requis |
| CVE-2025-62686 | — | Dylib hijacking apps sans Hardened Runtime | Patch requis |

## Failles Spécifiques macOS

### TCC Bypass
TCC (Transparency Consent Control) = garde les accès micro, cam, contacts, docs.
CVE-2025-43530 : `SecStaticCodeCreateWithPath` (path-based) au lieu d'audit token → TOCTOU → code arbitraire dans binaires Apple-signés → accès silencieux.

**Principe fondamental :** bypass une restriction SIP = bypass toutes les autres.

### SIP Bypass
CVE-2024-44243 : `storagekitd` avec entitlement `com.apple.rootless.install.heritable` → drop bundle dans `/Library/Filesystems` → binaires custom sans SIP.

### Dylib Hijacking — 4 vecteurs
```
1. DYLD_INSERT_LIBRARIES : bloqué par SIP + Hardened Runtime
   CVE-2025-62686 : apps SANS hardened runtime encore vulnérables

2. @rpath hijacking : plusieurs LC_RPATH → dylib absente chemin primaire
   → attaquant plante dans chemin antérieur

3. LC_LOAD_WEAK_DYLIB : dylibs optionnelles → attaquant fournit version malveillante

4. DYLD_FALLBACK_LIBRARY_PATH : process non restreint → $HOME/lib, /usr/local/lib
```
Fix : Hardened Runtime (`codesign --options runtime`), Library Validation, segment `__RESTRICT`, sanitiser variables DYLD_* au démarrage.

### Entitlements Hardened Runtime — Risques
| Entitlement | Risque |
|-------------|--------|
| `cs.allow-jit` | Requis V8/WebAssembly — augmente surface d'attaque |
| `cs.disable-library-validation` | HIGH — active dylib hijacking |
| `cs.debugger` | TRÈS HIGH — lecture/écriture mémoire autre process |
| `get-task-allow` | JAMAIS en production — tout process local injecte |

Helpers : fichiers d'entitlements séparés obligatoires (principe moindre privilège).

### XPC Exploitation — Sandbox Escapes
CVE-2024-54498 : path traversal dans `sharedfilelistd` → escape + sandbox token récupéré.
CVE-2025-31191 : supprimer ACL `com.apple.scopedbookmarksagent.xpc` → forger bookmarks → accès arbitraire.
Pattern chercheur jhftss : 10+ escapes via XPC services sans validation client.

```swift
// CORRIGÉ : toujours valider le client XPC par audit token
// (PID-based validation = TOCTOU vulnérable)
func listener(_ listener: NSXPCListener,
              shouldAcceptNewConnection conn: NSXPCConnection) -> Bool {
    // Vérifier audit token via SecCodeCopyGuestWithAttributes
    // Pas de vérification PID seul
}
```

### Keychain — Accès et Master Key
CVE-2025-24204 : `gcore` avec entitlement `com.apple.system-task-ports.read` → dump `securityd` → master Keychain key.

```swift
// VULNÉRABLE : accès non restreint
SecItemAdd(["account": "token", "service": "app"] as CFDictionary, nil)

// CORRIGÉ : ACL explicite + biométrie
let access = SecAccessControlCreateWithFlags(nil,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    .biometryCurrentSet, nil)
```

### Gatekeeper / Notarization / XProtect
- Notarization obligatoire pour distribution hors App Store (macOS 10.15+)
- XProtect : signatures malware automatiques (silencieux, non désactivable)
- Gatekeeper vérifie signature + notarisation au premier lancement

### Auto-Update Sparkle
CVE-2025-10015/10016 : pas de validation client XPC → trigger update avec bundle arbitraire → LPE root.
Fix : Sparkle version patchée, valider le client XPC update.

### Tauri / Electron — Spécificités macOS
- CVE-2025-14174 (WebKit/ANGLE) : impacte les deux frameworks simultanément
- Electron RunAsNode (CVE-2025-8672/9190) : désactiver `ELECTRON_RUN_AS_NODE` fuse en prod
- Tauri : Hardened Runtime activé par défaut dans la config de build Tauri v2

## Hardening Checklist macOS

- [ ] Hardened Runtime activé : `codesign --options runtime` — OBLIGATOIRE
- [ ] Library Validation activée — interdire dylibs non signées par Apple/développeur
- [ ] Entitlement `get-task-allow` absent du build de production (absolument)
- [ ] `cs.disable-library-validation` : éviter si possible — justifier si nécessaire
- [ ] Notarisation Apple obligatoire pour toute distribution externe
- [ ] App Sandbox activée (seatbelt) — entitlements au minimum nécessaire
- [ ] XPC services : valider client par audit token (pas PID)
- [ ] Keychain : `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` + biométrie pour secrets critiques
- [ ] Electron : désactiver fuse RunAsNode en prod, màj vers version sans CVE-2025-8672
- [ ] Variables DYLD_* sanitisées au démarrage si Hardened Runtime non actif
- [ ] Helpers : entitlements séparés, principle of least privilege par helper
- [ ] Sparkle : version patchée (CVE-2025-10015/10016) + validation XPC client
- [ ] Tauri updater : Ed25519 Minisign activé (défaut Tauri v2)
- [ ] macOS à jour — CVE-2026-20700 (dyld) impacte toutes les apps
