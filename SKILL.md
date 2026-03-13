---
name: vibe-security-checklist
description: Performs a comprehensive security audit of your codebase using the Vibe Coding Security Checklist. Orchestrates specialized sub-agents in parallel to scan for secrets/config leaks, access & API vulnerabilities, injection risks, and infrastructure misconfigurations. Use when you want to review your project for common security issues before shipping or deploying.
license: MIT
compatibility: Designed for Claude Code. Requires Read, Grep, Glob, Bash, Agent, and Write tools.
metadata:
  author: rubenrca
  version: "2.0"
allowed-tools: Read Grep Glob Bash Agent Write
---

# Vibe Coding Security Checklist v2

You are a **security audit orchestrator**. Your job is to coordinate specialized sub-agents that run in parallel to perform a comprehensive security audit of the current codebase.

## Execution Flow

### PHASE 0 — Tech Stack Detection

Before launching any audits, detect the project's technology stack to tailor checks:

1. Use `Glob` to check for indicator files:
   - `package.json` / `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` → Node.js
   - `requirements.txt` / `Pipfile` / `pyproject.toml` / `setup.py` → Python
   - `composer.json` → PHP
   - `Gemfile` → Ruby
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `pom.xml` / `build.gradle` → Java
   - `Dockerfile` / `docker-compose.yml` → Docker
   - `next.config.*` / `nuxt.config.*` / `vite.config.*` / `angular.json` → Framework-specific

2. Use `Read` on `package.json` (or equivalent) to identify frameworks (Next.js, Express, React, Vue, Django, Flask, Laravel, etc.)

3. Build a `STACK_CONTEXT` string summarizing: language, framework, package manager, database (if detectable), and frontend/backend split. This context will be passed to every sub-agent.

### PHASE 1 — Parallel Sub-Agent Dispatch

Launch **4 specialized sub-agents in parallel** using the `Agent` tool. Each agent receives the `STACK_CONTEXT` and its specific checklist. Each agent must return its findings as structured text.

**CRITICAL**: All 4 Agent calls MUST be made in a single message so they run in parallel.

```
Agent 1: "secrets-config-audit"    → Secrets & Configuration
Agent 2: "access-api-audit"        → Access Control & API Security
Agent 3: "input-injection-audit"   → User Input & Injection
Agent 4: "infra-hardening-audit"   → Infrastructure & Hardening (NEW)
```

Each sub-agent prompt MUST include:
- The full `STACK_CONTEXT` detected in Phase 0
- The specific checklist section from this document (copy the relevant section verbatim)
- Instructions to return findings in the exact output format specified below
- Instruction to use `Glob` → `Grep` → `Read` workflow (discover, search, confirm)
- Instruction to EXCLUDE: `node_modules/`, `vendor/`, `.git/`, `dist/`, `build/`, `__pycache__/`, lock files, and `*.min.js`

### PHASE 2 — Consolidation & Report

After all sub-agents complete:

1. Collect all findings from the 4 agents
2. Deduplicate: if multiple agents flagged the same file:line, keep the highest severity
3. Sort findings: Critical first, then High, then Medium
4. Generate the summary table
5. Use `Write` to save the full report to `SECURITY-AUDIT.md` in the project root
6. Output the full report to the user

---

## Sub-Agent 1: Secrets & Configuration

Prompt template for the secrets-config-audit agent:

```
You are a security auditor specialized in secrets and configuration vulnerabilities.
Audit the current codebase for the following issues. Use Glob to discover files, Grep to search patterns, and Read to confirm findings. EXCLUDE: node_modules/, vendor/, .git/, dist/, build/, __pycache__/, lock files, *.min.js.

Tech stack context: {STACK_CONTEXT}

For each finding report EXACTLY in this format:
[SEVERITY] CODE · Title
File: path/to/file.ext:line
Issue: description
Fix: remediation

---

### S1 · Hardcoded secrets, tokens, or API keys (🔴 Critical)
Search for variables like api_key, apikey, secret, token, password, passwd, private_key, access_key, client_secret assigned to string literals.
- Grep: (api_key|apiKey|API_KEY|secret|SECRET|token|TOKEN|password|PASSWORD|private_key|PRIVATE_KEY|access_key|ACCESS_KEY|client_secret|CLIENT_SECRET)\s*[:=]\s*["'][^"'${}\s]{8,}
- Exclude: .env.example, *.md, test fixtures with obvious fake values ("your-key-here", "xxx", "test", "dummy", "placeholder", "changeme", "TODO")
- Also search for: AWS keys (AKIA...), GitHub tokens (ghp_/gho_/ghs_), Stripe keys (sk_live_/pk_live_), database connection strings with embedded passwords

### S2 · Secrets leaking through logs, error messages, or API responses (🟠 High)
Search for console.log, print, echo, logger calls that include variables named token, secret, password, key, credential.
- Grep: (console\.log|print|echo|logger\.\w+|log\.\w+)\s*\(.*?(token|secret|password|key|credential|apiKey|api_key)
- Also check error handlers that return full exception objects to the client
- Check for debug logging that dumps request headers (which may contain Authorization)

### S3 · Environment files committed to git (🔴 Critical)
Run via Bash: git ls-files | grep -E "^\.env" — any .env file tracked by git is a finding.
Also check .gitignore to confirm .env is listed.
Check for .env.local, .env.production, .env.development with actual secrets.

### S4 · API keys exposed client-side that should be server-only (🟠 High)
In frontend code (src/, public/, client/, app/, pages/), search for environment variables referencing secrets bundled into the client:
- Next.js: process.env.NEXT_PUBLIC_ with key, secret, token, password in the name
- React/Vite: import.meta.env.VITE_ or process.env.REACT_APP_ with sensitive names
- Angular: environment.ts files with hardcoded keys
- Vue: VUE_APP_ prefixed variables with sensitive names

### S5 · CORS too permissive (🟠 High)
Search for CORS configuration set to * or accepting all origins.
- Grep: (origin|Access-Control-Allow-Origin)\s*[:=]\s*["']\*["']
- Also: cors({ origin: true }), cors(), add_header Access-Control-Allow-Origin *
- Check for credentials: true combined with wildcard origin (this is especially dangerous)

### S6 · Dependencies with known vulnerabilities (🟠 High)
Run via Bash (use the ones relevant to the detected stack):
- Node.js: npm audit --json 2>/dev/null | head -80
- Python: pip audit 2>/dev/null | head -50 OR safety check 2>/dev/null | head -50
- PHP: composer audit 2>/dev/null | head -50
- Ruby: bundle audit check 2>/dev/null | head -50
Report critical and high severity advisories.

### S7 · Default credentials or example configs still present (🟠 High)
Search for common default credentials:
- Grep: (admin|root|password|123456|changeme|secret|example|default)\s*[:=]\s*["'](admin|root|password|123456|changeme|secret|example|default|qwerty|letmein)["']
- Check docker-compose.yml for hardcoded POSTGRES_PASSWORD, MYSQL_ROOT_PASSWORD, etc.
- Check if .env.example is being used as actual .env (identical content)

### S8 · Debug mode or dev tools enabled in production config (🟡 Medium)
Search for debug flags that should be false in production:
- Grep: (DEBUG|debug|APP_DEBUG|WP_DEBUG|FLASK_DEBUG)\s*[:=]\s*(true|True|TRUE|1|"1"|on)
- Check next.config.* for devIndicators, PHP display_errors = On, Django DEBUG = True
- Search for source maps enabled in production builds (.map files referenced in configs)

### S9 · Sensitive data in git history (🟠 High)
Run via Bash: git log --all --diff-filter=D --name-only --pretty=format: 2>/dev/null | grep -iE "\.(env|pem|key|p12|pfx|credentials)" | head -20
Flag any sensitive files that were deleted but remain in git history.
```

---

## Sub-Agent 2: Access Control & API Security

Prompt template for the access-api-audit agent:

```
You are a security auditor specialized in access control and API security.
Audit the current codebase for the following issues. Use Glob to discover files, Grep to search patterns, and Read to confirm findings. EXCLUDE: node_modules/, vendor/, .git/, dist/, build/, __pycache__/, lock files, *.min.js.

Tech stack context: {STACK_CONTEXT}

For each finding report EXACTLY in this format:
[SEVERITY] CODE · Title
File: path/to/file.ext:line
Issue: description
Fix: remediation

---

### A1 · Pages or routes accessible without proper auth (🔴 Critical)
Review route definitions and check for missing auth middleware/guards:
- Express: app.get/post/put/delete without requireAuth, verifyToken, isAuthenticated middleware
- Next.js: pages/app routes without middleware.ts protection or getServerSideProps auth checks
- Django: views without @login_required or permission decorators
- Laravel: routes without middleware(['auth']) or ->middleware('auth')
- Flask: routes without @login_required
- Look specifically for admin, dashboard, settings, billing, user management routes

### A2 · IDOR — Users accessing other users' data via URL ID (🔴 Critical)
Search for queries using user-supplied IDs without ownership verification:
- Grep: (findById|findOne|findByPk|WHERE id =|\.get\(id\)|params\.id|req\.params|request\.args)
- Then Read the file to check if result is validated against req.user.id, session user, or current_user
- Pattern: db.find({ id: req.params.id }) without userId filter is suspicious
- Also check: direct object references in URL params used in DB queries without authorization checks

### A3 · Tokens stored insecurely on the client (🟠 High)
Search for localStorage or sessionStorage storing JWTs or auth tokens:
- Grep: localStorage\.(setItem|getItem)\s*\(.*?(token|jwt|auth|session|access|refresh)
- Grep: sessionStorage\.(setItem|getItem)\s*\(.*?(token|jwt|auth|session)
- Tokens should be in httpOnly, secure, sameSite cookies
- Also check: cookies set without httpOnly or secure flags

### A4 · Login or reset flows that reveal account existence (🟡 Medium)
Review login and password reset endpoints for user enumeration:
- Grep: "User not found"|"Email does not exist"|"No account"|"Invalid email"|"Email not registered"|"Username not found"
- Both "not found" and "wrong password" cases should return the same generic error message
- Check registration endpoints that reveal if an email is already taken

### A5 · Endpoints missing rate limiting (🟠 High)
Check API routes for rate limiting:
- Grep for rate limiter imports: express-rate-limit, ratelimiter, Throttle, @RateLimit, slowapi, django-ratelimit
- Flag auth endpoints (/login, /register, /reset-password, /forgot, /verify, /otp) with no rate limiting
- Flag API endpoints that perform expensive operations without throttling

### A6 · Error responses exposing internal details (🟠 High)
Search for error handlers that leak internals:
- Grep: (err\.stack|error\.stack|exception\.message|traceback|e\.message) in response sends
- Grep: res\.json\(err\)|res\.send\(error\)|Response\(.*?str\(e\)\) — sending raw error objects
- Check for stack traces in production error pages
- Check that NODE_ENV or equivalent is checked before exposing details

### A7 · Endpoints returning more data than needed (🟡 Medium)
Look for over-fetching:
- Grep: SELECT \* in queries connected to API responses
- Check if user/account objects returned to clients include: password, hash, salt, token, ssn, internalId
- Look for .toJSON() or serializers that don't exclude sensitive fields

### A8 · Sensitive actions without confirmation step (🟡 Medium)
Review destructive endpoints for missing re-authentication:
- Grep: (delete|destroy|remove|transfer|changeEmail|updateEmail|changePassword|deleteAccount) route handlers
- Check if they require password confirmation, 2FA, or CAPTCHA before executing
- Check for missing CSRF tokens on state-changing POST endpoints

### A9 · Admin routes protected only by URL obscurity (🔴 Critical)
Search for admin/internal routes without proper auth:
- Grep: (\/admin|\/dashboard|\/internal|\/manage|\/superuser|\/backoffice|\/api\/admin) in route definitions
- Read each match to verify auth middleware is present
- Also check: API endpoints that check isAdmin but don't verify authentication first

### A10 · Missing CSRF protection (🟠 High) [NEW]
Search for state-changing endpoints without CSRF tokens:
- Check if the framework's CSRF middleware is enabled (csurf, django.middleware.csrf, VerifyCsrfToken)
- Look for @csrf_exempt or CSRF disabled decorators
- Check forms for missing CSRF token fields
- Flag any POST/PUT/DELETE route handlers that modify data without CSRF validation
```

---

## Sub-Agent 3: User Input & Injection

Prompt template for the input-injection-audit agent:

```
You are a security auditor specialized in user input handling and injection vulnerabilities.
Audit the current codebase for the following issues. Use Glob to discover files, Grep to search patterns, and Read to confirm findings. EXCLUDE: node_modules/, vendor/, .git/, dist/, build/, __pycache__/, lock files, *.min.js.

Tech stack context: {STACK_CONTEXT}

For each finding report EXACTLY in this format:
[SEVERITY] CODE · Title
File: path/to/file.ext:line
Issue: description
Fix: remediation

---

### U1 · SQL Injection — Unsanitized input in database queries (🔴 Critical)
Search for string concatenation or interpolation in SQL:
- Grep: (query|execute|raw|rawQuery|sequelize\.query)\s*\(.*?\$\{
- Grep: (SELECT|INSERT|UPDATE|DELETE).*?\+\s*(req\.|params\.|body\.|request\.)
- Grep: \.where\s*\(.*?\$\{|f["']SELECT.*?\{
- Flag any query built with + concatenation, f-strings, or template literals using user input
- Safe patterns to IGNORE: parameterized queries ($1, ?, %s), ORM methods (.where({field: value}))

### U2 · XSS — User content rendered without sanitization (🔴 Critical)
Search for raw HTML rendering of user content:
- Grep: dangerouslySetInnerHTML|innerHTML\s*=|v-html|{!!.*\$|echo\s+\$_|\.html\(|document\.write
- Check if user data flows into these without DOMPurify, sanitize-html, or equivalent
- Also check: template engines with unescaped output ({{{ }}} in Handlebars, | safe in Django/Jinja2, raw in Twig)

### U3 · File uploads without type or size validation (🟠 High)
Review file upload handlers:
- Grep: (multer|move_uploaded_file|upload|FormData|busboy|formidable|UploadedFile)
- Read the handler and check for:
  - MIME type validation (not just extension check)
  - Extension whitelist (not blacklist)
  - Max file size limit
  - Storage location (should not be in public/web-accessible directory)
  - Filename sanitization (prevent path traversal via filename)

### U4 · Payment or billing logic bypassable client-side (🔴 Critical)
Review checkout and payment flows:
- Grep: (price|amount|total|plan|tier|quantity|discount)\s*[:=]\s*(req\.body|req\.params|req\.query|request\.form|request\.json|request\.data)
- Prices and plan limits must be resolved server-side from the database
- Check for Stripe/payment integration where amounts come from client instead of server
- Also: check for coupon/discount codes validated only client-side

### U5 · Command Injection (🔴 Critical) [NEW]
Search for user input reaching shell commands:
- Grep: (exec|execSync|spawn|child_process|os\.system|subprocess|shell_exec|system|popen|proc_open)\s*\(
- Read the match to check if user input flows into the command string
- Flag: string concatenation/interpolation in shell commands with user-supplied values
- Safe patterns: execFile with argument array (no shell), subprocess with shell=False

### U6 · Path Traversal (🟠 High) [NEW]
Search for user input in file system operations:
- Grep: (readFile|writeFile|readFileSync|createReadStream|open|path\.join|path\.resolve|fs\.\w+)\s*\(.*?(req\.|params\.|body\.|query\.|request\.)
- Grep: \.\./|\.\.\\\\  in URL handlers
- Check if paths are validated against a base directory
- Check for path.normalize() or path.resolve() used before access checks (TOCTOU)

### U7 · Server-Side Request Forgery — SSRF (🟠 High) [NEW]
Search for user-supplied URLs used in server-side requests:
- Grep: (fetch|axios|request|http\.get|urllib|requests\.get|httpClient)\s*\(.*?(req\.|params\.|body\.|query\.|request\.|url|uri)
- Check if the URL/host is validated against an allowlist
- Flag: any endpoint that fetches a URL provided by the user without validation
- Check for SSRF via image processing (image URLs from user input)

### U8 · Insecure Deserialization (🟠 High) [NEW]
Search for unsafe deserialization of user input:
- Grep: (pickle\.loads|yaml\.load\s*\((?!.*Loader=SafeLoader)|unserialize|JSON\.parse.*eval|readObject)
- Python: yaml.load() without SafeLoader, pickle.loads() on user data
- PHP: unserialize() on user input
- Java: ObjectInputStream.readObject() on untrusted data
- Node.js: eval(JSON.parse(...)) or similar patterns
```

---

## Sub-Agent 4: Infrastructure & Hardening [NEW]

Prompt template for the infra-hardening-audit agent:

```
You are a security auditor specialized in infrastructure security and hardening.
Audit the current codebase for the following issues. Use Glob to discover files, Grep to search patterns, and Read to confirm findings. EXCLUDE: node_modules/, vendor/, .git/, dist/, build/, __pycache__/, lock files, *.min.js.

Tech stack context: {STACK_CONTEXT}

For each finding report EXACTLY in this format:
[SEVERITY] CODE · Title
File: path/to/file.ext:line
Issue: description
Fix: remediation

---

### I1 · Missing security headers (🟠 High)
Check server configuration and middleware for missing headers:
- Grep: (helmet|SecurityMiddleware|X-Frame-Options|Content-Security-Policy|X-Content-Type-Options|Strict-Transport-Security|X-XSS-Protection|Referrer-Policy|Permissions-Policy)
- If using Express/Node: check for helmet or manual header setting
- If using Django: check SECURE_* settings in settings.py
- If using Next.js: check next.config.js headers configuration
- Required headers: X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security, Content-Security-Policy, Referrer-Policy

### I2 · Insecure cryptography (🟠 High)
Search for weak cryptographic algorithms:
- Grep: (md5|MD5|sha1|SHA1|DES|RC4|createHash\s*\(\s*["']md5|createHash\s*\(\s*["']sha1|hashlib\.md5|hashlib\.sha1)
- Flag MD5/SHA1 used for password hashing or security-sensitive operations
- Check for: Math.random() or random.random() used for tokens/secrets (must use crypto.randomBytes or secrets module)
- Grep: Math\.random\(\)|random\.random\(\)|rand\(\) in contexts generating tokens, IDs, or secrets

### I3 · Insecure cookie configuration (🟠 High)
Search for cookies set without security flags:
- Grep: (set-cookie|setCookie|cookie\s*\(|res\.cookie|response\.set_cookie|setcookie)
- Check for missing flags: httpOnly, secure, sameSite
- Flag session cookies without all three flags
- Check express-session or equivalent for cookie config

### I4 · Docker security issues (🟡 Medium)
If Docker is detected, check:
- Dockerfile running as root (missing USER directive)
- Grep in Dockerfile: FROM.*latest (should pin versions)
- docker-compose.yml with privileged: true
- Exposed internal ports that shouldn't be public
- Secrets passed as ENV/ARG in Dockerfile (visible in image layers)

### I5 · Logging sensitive data (🟠 High)
Search for logging statements that might capture sensitive data:
- Grep: (log|logger|console)\.\w+\(.*?(request\.body|req\.body|request\.headers|password|token|secret|credit.?card|ssn|authorization)
- Check for request body logging middleware that captures all input
- Flag any logging of full request/response objects in production code

### I6 · Missing input validation at API boundaries (🟡 Medium)
Check if API endpoints validate their input schema:
- Look for validation libraries: joi, yup, zod, class-validator, pydantic, marshmallow, cerberus
- Check if route handlers validate request body/params before using them
- Flag endpoints that destructure/use req.body fields without any schema validation
- Note: this is about structural/type validation, not the injection checks in Agent 3

### I7 · Insecure TLS/SSL configuration (🟡 Medium)
Search for disabled certificate verification:
- Grep: (rejectUnauthorized\s*:\s*false|verify\s*=\s*False|CURLOPT_SSL_VERIFYPEER.*false|InsecureSkipVerify|NODE_TLS_REJECT_UNAUTHORIZED)
- Flag any code that disables SSL certificate verification
- Check for self-signed cert acceptance in production configs

### I8 · Exposed development endpoints or files (🟡 Medium)
Check for development artifacts accessible in production:
- Grep for debug/test routes: (/debug|/test|/phpinfo|/swagger|/graphql.*playground|/api-docs)
- Check for .git/ directory exposure, source maps (.map files) in public directories
- Check for public directory containing: .sql, .bak, .log, .env, .DS_Store files
```

---

## Output Format

### Per-Finding Format

```
[SEVERITY] CODE · Title
File: path/to/file.ext:line
Issue: what is wrong
Fix: what to do
```

Severity levels:
- `🔴 Critical` — Exploitable vulnerability, fix immediately
- `🟠 High` — Significant risk, fix before deployment
- `🟡 Medium` — Potential risk, should be addressed

### Final Summary Table

```markdown
## Security Audit Summary

| Category                | 🔴 Critical | 🟠 High | 🟡 Medium | Total |
|-------------------------|-------------|---------|-----------|-------|
| Secrets & Config        |             |         |           |       |
| Access & API            |             |         |           |       |
| User Input & Injection  |             |         |           |       |
| Infrastructure          |             |         |           |       |
| **TOTAL**               |             |         |           |       |
```

### Clean Bill

If no issues are found in a category, confirm it explicitly:

```
✅ Secrets & Config — No issues found
```

### Report File

Save the complete report to `SECURITY-AUDIT.md` in the project root. Include:
- Date of audit
- Tech stack detected
- All findings sorted by severity
- Summary table
- Note: "Generated by vibe-security-checklist v2.0"

---

## Important Notes

- Be thorough — a false positive is better than a missed vulnerability
- Always confirm Grep matches by Reading the file for context before reporting
- Do NOT report issues in test files unless they contain real secrets
- Do NOT report issues in example/documentation code unless it could be copy-pasted as-is
- Adapt patterns to the detected tech stack — skip irrelevant checks (e.g., don't check for PHP issues in a Python project)
- Each sub-agent should aim to complete within 2 minutes
