# Stack Python — Security Patterns
> Tags: python, django, flask, fastapi
> Load when: project uses Python (detected via requirements.txt, pyproject.toml, *.py)

## CVEs Critiques

| CVE | CVSS | Framework | Impact | Fix |
|-----|------|-----------|--------|-----|
| CVE-2025-64459 | 9.1 | Django ORM | SQLi via `**kwargs` filter() | 5.2.8 / 5.1.14 / 4.2.26 |
| CVE-2025-57833 | 7.1 | Django ORM | SQLi `FilteredRelation` aliases | 5.2.6 / 5.1.12 / 4.2.24 |
| CVE-2025-27516 | Critical | Jinja2 | SSTI sandbox breakout → RCE | Jinja2 3.1.6 |
| CVE-2024-34069 | 7.5 | Werkzeug | CSRF → debugger PIN recalculable → RCE | Werkzeug 3.0.3 |
| CVE-2024-33663 | 7.5 | python-jose | JWT algo confusion HS256+ECDSA → forge | Migrer PyJWT |
| CVE-2025-32444 | 10.0 | vLLM/pickle | ZeroMQ + pickle → RCE | Remplacer pickle |
| CVE-2025-68481 | High | fastapi-users | OAuth state sans entropie → account takeover | fastapi-users 15.0.2 |
| CVE-2024-47874 | 8.7 | Starlette | DoS multipart OOM < 60s | Starlette 0.40.0 |

## Failles Spécifiques Django

### ORM Injection via `**kwargs` (CVE-2025-64459, CVSS 9.1)
```python
# ❌ VULN — attaquant envoie _connector=OR&is_superuser=True
users = User.objects.filter(**request.GET.dict())

# ✅ FIX — allowlist stricte
ALLOWED = {'username', 'email', 'is_active'}
users = User.objects.filter(**{k: v for k, v in request.GET.dict().items() if k in ALLOWED})
```

### Autres vecteurs ORM vulnérables
```python
User.objects.raw(f"WHERE username = '{user_input}'")          # ❌ SQLi
User.objects.raw("WHERE username = %s", [user_input])         # ✅
qs.annotate(val=RawSQL(f"WHERE bar = '{input}'", []))         # ❌
qs.annotate(val=RawSQL("WHERE bar = %s", [input]))            # ✅
Entry.objects.extra(where=["headline = '%s'" % input])        # ❌
Entry.objects.extra(where=['headline=%s'], params=[input])    # ✅
```

### DRF Mass Assignment + BOLA
```python
# ❌ VULN — expose is_staff
class Meta: model = User; fields = '__all__'

# ✅ FIX
class Meta:
    fields = ['username', 'email']
    read_only_fields = ['is_staff', 'is_superuser']
    extra_kwargs = {'password': {'write_only': True}}

# ❌ BOLA — accès à toutes les commandes
class OrderView(generics.RetrieveAPIView): queryset = Order.objects.all()
# ✅ FIX
def get_queryset(self): return Order.objects.filter(user=self.request.user)
```

## Failles Spécifiques Flask/Werkzeug

### Debugger RCE (CVE-2024-34069)
```python
app.run(debug=True, host='0.0.0.0')  # ❌ RCE en prod — PIN SHA1 recalculable via LFI
```
Cookie `__wzd*` forgeable pour bypasser le verrou 3 tentatives. Module Metasploit disponible.

### Session forgery (SECRET_KEY faible)
```python
# Brute-force : flask-unsign --unsign --cookie '<cookie>' --wordlist rockyou.txt
# ❌ Clé faible ou CHANGEME
# ✅ FIX
app.config['SECRET_KEY'] = os.environ.get('FLASK_SECRET_KEY') or secrets.token_hex(32)
```

### SSTI Jinja2 → RCE (CVE-2025-27516)
```python
return render_template_string(f'Hello {name}!')               # ❌ VULN
return render_template_string('Hello {{ name }}!', name=name) # ✅ FIX
```
Payload bypass filtres `__` : `{{request|attr('\x5f\x5fclass\x5f\x5f')}}`

## Failles Spécifiques FastAPI

### JWT — python-jose abandonné (CVE-2024-33663/33664)
```python
# ❌ VULN — algo non fixé → confusion HS256 + clé publique ECDSA
from jose import jwt; payload = jwt.decode(token, public_key)

# ✅ FIX — PyJWT avec algo explicite
import jwt
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"],
    audience="my-api", issuer="auth-server",
    options={"require": ["exp", "iat", "sub", "iss", "aud"]})
```

### WebSocket — auth avant accept()
```python
# ❌ VULN — accepte avant de valider
await websocket.accept(); token = await websocket.receive_text()

# ✅ FIX
async def ws(websocket: WebSocket, token: str = Query(...)):
    try: payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except: await websocket.close(code=1008); return
    await websocket.accept()
```

### Pydantic coercion bypass
```python
# {"is_admin": "yes"} → True en mode lax
# ✅ FIX — strict pour champs sensibles
class UserModel(BaseModel):
    model_config = ConfigDict(strict=True)
    is_admin: bool
```

## Python Transversal

### Pickle/YAML → RCE
```python
pickle.loads(data)           # ❌ CVSS 10.0 si data externe
yaml.load(data, Loader=...)  # ❌ RCE via !!python/object/apply:
yaml.safe_load(data)         # ✅
json.loads(data)             # ✅ alternatif
unserialize_allowed = {'allowed_classes': [SafeClass]}  # si pickle obligatoire
```

### subprocess shell injection
```python
subprocess.check_output('nslookup ' + hostname, shell=True)   # ❌
subprocess.check_output(['nslookup', hostname], shell=False)   # ✅
```

## Defense Checklist
- [ ] Django : allowlist stricte avant tout `filter(**kwargs)` — jamais `request.GET.dict()` direct
- [ ] Flask : `debug=False` en prod, `SECRET_KEY` depuis env var, `render_template()` jamais f-string
- [ ] FastAPI : migrer python-jose → PyJWT, algo fixé, claims `require` complets
- [ ] Pydantic : `ConfigDict(strict=True)` pour champs sensibles (is_admin, role, etc.)
- [ ] Pickle/YAML : interdire sur données externes — JSON/safetensors uniquement
- [ ] Versions min : Werkzeug ≥ 3.1.6, Jinja2 ≥ 3.1.6, Starlette ≥ 0.49.1, Flask-CORS ≥ 5.0.0
- [ ] Supply chain : `pip install --require-hashes`, `pip-audit` en CI/CD
- [ ] DoS pagination DRF : `max_page_size = 100` dans StandardPagination
