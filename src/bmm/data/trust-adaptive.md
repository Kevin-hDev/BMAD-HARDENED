# Trust-Adaptive Security Gates — BMAD v2

> Détection automatique du niveau de risque d'une story.
> Chaque niveau déclenche un niveau de review sécurité différent.

---

## Détection automatique du risque

Scanner les fichiers de la story (titre, AC, tâches, chemins de fichiers) pour ces patterns :

### Niveau HIGH (non-bypassable)

**Patterns dans le code/noms :**
`auth`, `login`, `logout`, `password`, `token`, `secret`, `api_key`, `apikey`,
`encrypt`, `decrypt`, `cipher`, `hash`, `hmac`, `sign`, `verify`,
`payment`, `billing`, `invoice`, `stripe`, `pii`, `gdpr`, `rgpd`,
`session`, `cookie`, `jwt`, `oauth`, `pkce`, `certificate`, `cert`, `tls`

**Répertoires :**
`auth/`, `security/`, `crypto/`, `payments/`, `secrets/`, `certificates/`, `keys/`

### Niveau MEDIUM

**Patterns dans le code/noms :**
`api`, `endpoint`, `route`, `handler`, `middleware`, `interceptor`,
`config`, `env`, `environment`, `database`, `query`, `orm`, `migration`,
`network`, `socket`, `http`, `fetch`, `request`, `response`

**Répertoires :**
`api/`, `middleware/`, `config/`, `database/`, `db/`, `routes/`

### Niveau LOW (défaut)

Tout ce qui ne correspond à aucun pattern HIGH ou MEDIUM.
UI pure, utilitaires, formatage, helpers sans accès réseau ou données sensibles.

---

## Review par niveau

| Niveau | Action sécurité | Désactivable ? | Coût tokens estimé |
|--------|----------------|----------------|--------------------|
| LOW | 10 réflexes rapides (checklist inline) | OUI (flag `-e`) | ~50 tokens |
| MEDIUM | INDEX_THEN_SELECTIVE (2-3 DATA files) + mini-STRIDE | NON | ~200 tokens |
| HIGH | INDEX_THEN_SELECTIVE (3-5 DATA files) + full STRIDE + Five-Persona | NON | ~500 tokens |

---

## Comportement par niveau

### LOW — Checklist rapide

Vérifier les 10 réflexes (security-rules.md) sous forme de checklist en fin de review.
Ne pas charger de DATA files sécurité sauf si un item de la checklist est rouge.

```
[ ] Pas de == sur secrets
[ ] Collections bornées
[ ] Pas de secret en dur
[ ] Entrées validées
[ ] Pas d'info interne dans les erreurs
[ ] CSPRNG si token/nonce
[ ] Pas de concaténation en commande système
[ ] Catch non vide
[ ] Pas de log de données sensibles
[ ] Zéroisation des buffers
```

### MEDIUM — INDEX_THEN_SELECTIVE adapté

1. Charger `security/index.md`
2. Sélectionner 2-3 fichiers selon les tags de la story
3. Exécuter un mini-STRIDE (Spoofing, Tampering, Information Disclosure uniquement)
4. Reporter les findings dans la section `## Security` de la story

### HIGH — Full review

1. Charger `security/index.md`
2. Sélectionner 3-5 fichiers selon les tags de la story
3. Exécuter full STRIDE (6 catégories)
4. Activer Five-Persona review (attaquant, défenseur, auditeur, utilisateur, opérateur)
5. Générer un threat model minimal en annexe de la story

---

## Impact sur Quick Flow (L1)

Le Quick Flow chargeait ZÉRO sécurité. Avec Trust-Adaptive :

| Quick Flow + niveau | Comportement |
|--------------------|--------------|
| L1 + LOW | 10 réflexes rapides inline (~50 tokens) |
| L1 + MEDIUM | INDEX_THEN_SELECTIVE adapté (~200 tokens) |
| L1 + HIGH | **Escalade automatique vers Full Flow** — pas de Quick Flow possible |

L'escalade L1→Full Flow est automatique et non-bypassable.
L'agent DOIT notifier : `[TRUST-ADAPTIVE: escalade L1→Full Flow, raison: HIGH détecté sur {pattern}]`

---

## Le flag `-e` (économie) NE désactive JAMAIS la sécurité

`-e` réduit la cérémonie (moins de docs, moins d'étapes formelles).
Il ne réduit PAS les checks de sécurité.

| Flag | Effet sur sécurité |
|------|--------------------|
| `-e` (économie) | AUCUN — tous les checks restent actifs |
| `-e` + LOW | Checklist inline toujours exécutée |
| `-e` + MEDIUM | INDEX_THEN_SELECTIVE toujours exécuté |
| `-e` + HIGH | Escalade Full Flow toujours déclenchée |

---

## Algorithme de détection (pseudo-code)

```
function detect_trust_level(story):
  content = story.title + story.ac + story.tasks + story.file_paths

  for pattern in HIGH_PATTERNS:
    if pattern in content.lower():
      return HIGH

  for pattern in MEDIUM_PATTERNS:
    if pattern in content.lower():
      return MEDIUM

  return LOW
```
