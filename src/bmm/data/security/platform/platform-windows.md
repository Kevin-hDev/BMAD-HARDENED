# Platform Windows — Security Patterns
> Tags: windows, desktop, os
> Load when: développement ou audit d'une app desktop Windows (Tauri, Electron, natif)

## CVEs Critiques Windows

| CVE | CVSS | Impact | Fix |
|-----|------|--------|-----|
| CVE-2026-21510 | 8.8 | Shell SmartScreen bypass 0-day | Patch fév 2026 |
| CVE-2026-21513 | 8.8 | MSHTML security bypass 0-day | Patch fév 2026 |
| CVE-2026-21519 | 7.8 | Desktop Window Manager EoP | Patch fév 2026 |
| CVE-2025-55234 | 9.8 | SMB auth bypass credential replay | Critique actif |
| CVE-2025-49708 | 9.9 | Graphics VM escape → SYSTEM hôte | Critique actif |
| CVE-2025-60724 | 9.8 | Graphics heap overflow RCE | Critique actif |
| CVE-2025-33073 | 8.8 | NTLM reflection → SYSTEM remote | Sans patch SMB signing |
| CVE-2025-24983 | 7.0 | Win32 Kernel UAF ciblé (ESET) | Patch mars 2025 |
| CVE-2025-62215 | 7.0 | Race condition + double-free → SYSTEM | Patch nov 2025 |
| CVE-2025-0411  | —   | 7-Zip MotW non propagé double-compression | Patché |
| CVE-2024-21412 | 8.8 | .url shortcut MotW bypass (APT Water Hydra) | Patché |
| CVE-2024-29988 | 8.8 | MotW bypass via archive zippée | Patché |
| CVE-2025-55305 | —   | Electron V8 heap snapshots non vérifiés | Fix Electron 35.7.5+ |

## Failles Spécifiques Windows

### UAC Bypass — Fodhelper
```
// VULNÉRABLE : écriture registre → auto-élévation sans prompt
HKCU\Software\Classes\ms-settings\shell\open\command = payload.exe
DelegateExecute = (vide)
// Variantes : eventvwr, computerdefaults, sdclt, CMSTP
```
Fix : UAC = "Always Notify", WDAC activé, Sysmon sur `HKCU\Software\Classes`.

### WDAC Bypass — DLL Side-Loading Electron
Legacy Teams (Electron, signé Microsoft) weaponisable : remplacer `/resources/app/` JS → Loki C2.
Tauri résistant : pas de répertoire JS remplaçable post-signature.
LOLBAS actifs : MSBuild, InstallUtil, mshta.

### AMSI Bypass (OpSec-safe actuel)
- Classique `AmsiScanBuffer` patch : détecté par ETWTi.
- **Hardware breakpoint VEH** (CrowdStrike Black Hat MEA) : breakpoint sur AmsiScanBuffer, VEH redirige vers return propre — non détecté.
- **AMSI Write Raid** (OffSec 2024-25) : zone writable dans System.Management.Automation.dll sans VirtualProtect.
- Détection : NtSetContextThread via ETW `Microsoft-Windows-Kernel-Audit-API-Calls`.

### DLL Hijacking — Ordre de recherche
```
Known DLLs → app dir → System32 → System → Windows → cwd → PATH
```
**Phantom DLL** : cible des DLLs inexistantes (plus dur à détecter).
**Side-loading** : planter une DLL malveillante dans le répertoire d'une app signée → bypass WDAC.

Outils privesc actifs :
| Outil | Statut | Prérequis |
|-------|--------|-----------|
| GodPotato | Très utilisé | SeImpersonatePrivilege |
| PrintSpoofer | Actif | SeImpersonatePrivilege |
| SigmaPotato | Actif | Support OS étendu |
| PrintNotifyPotato | Actif | Spooler désactivé OK |
| JuicyPotato | Mort Win10 1809+ | CLSID legacy |

### DPAPI — Faille domaine
Clé de backup domaine **immuable** → compromission = reconstruction domaine entier (SpecterOps juil 2025).
Credential Guard (VTL1) protège les master keys → mitigation principale.
```
// Desktop apps : utiliser CryptProtectData() avec CRYPTPROTECT_LOCAL_MACHINE
// JAMAIS stocker dans dossiers lisibles globalement
```

### Named Pipes — Pipe Squatting
```
// VULNÉRABLE : pipe créé sans DACL restrictive
CreateNamedPipe(L"\\\\.\\pipe\\myapp", ...)  // anyone peut connecter

// CORRIGÉ : DACL explicite
SECURITY_ATTRIBUTES sa = { ... }; // ACL = process user seulement
CreateNamedPipe(L"\\\\.\\pipe\\myapp", ..., &sa)
```

### COM Hijacking
Objet COM auto-élevé → rundll32 charge DLL craftée → opérations fichier privilégiées silencieuses.
Fix : surveiller modifications HKCU\Software\Classes\CLSID.

### MotW / SmartScreen
ISO/VHD/VHDX : fichiers montés n'héritent pas du MotW.
Depuis mars 2024 : EV certs ne donnent plus la réputation SmartScreen instantanément.

### AppContainer — Sandboxing
6 dimensions kernel-enforced. Pour apps Tauri : activer AppContainer si distribution Store.

## Hardening Checklist Windows

- [ ] UAC = "Always Notify" (pas "Notify only for app changes")
- [ ] WDAC activé (kernel mode, CI.dll) — AppLocker insuffisant (user mode)
- [ ] Credential Guard activé (Win11 22H2+ / Server 2025 : VTL1, LSAIso.exe)
- [ ] HVCI activé — bloque drivers non signés
- [ ] SMB signing activé partout — neutralise CVE-2025-33073 et NTLM relay
- [ ] NTLM désactivé si possible — ADCS : EPA activé, désactiver templates V1 schema
- [ ] Sysmon configuré : surveiller HKCU\Software\Classes, symbolic links, pipe creation
- [ ] DLL : vérifier que app dir ne contient pas de DLLs inattendues (Phantom DLL)
- [ ] Code signing Authenticode obligatoire — MSIX pour distribution Store
- [ ] Tauri updater : Ed25519 Minisign activé (non désactivable par défaut)
- [ ] Electron : màj vers 35.7.5+/36.8.1+ (CVE-2025-55305 ASAR integrity)
- [ ] DPAPI : `CryptProtectData()` avec `CRYPTPROTECT_LOCAL_MACHINE`, hors dossiers publics
- [ ] Named pipes : DACL explicite restrictive sur toutes les pipes créées par l'app
- [ ] WebView2 : Content Security Policy stricte, interdire navigation vers origines externes
