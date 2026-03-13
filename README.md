# vibe-security-checklist

An [Agent Skill](https://agentskills.io) that performs a comprehensive security audit of your codebase using the **Vibe Coding Security Checklist**.

## What's new in v2.0

- **Sub-agent orchestration** — 4 specialized agents run in parallel for faster, deeper audits
- **Auto tech stack detection** — adapts checks to your project's language and framework
- **31 security checks** across 4 categories (up from 21 across 3)
- **New category: Infrastructure & Hardening** — security headers, crypto, Docker, TLS
- **New checks** — Command Injection, Path Traversal, SSRF, CSRF, Insecure Deserialization, and more
- **Auto-saves report** to `SECURITY-AUDIT.md`

## Architecture

```
┌───────────────────────────────────┐
│       ORCHESTRATOR                │
│  Phase 0: Detect tech stack       │
│  Phase 1: Launch 4 sub-agents     │
│  Phase 2: Consolidate & report    │
└──────────┬────────────────────────┘
           │ (parallel)
   ┌───────┼───────────┬────────────┐
   ▼       ▼           ▼            ▼
┌───────┐┌────────┐┌──────────┐┌─────────┐
│Secrets││Access  ││Input &   ││Infra &  │
│& Conf ││& API   ││Injection ││Hardening│
└───────┘└────────┘└──────────┘└─────────┘
```

## Coverage

### 01 — Secrets & Config (9 checks)
Hardcoded keys, leaked env files, client-side secrets, CORS, vulnerable deps, debug mode, default credentials, git history secrets

### 02 — Access & API (10 checks)
Missing auth, IDOR, insecure token storage, user enumeration, rate limiting, error leaks, over-fetching, missing CSRF, admin routes

### 03 — User Input & Injection (8 checks)
SQL injection, XSS, file uploads, payment bypass, command injection, path traversal, SSRF, insecure deserialization

### 04 — Infrastructure & Hardening (8 checks)
Security headers, weak crypto, cookie config, Docker security, sensitive logging, input validation, TLS/SSL, dev endpoint exposure

Each finding includes file path, line number, severity (🔴 Critical / 🟠 High / 🟡 Medium), and a suggested fix.

## Install

### Claude Code

```bash
npx skills add rubenrca/vibe-security-checklist
```

### Manual

Copy the `vibe-security-checklist/` folder into your project's `.claude/skills/` directory.

## Usage

```
/vibe-security-checklist
```

Run it from your project root to audit the entire codebase. The report is also saved to `SECURITY-AUDIT.md`.

## Supported stacks

Auto-detects and adapts checks for:
- **Node.js** — Express, Next.js, Fastify, Nest.js
- **Python** — Django, Flask, FastAPI
- **PHP** — Laravel, WordPress
- **Ruby** — Rails
- **Go**, **Rust**, **Java**
- **Docker** environments

## Checklist reference

Based on the [Vibe Coding Security Checklist](https://x.com/DeFiMinty) by @DeFiMinty, extended with additional OWASP Top 10 checks.

## License

MIT
