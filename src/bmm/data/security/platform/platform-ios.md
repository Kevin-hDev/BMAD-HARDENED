# Platform iOS — Security Patterns
> Tags: ios, mobile
> Load when: développement ou audit d'une app iOS (Swift, React Native, Flutter, Capacitor)

## CVEs Critiques iOS — Zero-Days Exploités In-the-Wild

| CVE | CVSS | Composant | Type | Patch |
|-----|------|-----------|------|-------|
| CVE-2025-31200 | CRITICAL | CoreAudio (AAC) | Zero-click RCE via iMessage (Spyware, Apple SEAR + Google TAG) | iOS 18.4.1 |
| CVE-2025-31201 | CRITICAL | RPAC | Bypass Pointer Auth → kernel (chaîné avec -31200) | iOS 18.4.1 |
| CVE-2025-43200 | 9.8 | Messages/iCloud | Logic flaw → zero-click (Paragon Graphite) | iOS 18.3.1 |
| CVE-2025-43300 | 8.8 | ImageIO (DNG) | OOB write via WhatsApp (Spyware) | iOS 18.6.2 |
| CVE-2025-43529 | 8.8 | WebKit | Use-after-free → RCE (Google TAG) | iOS 26.2 |
| CVE-2025-14174 | 8.8 | WebKit/ANGLE | Memory corruption (chaîné avec -43529) | iOS 26.2 |
| CVE-2025-24201 | 8.8 | WebKit | OOB write → sandbox escape | iOS 18.3.2 |
| CVE-2025-24200 | HIGH | USB Restricted Mode | Bypass autorisation (Cellebrite/GrayKey) | iOS 18.3.1 |
| CVE-2025-24085 | HIGH | CoreMedia | Use-after-free → EoP (Spyware ~233$ dark web) | iOS 18.3 |
| CVE-2025-24204 | — | gcore (Keychain) | Entitlement bug → lecture Keystore | — |

**Chaîne la plus grave :** CVE-2025-31200 + CVE-2025-31201 — heap corruption décodeur AAC via iMessage (contourne BlastDoor) + bypass PAC → escalade kernel.

**Tendance :** bibliothèques de traitement d'images (ImageIO) via WhatsApp/iMessage zero-click = nouveau vecteur principal.

## Failles Spécifiques iOS

### Keychain — Niveaux d'Accès

| Niveau | Accessible | Sauvegardes | Sécurité |
|--------|-----------|-------------|---------|
| `kSecAttrAccessibleAlways` | Toujours | Oui | DÉPRÉCIÉ — CRITIQUE |
| `kSecAttrAccessibleWhenUnlocked` | Déverrouillé | Oui | Modérée |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Déverrouillé | Non | Haute |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | Déverrouillé + code | Non | Maximale |

```swift
// VULNÉRABLE : accessible même appareil verrouillé, inclus dans sauvegardes
kSecAttrAccessible as String: kSecAttrAccessibleAlways

// CORRIGÉ : biométrie + non-transférable
var error: Unmanaged<CFError>?
let access = SecAccessControlCreateWithFlags(
    kCFAllocatorDefault,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    [.biometryCurrentSet, .or, .devicePasscode], &error)
// Utiliser [kSecAttrAccessControl: access] dans SecItemAdd
```

**UserDefaults = plist non chiffré** — jamais de JWT, token, clé, PII.

### Data Protection — Fichiers
```swift
// NSFileProtectionComplete : fichiers inaccessibles 10s après verrouillage
description?.setOption(FileProtectionType.complete as NSObject,
                       forKey: NSPersistentStoreFileProtectionKey)
```

### ATS — App Transport Security
63% des apps le désactivent → faille MITM directe.
```xml
<!-- VULNÉRABLE -->
<key>NSAllowsArbitraryLoads</key><true/>

<!-- CORRIGÉ : exception ciblée uniquement pour hôte legacy -->
<key>NSAllowsArbitraryLoads</key><false/>
<!-- + NSExceptionDomains pour l'hôte legacy uniquement -->
```

### Certificate Pinning SPKI (survit aux rotations de cert)
```swift
// Vérifier sha256 de la clé publique — comparer à un Set<String> hardcodé
// Fail CLOSED : completionHandler(.cancelAuthenticationChallenge, nil)

// Tools de bypass connus : Frida/objection hookent SecTrustEvaluateWithError
// Détection Frida : port 27042, _dyld_image_count() → "FridaGadget", "MobileSubstrate"
```

### Jailbreak Detection
```swift
// Indicateurs à vérifier :
// - /Applications/Cydia.app, /usr/sbin/sshd, /etc/apt
// - Écriture hors sandbox : FileManager.default.fileExists("/private/jailbreak")
// - Symboles Frida dans dyld : "FridaGadget", "frida"
// - Forks de process : system(), popen() — retournent -1 sur device non-rooté
// Bypass connu : Frida peut patcher ces checks en 11 lignes (Flutter inclus)
```

### Erreurs CryptoKit Critiques
```swift
// VULNÉRABLES :
// Mode ECB → ciphertext identique pour blocs identiques
// IV statique en GCM → perte d'authenticité totale
// Clé dérivée d'un string simple → attaque dictionnaire

// CORRIGÉS :
// - CryptoKit (iOS 13+) : AEAD par défaut — préférer à CommonCrypto
// - PBKDF2 ≥ 100K rounds pour dérivation depuis mot de passe
// - GCM : IV aléatoire CSPRNG pour chaque chiffrement
import CryptoKit
let key = SymmetricKey(size: .bits256)  // CSPRNG
let nonce = AES.GCM.Nonce()             // Aléatoire
let sealed = try AES.GCM.seal(data, using: key, nonce: nonce)
```

### OAuth — Vol via Schéma Custom
```swift
// VULNÉRABLE : schéma custom → n'importe quelle app peut le réclamer
callbackURLScheme: "myapp"

// CORRIGÉ : Universal Links (iOS 17.4+)
callback: .https(host: "example.com", path: "/oauth/callback")
session.prefersEphemeralWebBrowserSession = true  // Pas de partage cookies Safari
```

### Method Swizzling / Runtime Attacks
Frida et Cycript peuvent swizzler toute méthode Objective-C à runtime.
Swift dynamique (`@objc dynamic`) vulnérable.
Fix : éviter `@objc dynamic` sur méthodes de sécurité critiques, détecter Frida.

## Hardening Checklist iOS

- [ ] Keychain : `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` + biométrie pour tokens/clés
- [ ] Jamais de secret dans UserDefaults, NSLog, fichiers sans Data Protection
- [ ] NSFileProtectionComplete sur tous les fichiers sensibles
- [ ] ATS activé (`NSAllowsArbitraryLoads = false`), exceptions ciblées uniquement
- [ ] Certificate pinning SPKI sur toutes les connexions API (≥2 pins : actif + backup)
- [ ] iOS minimum ≥ iOS 17 pour éviter vulnérabilités corrigées (USB Restricted Mode, etc.)
- [ ] CryptoKit pour tout nouveau code crypto — CommonCrypto uniquement si legacy
- [ ] Universal Links pour OAuth callbacks (pas de schémas custom)
- [ ] `prefersEphemeralWebBrowserSession = true` sur ASWebAuthenticationSession
- [ ] Détection jailbreak multi-indicateurs (pas un seul check)
- [ ] Détection Frida : port 27042, noms dyld (`FridaGadget`, `MobileSubstrate`)
- [ ] Méthodes critiques de sécurité : ne pas exposer en `@objc dynamic`
- [ ] ImageIO : vérifier et à jour — CVE-2025-43300 via images DNG
- [ ] WebKit (WKWebView) : màj OS obligatoire — CVE-2025-43529/14174
