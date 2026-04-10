---
name: android-audit
description: >
  Comprehensive security audit for web and Android applications built with Next.js,
  Supabase, Stripe, and Kotlin/Android. Runs sub-audits covering: Supabase RLS & SQL
  injection, API route auth & middleware, Stripe webhook & billing security, environment
  variable / secrets hygiene, client-side data exposure, HTTP headers / CORS / CSP,
  Android BuildConfig secrets, Android auth & session management, local data storage,
  network security config, component/IPC exposure, Google Play Billing, and dependency
  supply chain. Use this skill whenever the user asks to audit, review, or harden the
  security of a web app, Android app, API, or full-stack project. Also trigger when the
  user mentions: "security review", "RLS check", "auth audit", "Stripe security",
  "penetration test prep", "hardening", "vulnerability scan", "security checklist",
  "APK security", "Android security", "Play Billing audit", "manifest audit",
  or asks "is my app secure?" — even casually. Works on Next.js + Supabase projects,
  Android/Kotlin apps, or both together as a full-stack audit.
---

# Security Audit Skill

A structured, repeatable security audit for full-stack applications. Designed for
Next.js + Supabase + Stripe web stacks and Kotlin/Android native clients, but the
patterns generalize to similar architectures.

## Overview

This skill walks through up to thirteen audit domains (six web, seven Android),
producing a severity-rated findings report with file-level references and remediation
guidance. It auto-detects which platform(s) are present and runs only the relevant audits.

## Audit Domains

### Web Domains

| Domain | Reference File | Detects via |
|---|---|---|
| Supabase RLS & SQL | `references/supabase-rls.md` | `supabase/migrations/`, `supabase/` dir |
| API Auth & Middleware | `references/api-auth.md` | `app/api/` or `pages/api/` dirs |
| Stripe Security | `references/stripe.md` | `stripe` in package.json |
| Env & Secrets Hygiene | `references/env-secrets.md` | `.env*` files, `next.config.*` |
| Client-Side Exposure | `references/client-exposure.md` | `"use client"` files, React components |
| Headers & CORS | `references/headers-cors.md` | `next.config.*`, `middleware.*` |

### Android Domains

| Domain | Reference File | Detects via |
|---|---|---|
| Secrets & BuildConfig | `references/android-secrets.md` | `build.gradle.kts` with `buildConfigField` |
| Auth & Sessions | `references/android-auth.md` | Supabase/Google auth in dependencies |
| Local Data Storage | `references/android-storage.md` | Room, DataStore, file operations |
| Network Security | `references/android-network.md` | OkHttp, Ktor, network_security_config.xml |
| Components & IPC | `references/android-components.md` | `AndroidManifest.xml` |
| Google Play Billing | `references/android-billing.md` | `com.android.vending.BILLING` permission |
| Dependencies & Supply Chain | `references/android-deps.md` | `build.gradle.kts`, version catalog |

## Workflow

### Step 1: Detect Stack

Scan the project root to determine which platform(s) and audit domains apply:

```
Web project detection:
  - supabase/ directory or migrations → enable RLS audit
  - app/api/ or pages/api/ → enable API auth audit
  - "stripe" in package.json dependencies → enable Stripe audit
  - .env* files → always enable env audit
  - any client components → always enable client exposure audit
  - next.config.* or middleware.* → enable headers audit

Android project detection:
  - build.gradle.kts with android {} block → Android project detected
  - AndroidManifest.xml → enable components & IPC audit
  - buildConfigField declarations → enable secrets audit
  - Supabase or Google auth in dependencies → enable auth audit
  - Room or DataStore in dependencies → enable storage audit
  - OkHttp, Retrofit, or Ktor in dependencies → enable network audit
  - com.android.vending.BILLING permission → enable billing audit
  - Always enable dependency audit for Android projects

Full-stack detection:
  - If both web and Android projects are detected (e.g., monorepo or the user
    provides both), run both sets of audits and add a cross-cutting section that
    checks consistency between the two (e.g., server-side purchase verification
    for Play Billing, shared Supabase auth assumptions, matching RLS expectations).
```

If the user requests a specific sub-audit only (e.g., "just check my RLS policies"),
run only that domain. Otherwise, run all detected domains.

Report which domains were detected and which will be audited before proceeding.

### Step 2: Run Each Sub-Audit

For each enabled domain, read the corresponding reference file from `references/`.
Follow the checklist in that file against the actual project source code.

For each finding, record:

- **Severity**: Critical / High / Medium / Low / Info
- **Location**: File path and line number (or line range)
- **Finding**: What the issue is
- **Risk**: What could go wrong if exploited
- **Remediation**: Specific fix with code example where possible

Use these severity definitions consistently:

- **Critical**: Direct path to data breach, auth bypass, or privilege escalation.
  Examples: missing RLS on a user data table, exposed service role key in client bundle,
  unsigned Stripe webhooks.
- **High**: Exploitable with moderate effort or yields significant data.
  Examples: overly permissive RLS policy, missing auth on admin API route,
  CORS wildcard on authenticated endpoints.
- **Medium**: Defense-in-depth gap or conditional exploit path.
  Examples: no rate limiting on auth endpoints, missing CSP header,
  CSRF token not validated on state-changing routes.
- **Low**: Best practice violation with limited direct exploit path.
  Examples: verbose error messages in production, missing security headers
  like X-Content-Type-Options, loose cookie flags.
- **Info**: Observation or recommendation, no direct vulnerability.
  Examples: unused env vars, deprecated auth pattern still working,
  opportunity to tighten a permissive-but-safe policy.

### Step 3: Produce the Report

Generate a markdown report with this structure:

```markdown
# Security Audit Report
**Project**: [name]
**Date**: [date]
**Domains Audited**: [list]

## Executive Summary
[2-3 sentence overview: total findings by severity, most critical issues]

## Critical & High Findings
[Each finding with full detail, remediation code]

## Medium Findings
[...]

## Low & Info Findings
[...]

## Checklist Summary
[Table: domain → pass/fail/partial for each major check]

## Recommended Next Steps
[Prioritized action items]
```

Save the report to `/mnt/user-data/outputs/security-audit-report.md` and present it.
Also give an inline summary in conversation highlighting the critical/high items.

### Step 4: Interactive Follow-Up

After presenting findings, offer to:
- Deep-dive on any specific finding
- Generate fix patches for critical/high issues
- Re-audit a specific domain after fixes are applied
- Explain the risk model for any finding

## Scoping Options

The user can narrow scope with keywords:

**Web-specific:**
- "audit RLS" or "check my RLS policies" → Supabase RLS only
- "audit API routes" or "check auth" → API auth only
- "audit Stripe" or "check webhooks" → Stripe only
- "check my env" or "secrets audit" → Env hygiene only
- "check client exposure" → Client-side only
- "check headers" or "CORS audit" → Headers only

**Android-specific:**
- "check BuildConfig" or "APK secrets" → Android secrets only
- "Android auth audit" or "check token storage" → Android auth only
- "check local storage" or "Room audit" → Android storage only
- "network security" or "certificate pinning" → Android network only
- "manifest audit" or "check exported components" → Components & IPC only
- "billing audit" or "Play Billing" → Google Play Billing only
- "dependency audit" or "CVE check" → Dependencies only

**Cross-platform:**
- "full audit" or just "security audit" → All detected domains
- "web audit" → All web domains only
- "Android audit" or "mobile audit" → All Android domains only

## Important Notes

- This audit covers application-level security. It does not cover infrastructure,
  network, or cloud provider configuration.
- Findings are based on static analysis of source code. Runtime behavior,
  deployed configuration, and third-party dependency vulnerabilities are out of scope.
- This is not a substitute for professional penetration testing, but it catches
  the most common and impactful issues in indie/small-team web apps.
- When in doubt about severity, err on the side of rating higher. It's better
  to flag something that turns out to be fine than to miss a real issue.
