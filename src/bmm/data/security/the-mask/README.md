# The Mask — Exploit DATA

> Ce dossier contient les données d'exploitation actualisées pour l'agent The Mask.
> Chaque fichier couvre un domaine d'attaque avec des techniques, PoCs, et chaînes
> d'exploitation à jour (2026+).

## Structure

| Fichier | Contenu | À enrichir avec |
|---------|---------|----------------|
| `atk-exploit-web.md` | SSRF, SQLi, XSS, CSRF, deserialization — PoCs concrets | Derniers CVEs, bypass WAF, techniques confs |
| `atk-exploit-desktop.md` | DLL hijacking, IPC abuse, sandbox escape, UAC/polkit/TCC | Recherches récentes Chromium, V8, Tauri |
| `atk-exploit-llm.md` | Prompt injection (directe/XPIA), jailbreaks, MCP abuse, exfil | Papers récents, Policy Puppetry, XPIA incidents |
| `atk-exploit-mobile.md` | Cert pinning bypass, deep link hijack, WebView RCE, keystore | Outils Frida/objection actuels, iOS bypass |
| `atk-exploit-chains.md` | Chaînes multi-étapes (low+low=critical), 6 kill chains complètes | Incidents réels, bug bounty reports |
| `atk-exploit-social.md` | Spear phishing, MFA bypass, pretexting, typosquatting, OAuth | Nouvelles techniques deepfake, vishing |

## Règle de chargement

The Mask charge ce dossier via INDEX_THEN_SELECTIVE :
1. Lire ce README pour identifier les fichiers pertinents
2. Charger 1-2 fichiers max selon le contexte de la discussion
3. Compléter avec une recherche web pour les techniques les plus récentes

## Comment enrichir

Kevin enrichit ces fichiers avec des recherches approfondies.
Les fichiers doivent contenir :
- Techniques actuelles (pas obsolètes)
- PoCs concrets (pas juste de la théorie)
- Chaînes d'exploitation (comment combiner des failles)
- Références (CVEs, papers, confs)
