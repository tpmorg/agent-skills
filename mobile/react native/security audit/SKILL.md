---
name: react-native-audit
description: >
  Comprehensive security audit for React Native applications, including Expo and
  bare React Native projects. Runs sub-audits covering: secrets and environment
  variable hygiene, authentication and session management, local storage and secure
  persistence, network security and certificate handling, deep links and OAuth
  callback safety, permission usage and native surface exposure, and dependency
  supply chain risk. Use this skill whenever the user asks to audit, review, or
  harden the security of a React Native app, Expo app, mobile client, API-connected
  app, or hybrid mobile project. Also trigger when the user mentions: "security review",
  "React Native security", "Expo security", "token storage", "mobile auth audit",
  "deep link audit", "permissions audit", "hardening", "vulnerability scan",
  "secrets audit", or asks "is my app secure?" — even casually. Works on Expo-managed,
  bare React Native, or mixed React Native + native module projects.
---

# Security Audit Skill

A structured, repeatable security audit for React Native applications. Designed for
Expo-managed and bare React Native apps, but the patterns generalize to similar
JavaScript-based mobile clients.

## Overview

This skill walks through seven audit domains, producing a severity-rated findings
report with file-level references and remediation guidance. It auto-detects which
domains apply based on the actual project structure.

## Audit Domains

| Domain | Reference File | Detects via |
|---|---|---|
| Secrets & Config | `references/react-native-secrets.md` | `.env*`, `app.config.*`, `expo.extra`, config modules |
| Auth & Sessions | `references/react-native-auth.md` | Supabase, Firebase, Auth0, Clerk, custom auth code |
| Local Storage | `references/react-native-storage.md` | AsyncStorage, SecureStore, MMKV, Keychain, SQLite |
| Network Security | `references/react-native-network.md` | fetch/axios, API clients, certificate pinning libs |
| Deep Links & OAuth | `references/react-native-deep-links.md` | Linking config, Expo linking, OAuth callbacks |
| Permissions & Native Surface | `references/react-native-permissions.md` | permissions libs, `app.json`, native manifests/plists |
| Dependencies & Supply Chain | `references/react-native-deps.md` | `package.json`, lockfiles, native modules |

## Workflow

### Step 1: Detect Stack

Scan the project root to determine which audit domains apply:

```
React Native project detection:
  - package.json with react-native or expo dependency → React Native project detected
  - app.json / app.config.* / expo config → Expo app patterns
  - android/ or ios/ directory → bare or prebuilt native surface present

Domain detection:
  - .env* files, config modules, expo.extra → enable secrets audit
  - auth SDKs or auth flows in source → enable auth audit
  - AsyncStorage, SecureStore, MMKV, Keychain, SQLite → enable storage audit
  - fetch, axios, GraphQL clients, native networking config → enable network audit
  - Linking, deep links, OAuth redirects → enable deep links audit
  - permissions declarations or camera/location/etc. usage → enable permissions audit
  - package.json + lockfile → always enable dependency audit
```

If the user requests a specific sub-audit only (e.g., "just check token storage"),
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

- **Critical**: Direct path to account takeover, secret exposure, auth bypass,
  or sensitive data compromise.
- **High**: Exploitable with moderate effort or exposes meaningful user data,
  billing, or privileged behavior.
- **Medium**: Defense-in-depth gap or conditional exploit path.
- **Low**: Best practice violation with limited direct exploit path.
- **Info**: Observation or recommendation, no direct vulnerability.

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
...

## Low & Info Findings
...

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

- "check env" or "secrets audit" → Secrets & Config only
- "token storage" or "auth audit" → Auth & Sessions only
- "storage audit" or "check AsyncStorage" → Local Storage only
- "network security" or "certificate pinning" → Network Security only
- "deep link audit" or "OAuth callback review" → Deep Links & OAuth only
- "permissions audit" → Permissions & Native Surface only
- "dependency audit" or "supply chain review" → Dependencies only
- "full audit" or just "security audit" → All detected domains

## Important Notes

- This audit covers application-level security. It does not cover backend
  infrastructure, MDM policy, or app store account configuration.
- Findings are based on static analysis of source code and checked-in config.
  Deployed backend behavior and third-party service configuration may still matter.
- This is not a substitute for professional mobile penetration testing, but it
  catches the most common and impactful issues in modern React Native apps.
- When in doubt about severity, err on the side of rating higher. It's better
  to flag something that turns out to be fine than to miss a real issue.
