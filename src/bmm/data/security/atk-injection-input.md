# atk-injection-input.md
> Attack patterns where untrusted input is interpreted as code by a query language, markup parser, or template engine.

---

## 1. SQL Injection — String Interpolation

**Description:** User-controlled input is concatenated directly into a SQL string instead of being bound as a parameterized value. The input is parsed as SQL syntax, not as data.

**Vector:** Any code path that constructs a query via string formatting (`format!`, f-strings, `+` concatenation, string templates) and then executes it against a database.

**Impact:** Data exfiltration via `UNION SELECT`, schema enumeration via `sqlite_master` / `information_schema`, authentication bypass via `OR 1=1`, and — on APIs that accept multi-statement execution — arbitrary DDL/DML.

**Code review signs:**
- String formatting functions (`format!`, `sprintf`, f-strings, template literals) appearing in the same code block as SQL keywords (`SELECT`, `INSERT`, `WHERE`, `FROM`).
- Multi-statement execution APIs (e.g., `execute_batch`, `executescript`, `multi_query`) receiving any externally derived string.
- ORM methods that accept raw query fragments (`.raw()`, `.extra()`, `literalQuery`) mixed with unvalidated input.

---

## 2. SQL Injection — LIKE / GLOB Wildcard Bypass

**Description:** Even when parameterized queries are used correctly, the wildcard characters `%` (match any sequence) and `_` (match exactly one character) inside a `LIKE` or `GLOB` clause are still interpreted by the SQL engine after binding. The binding prevents SQL syntax injection but not pattern-matching abuse.

**Vector:** Search fields that pass user input as the bound value of a `LIKE` predicate without escaping wildcard metacharacters first.

**Impact:**
- `%` alone matches every row — full table dump with a single query.
- `%keyword%` extracts rows containing sensitive words.
- Complex patterns cause full-table sequential scans — denial of service on large tables.

**Code review signs:**
- `LIKE` or `GLOB` clauses where the bound parameter comes from user input.
- Absence of a wildcard-escape preprocessing step before the bind call.
- Missing `ESCAPE` clause on the `LIKE` predicate.

---

## 3. SQL Injection — Second-Order (Stored Payload)

**Description:** A malicious payload is stored safely in the database via a parameterized insert (no injection at write time). Later, a different code path reads that stored value and interpolates it into a dynamically built query — triggering the injection.

**Vector:** External content (scraped data, user-supplied names, LLM-generated text, imported config files) stored in one operation and re-used in raw SQL construction in a separate operation, often written by a different developer.

**Impact:** Same as direct SQL injection. The delay between storage and trigger makes detection harder; automated scanners often miss it because the injection point and the dangerous re-use are in different code locations.

**Code review signs:**
- A database read whose result is immediately passed into a string-formatting function that builds SQL.
- Fields tagged as "external", "scraped", "imported", or "llm_output" appearing in query-construction code.
- Lack of a consistent policy requiring parameterized queries even for values sourced from the database itself.

---

## 4. SQL Injection — Information Leakage via Error Messages

**Description:** Database error objects contain internal details — table names, column names, full query text, error offsets. When these are serialized to strings and returned to the caller (API response, frontend, log file), an attacker learns the schema without executing a successful injection.

**Vector:** Error handlers that call `.to_string()`, `.message`, or equivalent on raw database exception objects before returning a response.

**Impact:** The attacker learns exact table and column names, which dramatically reduces the effort needed to craft working injection payloads.

**Code review signs:**
- `map_err(|e| e.to_string())` or equivalent on database calls in API handler functions.
- Database exceptions caught and re-thrown with their original message attached.
- No generic error mapping layer between the database driver and the API boundary.

---

## 5. XSS — Stored via Scraped or External Content

**Description:** Content fetched from an external source (RSS feeds, web scraping, third-party APIs) contains HTML or JavaScript. The ingestion layer — a parser like BeautifulSoup, an HTML library, or a JSON deserializer — faithfully preserves that content. When the application later renders it, the payload executes.

**Important nuance:** An HTML parser is not a sanitizer. Parsers preserve `<script>`, event handler attributes (`onerror`, `onload`, `onclick`), SVG, and MathML nodes. Treating a parser as a sanitizer is a common architectural mistake.

**Vector:** External content is fetched, stored, and rendered without a dedicated sanitization step that removes or encodes executable constructs.

**Impact:** In a web application: session hijacking, credential theft. In a desktop application with a privileged native bridge (e.g., Electron/Tauri IPC), a stored XSS payload can invoke system-level APIs — read local files, execute subprocesses — escalating to local code execution.

**Code review signs:**
- Rendering functions (`.innerHTML`, `dangerouslySetInnerHTML`, `v-html`) receiving values that originated from external sources.
- Absence of a sanitization library call between the fetch/parse step and the render step.
- HTML parsers imported and used without a companion sanitizer import.

**Real incident:** CVE-2025-53773 — GitHub Copilot (CVSS 9.6, 2025): prompt injection via external content caused XSS in the AI output rendering pipeline, demonstrating the same pattern across LLM-generated content.

---

## 6. XSS — DOM-Based

**Description:** JavaScript reads a value from a browser-controlled source (URL fragment, `document.referrer`, `window.name`, `localStorage`) and writes it into the DOM via a sink that executes scripts. The server is never involved; the payload lives entirely in the client.

**Vector:** Client-side code that reads URL parameters or hash values and writes them to `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout(string)`, or `href` attributes without sanitization.

**Impact:** Same as reflected XSS — the exploit URL can be delivered via phishing.

**Code review signs:**
- `location.hash`, `location.search`, or `document.referrer` used as input to any DOM-writing function without encoding.
- `javascript:` URIs in `href` or `src` attributes constructed from external data.
- React: `href={userValue}` without scheme validation — React 18/19 warns but does not block `javascript:` URIs on click.

---

## 7. XSS — Mutation XSS (mXSS)

**Description:** A sanitizer processes HTML and produces output it considers clean. The browser's HTML parser then re-parses that output in context (e.g., inside a `<table>`, a MathML node, a `<template>`) and produces a different DOM tree containing executable script. The transformation happens after sanitization.

**Vector:** Applications that sanitize once and then insert the result into a different HTML context than the one used during sanitization, or that rely on a single sanitizer without defense-in-depth.

**Impact:** Bypasses sanitizers entirely, including well-maintained ones with active CVE records.

**Code review signs:**
- Single-pass sanitization with no secondary encoding at the render site.
- Content inserted into `<table>`, SVG, or MathML contexts after sanitization designed for a generic HTML context.

**Real incident:** CVE-2025-26791 — DOMPurify (CVSS 4.5, February 2025): MathML/SVG namespace confusion caused mXSS after sanitization. Fixed in DOMPurify v3.2.4.

---

## 8. XSS → Privilege Escalation via Native Bridge

**Description:** In applications that embed a web renderer with access to a privileged native API layer (desktop apps, mobile WebViews), a successful XSS payload can invoke native commands — reading files, spawning processes, or exfiltrating secrets — beyond what a browser sandbox would permit.

**Vector:** XSS on any rendered page combined with exposed native bridge APIs (`window.__TAURI_INTERNALS__`, `window.webkit.messageHandlers`, Electron `ipcRenderer`, etc.).

**Impact:** XSS becomes local code execution. An attacker who injects script into a news feed reader with filesystem access can read SSH keys, credentials files, or environment variables without any further exploit.

**Code review signs:**
- Native bridge APIs available globally (not scoped or protected).
- CSP missing or too permissive (`unsafe-inline`, `connect-src *`) in the app configuration.
- No capability restriction on which native commands can be invoked from which page origins.

**Real incident:** CVE-2025-31477 — tauri-plugin-shell < 2.2.1 (CVSS 9.8, April 2025): XSS could invoke `shell.open()` with `file://` or `smb://` URIs.

---

## 9. Template Injection (Server-Side and Client-Side)

**Description:** User-controlled input is embedded inside a template string that is later evaluated by a template engine. The engine interprets the input as template directives, enabling the attacker to execute arbitrary expressions within the engine's scope.

**Vector:** Any template rendering call where part of the template string — not just the data values — is derived from user input. Common in email builders, report generators, and configuration-driven renderers.

**Impact:** Ranges from data exfiltration (reading server-side variables) to full remote code execution when the template engine has access to system primitives (Python `os.popen`, Java `Runtime.exec`, JavaScript `Function` constructor).

**Code review signs:**
- Template engine render calls where the template itself (not just the context/data) contains user-supplied strings.
- Expression delimiters (`{{`, `${`, `<%=`, `#{`) appearing in concatenated strings built from external input.
- `eval()`, `new Function(string)`, or equivalent used to evaluate strings that include external values.

---

## 10. Header Injection (HTTP Response Splitting)

**Description:** Unvalidated user input is placed inside an HTTP response header. If the input contains carriage-return / line-feed characters (`\r\n`), the attacker can inject additional headers or split the response into two separate HTTP messages, each with attacker-controlled content.

**Vector:** Redirect targets, cookie values, or custom header values derived from user input without stripping CR/LF.

**Impact:** Cache poisoning, cookie injection, phishing via crafted redirect pages, bypassing security headers (CSP, HSTS) injected into the second response.

**Code review signs:**
- `Location`, `Set-Cookie`, or custom header values constructed from user input without CR/LF stripping.
- Redirect logic that reflects a `?returnUrl=` or `?next=` parameter directly into a `Location` header.

---

## Universal Grep Patterns (language-agnostic concepts)

| What to look for | Risk |
|---|---|
| String format/concatenation adjacent to SQL keyword literals | SQL injection |
| `LIKE` or `GLOB` without adjacent escape logic | Wildcard injection |
| DB read result passed to string-format then to DB execute | Second-order injection |
| Raw DB exception serialized into API response | Schema leakage |
| HTML parser import without sanitizer import in same pipeline | Stored XSS |
| `innerHTML`/`dangerouslySetInnerHTML` receiving externally sourced values | DOM XSS |
| Template render call where template string is user-derived | Template injection |
| Header-setting code whose value includes a URL or user string | Header injection |
