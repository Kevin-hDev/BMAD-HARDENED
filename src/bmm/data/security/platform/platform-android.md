# Platform Android — Security Patterns
> Tags: android, mobile
> Load when: développement ou audit d'une app Android (Kotlin, React Native, Flutter, Capacitor)

## CVEs Critiques Android

| CVE | CVSS | Composant | Type | Impact |
|-----|------|-----------|------|--------|
| CVE-2025-48593 | CRITICAL | Android System (13-16) | Zero-click RCE | Sans interaction utilisateur |
| CVE-2025-21042 | — | Samsung libimagecodec.quram.so | OOB via DNG | LANDFALL spyware (Irak, Iran, Turquie) |
| CVE-2025-21043 | — | Samsung libimagecodec.quram.so | OOB via DNG | Chaîné avec -21042 |
| CVE-2024-53104 | — | Kernel driver UVC | EoP via USB | Chaîne Cellebrite Serbie |
| CVE-2024-53197 | — | Kernel driver ALSA USB | EoP via USB | Chaîne Cellebrite Serbie |
| CVE-2024-50302 | — | Kernel driver HID | EoP via USB | Chaîne Cellebrite Serbie |
| CVE-2025-48572 | 7.4 | Framework | EoP | Activement exploité déc 2025 |
| CVE-2025-48633 | — | Framework | Info disclosure | Activement exploité |
| CVE-2026-21385 | 7.8 | Qualcomm Graphics (235 chipsets) | Integer overflow | CISA KEV 3 mars 2026 |
| CVE-2022-20138 | — | PendingIntent | Hijacking via fillIn() | Voir code ci-dessous |

**Spyware LANDFALL :** CVE-2025-21042/21043 — chaîne Samsung via fichiers DNG malformés.
**Chaîne USB Cellebrite (Serbie) :** 3 CVEs kernel drivers enchaînés → root sans déverrouillage.

## Failles Spécifiques Android

### Keystore — StrongBox vs TEE
```kotlin
// TEE = tous appareils modernes (Trusted Execution Environment)
// StrongBox = Titan M, Android 9+, 35-55× plus lent mais résistant attaques physiques
val hasStrongBox = context.packageManager
    .hasSystemFeature(PackageManager.FEATURE_STRONGBOX_KEYSTORE)

// Vérification niveau sécurité (API 31+) — isInsideSecureHardware() déprécié
val keyInfo = factory.getKeySpec(secretKey, KeyInfo::class.java)
// keyInfo.securityLevel → SOFTWARE / TEE / STRONGBOX
```

### SharedPreferences — Stockage Non Chiffré
```kotlin
// VULNÉRABLE : lisible sur appareil rooté / backup non chiffré
context.getSharedPreferences("prefs", MODE_PRIVATE).edit()
    .putString("auth_token", token).apply()

// CORRIGÉ : EncryptedSharedPreferences (AES256-SIV clés + AES256-GCM valeurs)
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build()
val securePrefs = EncryptedSharedPreferences.create(
    context, "secure_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM)

// Note : androidx.security dépréciée avr 2025 → DataStore + Tink pour nouveaux projets
// CRITIQUE : exclure des sauvegardes Auto Backup — MasterKey ne survit pas à une restauration
```

### TrustManager — MITM Total
```kotlin
// VULNÉRABLE — accepte tout certificat — rejet Google Play
object : X509TrustManager {
    override fun checkServerTrusted(chain: Array<X509Certificate>, type: String) {}
    override fun checkClientTrusted(chain: Array<X509Certificate>, type: String) {}
    override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
}

// CORRIGÉ : TrustManager plateforme + pinning OkHttp (≥2 pins obligatoires)
val pinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/HASH_PRINCIPAL=")
    .add("api.example.com", "sha256/HASH_BACKUP=")  // backup obligatoire
    .build()
val client = OkHttpClient.Builder().certificatePinner(pinner).build()
```

### Network Security Config — Pinning Déclaratif
```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
  <base-config cleartextTrafficPermitted="false"/>
  <domain-config>
    <domain includeSubdomains="true">api.example.com</domain>
    <pin-set expiration="2027-01-01">
      <pin digest="SHA-256">HASH_PRINCIPAL=</pin>
      <pin digest="SHA-256">HASH_BACKUP=</pin>
    </pin-set>
  </domain-config>
</network-security-config>
<!-- AndroidManifest : android:networkSecurityConfig="@xml/network_security_config" -->
```

### Intent Hijacking — Activités Exportées
```xml
<!-- VULNÉRABLE : activité exportée sans permission -->
<activity android:name=".TransferActivity" android:exported="true"/>
<!-- Attaque : adb shell am start -n com.bank/.TransferActivity --es recipient "attacker" -->

<!-- CORRIGÉ : permission signature -->
<permission android:name="com.bank.permission.TRANSFER"
            android:protectionLevel="signature"/>
<activity android:name=".TransferActivity"
          android:permission="com.bank.permission.TRANSFER"/>
```

### PendingIntent Mutable (CVE-2022-20138)
```kotlin
// VULNÉRABLE : intent vide + FLAG_MUTABLE → hijacking via fillIn()
PendingIntent.getActivity(context, 0, Intent(), PendingIntent.FLAG_MUTABLE)

// CORRIGÉ : intent explicite + FLAG_IMMUTABLE + FLAG_ONE_SHOT
val intent = Intent(context, MainActivity::class.java).apply {
    action = "com.bank.VIEW"
    setPackage(context.packageName)  // Explicite = pas de hijacking
}
PendingIntent.getActivity(context, 0, intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_ONE_SHOT)
```

### Content Provider — SQLi
```kotlin
// VULNÉRABLE : concaténation dans query
cursor = db.rawQuery("SELECT * FROM users WHERE id = " + userId, null)

// CORRIGÉ : paramètres séparés
cursor = db.query("users", null, "id = ?", arrayOf(userId), null, null, null)
// + ContentProvider avec permission readPermission/writePermission explicites
```

### Root Detection — Play Integrity
```kotlin
// SafetyNet entièrement déprécié mai 2025 → Play Integrity API
// MEETS_STRONG_INTEGRITY = Android 13+, attestation hardware, màj sécu < 12 mois

// Bypass connu : Magisk "Play Integrity Fix"
// Android 13+ attestation hardware le rend difficile mais pas impossible

// Indicateurs root complémentaires :
// - /system/app/Superuser.apk, /sbin/su, /system/bin/su
// - Packages : com.topjohnwu.magisk, com.noshufou.android.su
// - Build.TAGS == "test-keys"
// - Zygisk : détectable via /proc/maps → "zygisk"
```

### Backup — Exclusion Données Sensibles
Android 12+ : `data-extraction-rules` dans le manifest pour exclure `sharedpref` et `database` sensibles de cloud-backup et device-transfer.
CRITIQUE : MasterKey EncryptedSharedPreferences ne survit pas à une restauration backup — exclure obligatoirement.

## Hardening Checklist Android

- [ ] Keystore hardware-backed (TEE minimum, StrongBox pour tokens critiques)
- [ ] EncryptedSharedPreferences ou DataStore+Tink — jamais SharedPreferences pour secrets
- [ ] TrustManager plateforme uniquement — jamais de TrustManager permissif
- [ ] Certificate pinning OkHttp + Network Security Config (≥2 pins : actif + backup)
- [ ] `cleartextTrafficPermitted="false"` dans Network Security Config
- [ ] Toutes les activités/services/receivers : `android:exported` explicite
- [ ] Activities sensibles : `android:permission` avec `protectionLevel="signature"`
- [ ] PendingIntents : FLAG_IMMUTABLE + intent explicite (package + action définis)
- [ ] Content Providers : permissions `readPermission`/`writePermission`, paramètres SQL séparés
- [ ] Données sensibles exclues de Auto Backup (data-extraction-rules)
- [ ] Play Integrity API (remplace SafetyNet déprécié mai 2025) — vérifier côté serveur
- [ ] `android:debuggable="false"` en production (vérifié par Play Store)
- [ ] ProGuard/R8 activé — obfuscation du bytecode
- [ ] Android minimum API 26+ (Android 8+) pour éviter vulnérabilités historiques
- [ ] Màj kernel Android prioritaire — CVE-2026-21385 Qualcomm (235 chipsets, CISA KEV)
