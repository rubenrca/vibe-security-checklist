# vibe-security-checklist

An [Agent Skill](https://agentskills.io) that performs a comprehensive security audit of your codebase using the **Vibe Coding Security Checklist**.

Covers 21 checks across 3 categories:

- **01 — Secrets & Config** — hardcoded keys, env files in git, CORS, debug mode, vulnerable deps
- **02 — Access & API** — missing auth, IDOR, insecure token storage, rate limiting, data exposure
- **03 — User Input** — SQL injection, XSS, file upload validation, client-side payment bypass

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

Run it from your project root to audit the entire codebase.

## Checklist reference

Based on the [Vibe Coding Security Checklist](https://x.com/DeFiMinty) by @DeFiMinty.

## License

MIT
