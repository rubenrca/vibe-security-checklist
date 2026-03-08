---
name: vibe-security-checklist
description: Performs a comprehensive security audit of your codebase using the Vibe Coding Security Checklist. Scans for secrets/config leaks, access & API vulnerabilities, and user input risks. Use when you want to review your project for common security issues before shipping or deploying.
license: MIT
compatibility: Designed for Claude Code. Requires Read, Grep, Glob, and Bash tools.
metadata:
  author: rubenrca
  version: "1.0"
allowed-tools: Read Grep Glob Bash
---

# Vibe Coding Security Checklist

You are performing a **security audit** of the current codebase using the Vibe Coding Security Checklist. Scan systematically, report every finding with file path and line number, and suggest a fix for each issue.

## How to run the audit

1. Use `Glob` to discover relevant source files (`.js`, `.ts`, `.jsx`, `.tsx`, `.py`, `.php`, `.env*`, `*.config.*`, `*.json`, `*.yaml`, `*.yml`)
2. Use `Grep` to search for patterns listed in each check
3. Use `Read` to inspect suspicious files in context
4. Report findings grouped by category with severity: 🔴 Critical / 🟠 High / 🟡 Medium
5. At the end, output a summary table with total issues per category

---

## 01 — SECRETS & CONFIG

### S1 · Hardcoded secrets, tokens, or API keys
Search for patterns like: `api_key`, `apikey`, `secret`, `token`, `password`, `passwd`, `private_key`, `access_key`, `client_secret` assigned to string literals.
- Grep: `(api_key|apiKey|API_KEY|secret|SECRET|token|TOKEN|password|PASSWORD)\s*[:=]\s*["'][^"'${\s]{8,}`
- Exclude: `.env.example`, `*.md`, test fixtures with obvious fake values like `"your-key-here"`
- Severity: 🔴 Critical

### S2 · Secrets leaking through logs, error messages, or API responses
Search for `console.log`, `print`, `echo`, `logger` calls that include variables named `token`, `secret`, `password`, `key`, `credential`.
- Grep: `(console\.log|print|echo|logger\.\w+)\s*\(.*?(token|secret|password|key|credential)`
- Also check error handlers that return full exception objects to the client
- Severity: 🟠 High

### S3 · Environment files committed to git
Run: `git ls-files | grep -E "^\.env"` — any `.env` file tracked by git is a finding.
Also check `.gitignore` to confirm `.env` is listed.
- Severity: 🔴 Critical

### S4 · API keys exposed client-side that should be server-only
In frontend code (`src/`, `public/`, `client/`), search for environment variables referencing secrets that are bundled into the client:
- Next.js: any `process.env.NEXT_PUBLIC_` variable containing `key`, `secret`, `token`
- React/Vite: `import.meta.env.VITE_` or `process.env.REACT_APP_` with sensitive names
- Severity: 🟠 High

### S5 · CORS too permissive
Search for CORS configuration set to `*` or accepting all origins without restriction.
- Grep: `(origin|Access-Control-Allow-Origin)\s*[:=]\s*["']\*["']`
- Also check framework-specific configs: `cors({ origin: true })`, `add_header Access-Control-Allow-Origin *`
- Severity: 🟠 High

### S6 · Dependencies with known vulnerabilities
Run: `npm audit --json 2>/dev/null | head -50` or `composer audit 2>/dev/null` depending on the stack.
Report critical and high severity advisories.
- Severity: 🟠 High

### S7 · Default credentials or example configs still present
Search for common default credentials left in config files:
- Grep: `(admin|root|password|123456|changeme|secret|example|test)\s*[:=]\s*["'](admin|root|password|123456|changeme|secret|example|test)["']`
- Check `docker-compose.yml`, `config/database.*`, `.env.example` used as actual `.env`
- Severity: 🟠 High

### S8 · Debug mode or dev tools enabled in production
Search for debug flags that should be `false` in production:
- Grep: `(DEBUG|debug|APP_DEBUG|WP_DEBUG)\s*[:=]\s*(true|True|TRUE|1|on)`
- Check `next.config.*` for `reactStrictMode` issues, PHP `display_errors = On`, Django `DEBUG = True`
- Severity: 🟡 Medium

---

## 02 — ACCESS & API

### A1 · Pages or routes accessible without proper auth
Review route definitions and check for missing auth middleware/guards:
- Search for route handlers that lack auth checks (`@login_required`, `middleware(['auth'])`, `requireAuth`, `verifyToken`, etc.)
- Look for admin or dashboard routes with no protection
- Severity: 🔴 Critical

### A2 · IDOR — Users accessing other users' data via URL ID
Search for queries that use user-supplied IDs without verifying ownership:
- Grep: `(findById|findOne|WHERE id =|\.get\(id\)|params\.id|req\.params)` then check if result is validated against `req.user.id` or session
- Pattern: `db.find({ id: req.params.id })` without `userId` filter is suspicious
- Severity: 🔴 Critical

### A3 · Tokens stored insecurely on the client
Search for `localStorage` or `sessionStorage` storing JWTs or auth tokens:
- Grep: `localStorage\.(setItem|getItem)\s*\(.*?(token|jwt|auth|session)`
- Tokens should be in `httpOnly` cookies. Flag any token stored in `localStorage`
- Severity: 🟠 High

### A4 · Login or reset flows that reveal account existence
Review login and password reset endpoints. Check if error messages differ for "user not found" vs "wrong password":
- Grep error message strings: `"User not found"`, `"Email does not exist"`, `"No account"`, `"Invalid email"`
- Both cases should return the same generic error
- Severity: 🟡 Medium

### A5 · Endpoints missing rate limiting
Check API routes for rate limiting middleware:
- Grep: look for rate limiter imports/usage (`express-rate-limit`, `ratelimiter`, `Throttle`, `@RateLimit`)
- Flag auth endpoints (`/login`, `/register`, `/reset-password`, `/forgot`) with no rate limiting
- Severity: 🟠 High

### A6 · Error responses exposing internal details
Search for error handlers that send stack traces, query strings, or internal paths to clients:
- Grep: `(err\.stack|error\.stack|exception\.message|e\.message)` in response sends
- Grep: `res\.json\(err\)` or `res\.send\(error\)` — sending raw error objects
- Severity: 🟠 High

### A7 · Endpoints returning more data than needed
Look for queries that return full model/table rows when only a few fields are needed:
- Grep: `\.select\(\*\)` or `SELECT \*` in API response contexts
- Check if user objects returned to API include `password`, `hash`, `salt`, `token`
- Severity: 🟡 Medium

### A8 · Sensitive actions without confirmation step
Review destructive endpoints (delete account, change email, transfer funds) for lack of re-authentication or confirmation:
- Grep: `(delete|destroy|remove|transfer|changeEmail|updateEmail)` route handlers
- Check if they require password confirmation or 2FA before executing
- Severity: 🟡 Medium

### A9 · Admin routes protected only by URL obscurity
Search for admin/internal routes that rely solely on a secret URL rather than auth middleware:
- Grep: `(\/admin|\/dashboard|\/internal|\/manage|\/superuser)` in route definitions — check each for auth middleware
- Severity: 🔴 Critical

---

## 03 — USER INPUT

### U1 · Unsanitized input reaching database queries (SQL Injection)
Search for string concatenation or interpolation in SQL queries:
- Grep: `(query|execute|raw)\s*\(.*?\$\{|SELECT.*?\+\s*(req\.|params\.|body\.)|\bwhere\s+\w+\s*=\s*['"]\s*\+`
- Flag any query built with `+` concatenation or template literals using user input
- Severity: 🔴 Critical

### U2 · XSS — User content rendered without sanitization
Search for places where user-supplied content is rendered as raw HTML:
- Grep: `dangerouslySetInnerHTML`, `innerHTML\s*=`, `v-html`, `{!! `, `echo \$_`
- Check if any user data flows into these without sanitization
- Severity: 🔴 Critical

### U3 · File uploads without type or size validation
Review file upload handlers for missing validation:
- Grep: `(multer|move_uploaded_file|upload|FormData)` — then read the handler
- Check: MIME type validation, extension whitelist, max file size limit
- Flag uploads that only check extension (not MIME type) or have no size limit
- Severity: 🟠 High

### U4 · Payment or billing logic bypassable client-side
Review checkout and payment flows:
- Search for price, plan, or amount values that come from the request body/params without server-side validation
- Grep: `(price|amount|total|plan|tier)\s*[:=]\s*(req\.body|req\.params|req\.query)`
- Prices and plan limits must always be resolved server-side from the database, never trusted from the client
- Severity: 🔴 Critical

---

## Output format

For each finding, report:

```
[SEVERITY] CategoryCode · Title
File: path/to/file.ts:line
Issue: what is wrong
Fix: what to do
```

At the end, output:

```
## Security Audit Summary
| Category          | Critical | High | Medium | Total |
|-------------------|----------|------|--------|-------|
| Secrets & Config  |          |      |        |       |
| Access & API      |          |      |        |       |
| User Input        |          |      |        |       |
| TOTAL             |          |      |        |       |
```

If no issues are found in a category, explicitly confirm it passed. Be thorough — a false positive is better than a missed vulnerability.
