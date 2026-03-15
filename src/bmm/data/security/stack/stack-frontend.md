# Stack Frontend SPA — Security Patterns
> Tags: vue, nuxt, angular, svelte, sveltekit, spa, frontend, vite
> Load when: project uses Vue/Angular/Svelte (detected via vue.config.js, angular.json, svelte.config.js)

## CVEs Critiques

| CVE | CVSS | Framework | Impact | Fix |
|-----|------|-----------|--------|-----|
| CVE-2026-22610 | 8.5 | Angular compiler | XSS SVG `data:text/javascript` | ≥ 19.2.18 / 20.3.16 / 21.0.7 |
| CVE-2025-66412 | 8.5 | Angular compiler | XSS SVG animation/MathML | ≥ 19.2.17 / 20.3.15 / 21.0.2 |
| CVE-2025-59052 | 7.1 | Angular SSR | Race condition → tokens cross-requêtes | ≥ 18.2.14 / 19.2.15 |
| CVE-2025-66035 | 7.7 | Angular common | Fuite XSRF via URL `//attacker.com` | ≥ 19.2.16 / 20.3.14 |
| CVE-2024-34344 | 8.6 | Nuxt | Path Traversal / RCE | Nuxt ≥ 3.12.4 |
| CVE-2026-22803 | High | SvelteKit | DoS mémoire remote functions | kit ≥ 2.49.5 |
| CVE-2025-32388 | Medium | SvelteKit | XSS searchParams dans boot script | Patché |
| CVE-2024-45811 | Medium | Vite | Bypass server.fs.deny `?import&raw` | Vite ≥ 5.4.6 |

## XSS — HTML brut (Vue / Angular / Svelte)

```vue
<!-- Vue VULN --> <div v-html="userComment"></div>
<!-- Vue FIX  --> <div v-html="sanitized"></div>
<!-- sanitized = computed(() => DOMPurify.sanitize(userComment.value)) -->
```
```svelte
<!-- Svelte VULN --> <div>{@html userComment}</div>
<!-- Svelte FIX  --> <div>{@html DOMPurify.sanitize(userComment)}</div>
<!-- Linter : eslint svelte/no-at-html-tags -->
```
```typescript
// Angular VULN : bypassSecurityTrustHtml(userInput) sur input externe = XSS
// Angular FIX  : sanitizer.sanitize(SecurityContext.HTML, userInput) ou DOMPurify
```

## Injection de template → RCE navigateur

```javascript
// Vue VULN — template depuis API
const DynamicComponent = { template: templateFromApi }
// Payload : {{ constructor.constructor('alert(document.cookie)')() }}
// FIX : build runtime-only, jamais runtimeCompiler: true
```
```typescript
// Angular VULN — JIT avec template utilisateur
Component({ template: userTemplate })(class {})
// FIX : compilation AOT (défaut CLI) — jamais concaténer input dans template
```

## Contamination cross-requêtes SSR

```typescript
// Nuxt VULN — ref() module = partagé entre toutes les requêtes
const cartItems = ref<CartItem[]>([])
// Nuxt FIX
const cartItems = useState<CartItem[]>('cart-items', () => [])
// Pinia FIX : sessionToken: skipHydrate(sessionToken) — évite sérialisation __NUXT_DATA__

// Svelte VULN — singleton serveur
let cachedUser: User | null = null;
// Svelte FIX — fresh par requête, Context API (setContext/getContext)
export async function load({ fetch }) { return { user: await (await fetch('/api/user')).json() }; }
```
Angular CVE-2025-59052 : injecteur global → tokens cross-requêtes — services scopés par requête.

## Authentification SPA

| Mécanisme | XSS lisible | Recommandation |
|-----------|:-----------:|----------------|
| localStorage / sessionStorage | OUI | EVITER |
| Cookie httpOnly+Secure+SameSite | NON | RECOMMANDE |
| Variable mémoire JS | Non | Access token court uniquement |

**Pattern BFF** : BFF = client OAuth côté serveur, refresh token cookie httpOnly, access token (15 min) mémoire.

```javascript
// JWT algo confusion — VULN : jwt.verify(token, publicKey) // algo du header accepté
jwt.verify(token, publicKey, { algorithms: ['RS256'] }) // ✅ FIX

// OAuth PKCE — crypto.getRandomValues() uniquement, méthode S256, verifier ≥ 43 chars
```

## Secrets et supply chain

```javascript
// Vite server.fs.deny bypass (CVE-2024-45811)
// curl "@fs/secret.txt?import&raw" → contenu complet malgré deny

// Source maps prod = code source public
export default defineConfig({ build: { sourcemap: false } }) // Vite
// "sourceMap": false dans angular.json prod | webpack: devtool: false
```

| Framework | Préfixe embarqué (dangereux) |
|-----------|:-----------------------------|
| Vite | `VITE_*` |
| SvelteKit | `PUBLIC_*` |

> Incident Sprocket : credentials AWS + GitHub tokens extraits de bundles `VITE_*`.

```bash
# Supply chain Shai-Hulud (sept 2025) : chalk, debug, ansi-styles compromis
npm ci                         # lockfile strict — jamais npm install en CI
npm config set ignore-scripts true
npx socket scan                # scanner dépendances
# Polyfill.io : SRI obligatoire pour toute ressource CDN
```

## Headers de sécurité

```javascript
// CSP nonce + strict-dynamic
const nonce = crypto.randomBytes(16).toString('base64');
// script-src 'self' 'nonce-${nonce}' 'strict-dynamic'; frame-ancestors 'none'; base-uri 'self'

// Trusted Types (Angular v11+)
const policy = trustedTypes.createPolicy('myPolicy', { createHTML: (i) => DOMPurify.sanitize(i) });
element.innerHTML = policy.createHTML(userInput); // 75% vulns XSS bloquées
```

## Defense Checklist
- [ ] Vue/Svelte : DOMPurify avant tout `v-html` / `{@html}` — linter activé
- [ ] Angular : AOT uniquement, jamais `bypassSecurityTrustHtml()` sur input externe
- [ ] SSR : état isolé par requête — `useState()` Nuxt, Context API Svelte, services scopés Angular
- [ ] Tokens : cookie httpOnly+Secure+SameSite — jamais localStorage
- [ ] JWT : algo fixé explicitement côté serveur — jamais confiance au header `alg`
- [ ] Source maps : `sourcemap: false` en prod — jamais exposer
- [ ] Préfixes `VITE_*`/`PUBLIC_*` : jamais de secrets
- [ ] CDN tiers : SRI (integrity hash) obligatoire
- [ ] npm ci strict + ignore-scripts + socket scan en CI
- [ ] Vite dev : jamais `--host 0.0.0.0` hors réseau isolé
