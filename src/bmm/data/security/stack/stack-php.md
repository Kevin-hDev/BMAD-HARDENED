# Stack PHP/Laravel — Security Patterns
> Tags: php, laravel, livewire
> Load when: project uses PHP (detected via composer.json, *.php, artisan)

## CVEs Critiques

| CVE | CVSS | Framework | Impact | Fix |
|-----|------|-----------|--------|-----|
| CVE-2025-54068 | Critical | Laravel Livewire | RCE sans auth via hydratation | Livewire 3.6.4+ |
| CVE-2024-47823 | 9.8 | Laravel Livewire | RCE upload MIME bypass | Livewire 3.5.2+ |
| CVE-2024-21546 | 9.8 | Laravel Livewire | Bypass `filename.php.` | Patché |
| CVE-2024-52301 | 8.7 | Laravel | Env manipulation via query string → debug mode | Laravel 11.31.0+ |
| CVE-2025-52662 | Medium | Nuxt DevTools | XSS → RCE WebSocket (hors PHP, adjacent) | DevTools ≥ 2.6.4 |

## Failles Spécifiques Laravel

### APP_KEY exposée — 260 000+ sur GitHub → RCE triviale
```php
// 260 000+ APP_KEY trouvées sur GitHub → PHPGGC (20+ chaînes Laravel/RCE1–RCE20)
decrypt($value, false);  // ✅ JAMAIS omettre false — désactive désérialisation auto
// JAMAIS commiter .env dans Git (ajouter .env au .gitignore dès init)
```

### Mass Assignment + SQLi Raw
```php
// ❌ VULN — is_admin=1 passé via formulaire
protected $guarded = [];
User::create($request->all());
DB::table('users')->insert($request->all()); // contourne Eloquent

// ✅ FIX
protected $fillable = ['name', 'email', 'password'];
$validated = $request->validate(['name' => 'required|string|max:255', ...]);
User::create($validated);

// ❌ SQLi Laravel-spécifique
User::whereRaw('email = "' . $email . '"')->get();
$col = $request->input('sort');
User::orderByRaw($col)->get(); // injection orderBy

// ✅ FIX
User::whereRaw('email = ?', [$email])->get();
$allowed = ['name', 'email', 'created_at'];
if (in_array($col, $allowed)) User::orderBy($col)->get();
```

### RCE Livewire (CVE-2025-54068)
Hydratation propriétés Livewire v3 ≤ 3.6.3 sans vérification d'authentification → RCE. Affecter sans patch = compromission complète.

### Upload triple validation (CVE-2024-47823)
```php
// ❌ VULN — extension devinée via MIME → shell.php uploadé comme image/png
$request->validate(['file' => 'file|mimes:jpg,png']);

// ✅ FIX — MIME + extension + taille + stockage privé
$request->validate(['file' => ['file', 'mimes:pdf,jpg,png', 'max:10240',
  fn($a, $v, $fail) => in_array(strtolower($v->getClientOriginalExtension()),
    ['php','phar','phtml','php3']) && $fail('PHP not allowed.')]]);
$file->storeAs('documents', Str::uuid().'.'.$file->guessExtension(), 'private');
// Nginx : location ~* /uploads/.*\.php$ { deny all; }
```

### CVE-2024-52301 — env via query string (CVSS 8.7)
`register_argc_argv` activé → env changé via `?APP_ENV=local` → debug mode → secrets exposés.
```php
// php.ini ou php-fpm.conf
// register_argc_argv = Off   (PHP 8.5 déprécie pour HTTP)
// ✅ FIX : Laravel 11.31.0+ + PHP 8.4+
```

## Failles Spécifiques PHP Natif

### `unserialize()` + `phar://` → RCE
```php
unserialize($_COOKIE['session_data']); // ❌ — chaînes gadgets PHPGGC disponibles

// ✅ FIX
json_decode($_COOKIE['data'], true);
// Si unserialize obligatoire :
unserialize($input, ['allowed_classes' => [SafeClass::class]]);
```
`phar://` : `file_exists()`, `fopen()`, `is_dir()` sur `phar://` → désérialisation implicite → même RCE.

### Type juggling PHP 8.x
```php
// ❌ VULN — {"password": true} peut bypasser (true == any_string)
if ($input['password'] == $stored_password) authenticate();

// ✅ FIX — type strict + comparaison timing-constant
if (!is_string($input['password'])) throw new \InvalidArgumentException();
if (!hash_equals($stored_hash, hash('sha256', $input['password']))) deny_access();
```

### PHP Filter Chains LFI → RCE (CVE-2026-22200)
```
# Toute LFI = RCE sans upload via chaînes de filtres
page=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF8.UTF7|.../resource=/etc/passwd
```
Synacktiv technique : `convert.iconv.*` chaîné → octets arbitraires → code exécutable. Activement exploité (osTicket CVE-2026-22200).

```php
// ❌ include($_GET['page'] . '.php');
// ✅ FIX — whitelist stricte
$allowed = ['home', 'about', 'contact'];
$page = in_array($_GET['page'], $allowed) ? $_GET['page'] : 'home';
include('pages/' . $page . '.php');
// + open_basedir = /var/www/html + allow_url_include = Off dans php.ini
```

### `extract()` — corruption mémoire / EXTR_REFS
```php
extract($_GET);          // ❌ ?is_admin=1 écrase variables
extract($_GET, EXTR_REFS); // ❌ use-after-free → RCE contourne disable_functions

$username = $_POST['username'] ?? ''; // ✅ extraction explicite
```

### Blade injection Laravel
```php
// ❌ VULN — {!! $userInput !!} = HTML non échappé → XSS
{!! $userInput !!}

// ✅ FIX — double accolades = échappement automatique
{{ $userInput }}
// Si HTML requis : DOMPurify côté client ou HTMLPurifier côté serveur
```

## Defense Checklist
- [ ] APP_KEY : jamais dans Git, rotation si exposée, PHPGGC check des chaînes gadgets
- [ ] Mass assignment : `$fillable` explicite — jamais `$guarded = []` ni `create($request->all())`
- [ ] Livewire : version ≥ 3.6.4 — CVE-2025-54068 critique CVSS non divulgué
- [ ] Upload : validation MIME + extension + liste noire PHP + stockage privé + Nginx deny PHP dans uploads
- [ ] `unserialize()` : `allowed_classes` restreint ou remplacer par JSON
- [ ] PHP Filter Chains : `open_basedir`, `allow_url_include=Off`, whitelist `include`/`require`
- [ ] Type comparison : `===` strict et `hash_equals()` pour secrets — jamais `==`
- [ ] `extract()` : interdit sur données externes — extraction variable par variable
- [ ] PHP 8.4+ : PHP 8.1 EOL décembre 2025, `#[SensitiveParameter]` masque secrets en stack traces
- [ ] Blade : `{{ }}` uniquement — `{!! !!}` réservé aux données déjà sanitisées
