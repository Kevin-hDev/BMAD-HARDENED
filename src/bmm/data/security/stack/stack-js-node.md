# Stack JS/Node — Security Patterns
> Tags: javascript, node, express, nextjs, nestjs
> Load when: project uses Node.js (detected via package.json, node_modules/, *.js/*.ts)

## CVEs Critiques

| CVE | CVSS | Framework | Impact | Fix |
|-----|------|-----------|--------|-----|
| CVE-2025-55182 | 10.0 | Next.js RSC | RCE pré-auth via déserialisation Flight | React 19.0.1+, Next.js 15.5.7+ |
| CVE-2025-29927 | 9.1 | Next.js | Middleware auth bypass complet | Next.js 15.2.3+ |
| CVE-2025-54782 | 9.4 | NestJS DevTools | RCE via CSRF + sandbox escape | @nestjs/devtools 0.2.1+ |
| CVE-2022-29078 | 9.8 | Express EJS | SSTI → RCE via req.query dans render() | EJS 3.1.7+ |
| CVE-2025-49826 | 7.5 | Next.js ISR | DoS persistant race condition cache | Next.js 15.1.8+ |
| CVE-2026-2293 | 7.5 | NestJS Fastify | Auth bypass TOCTOU URL decode | @nestjs/platform-fastify 11.1.14+ |
| CVE-2025-47935 | 7.5 | Express Multer | DoS mémoire upload | Multer 2.0.0+ |

## Failles Spécifiques Express

### Prototype Pollution (CVE-2022-24999)
```javascript
// ❌ VULN — __proto__[isAdmin]=true dans body → auth bypass
const user = {}; Object.assign(user, req.body);
if (user.isAdmin) res.send("Admin Access");

// ✅ FIX — destructuration + objet sans prototype
const { username, email } = req.body;
const user = Object.create(null); user.isAdmin = false;
```

### EJS SSTI → RCE (CVE-2022-29078, CVSS 9.8)
```javascript
// ❌ VULN — req.query directement dans render()
app.get('/page', (req, res) => res.render('page', req.query));
// Payload : /page?settings[view%20options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('id');s

// ✅ FIX
const safeData = { name: String(req.query.name || '').slice(0, 100) };
res.render('page', safeData);
```
> JAMAIS passer `req.query`, `req.body`, `req.params` directement à `res.render()`.

### Path traversal `sendFile()`
```javascript
// ❌ VULN — GET /files/..%2F..%2Fetc%2Fpasswd
res.sendFile(path.join(__dirname, 'uploads', req.params.filename));

// ✅ FIX
const rootDir = path.resolve(__dirname, 'uploads');
const filePath = path.resolve(rootDir, req.params.filename);
if (!filePath.startsWith(rootDir + path.sep)) return res.status(403).send('Forbidden');
```

### Session fixation + CORS reflection
```javascript
// ❌ VULN session — ID inchangé après login (CWE-384)
req.session.userId = user.id;
// ✅ FIX
req.session.regenerate((err) => { req.session.userId = user.id; req.session.save(cb); });

// ❌ VULN CORS — reflection d'origin + credentials
res.header('Access-Control-Allow-Origin', req.headers.origin);
res.header('Access-Control-Allow-Credentials', 'true');
// ✅ FIX — liste blanche stricte
app.use(cors({ origin: (o, cb) => ['https://app.example.com'].includes(o) ? cb(null, true) : cb(new Error()), credentials: true }));
```

## Failles Spécifiques Next.js

### CVE-2025-29927 — Bypass middleware (CVSS 9.1)
```http
GET /admin HTTP/1.1
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```
Header interne `x-middleware-subrequest` spoofable → tout middleware court-circuité.

```nginx
# Mitigation si update impossible
proxy_set_header x-middleware-subrequest "";
```
> Ne JAMAIS reposer uniquement sur le middleware pour l'auth. Vérifier au niveau data.

### CVE-2025-55182 « React2Shell » — RCE pré-auth (CVSS 10.0)
Protocole Flight RSC : déserialisation sans validation → RCE via POST HTTP avant toute auth.
- Toute app `create-next-app` App Router vulnérable par défaut
- 39% des envs cloud vulnérables (Wiz Research), exploitation active J+2
- Affectées : React 19.0.0–19.2.0, Next.js 15.x/16.x

### Server Actions — chaque action = endpoint POST public
```typescript
// ❌ VULN — aucune auth, aucune validation
'use server'
export async function deleteUser(userId: string) {
  await db.user.delete({ where: { id: userId } });
}

// ✅ FIX
'use server'
const schema = z.object({ userId: z.string().uuid() });
export async function deleteUser(input: unknown) {
  const session = await auth(); if (!session?.user) throw new Error('Unauthorized');
  const { userId } = schema.parse(input);
  if (session.user.role !== 'admin') throw new Error('Forbidden');
  await db.user.delete({ where: { id: userId } });
}
```

### Fuite Server → Client + SSRF
```typescript
// ❌ VULN — hash pwd, SSN sérialisés dans payload RSC
const userData = await sql`SELECT * FROM user WHERE slug = ${slug}`;
return <Profile user={userData} />

// ✅ FIX
import 'server-only';
return await sql`SELECT name, bio, avatar FROM user WHERE slug = ${slug}`;
```
`NEXT_PUBLIC_` embarque les secrets dans le bundle client au build — ne jamais préfixer des secrets.

## Failles Spécifiques NestJS

### CVE-2025-54782 — RCE DevTools (CVSS 9.4)
`@nestjs/devtools-integration` ≤ 0.2.0 : `vm.runInNewContext` + pas de validation Origin → CSRF → sandbox escape → RCE.

### CVE-2026-2293 — Bypass TOCTOU URL decode (CVSS 7.5)
```
GET /%61dmin → Middleware voit "/%61dmin" (no match) → Fastify décode → /admin → Handler exécuté
GET /admin/   → Middleware no-match trailing slash → Fastify normalise → /admin → Handler exécuté
```

### Guard + ValidationPipe
```typescript
// ❌ VULN — RolesGuard sans AuthGuard : req.user = undefined
@UseGuards(RolesGuard)

// ✅ FIX — Auth d'abord
@UseGuards(JwtAuthGuard, RolesGuard)

// ✅ ValidationPipe global
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  transformOptions: { enableImplicitConversion: false }
}));
```
GraphQL : `depthLimit(7)`, `createComplexityLimitRule(1000)`, `introspection: false` en prod.
WebSocket NestJS : Guards ne s'exécutent PAS sur `handleConnection()` → auth dans `handleConnection()`.

## Defense Checklist
- [ ] Express : jamais `req.query`/`req.body` direct dans `res.render()` — extraction explicite
- [ ] Express : `req.session.regenerate()` après login — jamais réutiliser l'ID de session
- [ ] Next.js : update immédiat CVE-2025-29927 + CVE-2025-55182 — critique CVSS 9.1/10.0
- [ ] Next.js : auth dans chaque Server Action — jamais confier au middleware seul
- [ ] NestJS : `@UseGuards(JwtAuthGuard, RolesGuard)` — ordre Auth avant Role obligatoire
- [ ] NestJS : `ValidationPipe(whitelist: true, forbidNonWhitelisted: true)` global
- [ ] NestJS DevTools : désactivé en prod — `@nestjs/devtools-integration` hors prod uniquement
- [ ] CORS : liste blanche stricte — jamais reflection d'origin + credentials: true
