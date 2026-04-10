# Agent Skills

Reusable skill definitions and supporting references for AI-assisted workflows.

## Current Contents

- `web/security audit/`: Security audit skill for web apps, focused on Next.js, Supabase, and Stripe.
- `web/security audit/SKILL.md`: Main skill definition and workflow.
- `web/security audit/*.md`: Reference checklists for API auth, Stripe security, env/secrets hygiene, client exposure, headers/CORS, and Supabase RLS.
- `web/security audit/security-audit.skill`: Packaged export of the same skill.
- `mobile/`: Placeholder for future mobile-focused skills.

## Purpose

This repository is a small, growing collection of practical agent skills that can be reused across projects. The first skill provides a structured, severity-based security review workflow for modern full-stack web apps.

## Install

### Codex

Install a skill by copying its folder into `~/.codex/skills/`:

```bash
mkdir -p ~/.codex/skills
cp -R "web/security audit" ~/.codex/skills/security-audit
```

Codex expects each skill to live in its own directory with a `SKILL.md` file at the root.

### Claude

Install a skill by copying its folder into `~/.claude/skills/`:

```bash
mkdir -p ~/.claude/skills
cp -R "web/security audit" ~/.claude/skills/security-audit
```

Claude uses the same basic layout: one folder per skill with `SKILL.md` inside.

### System-Wide / Shared Install

For a shared local source of truth, place this repo in a common location and symlink individual skills into each tool:

```bash
sudo mkdir -p /opt/agent-skills
sudo cp -R . /opt/agent-skills/repo

mkdir -p ~/.codex/skills ~/.claude/skills
ln -sfn "/opt/agent-skills/repo/web/security audit" ~/.codex/skills/security-audit
ln -sfn "/opt/agent-skills/repo/web/security audit" ~/.claude/skills/security-audit
```

This keeps the skill maintained in one place while exposing it to both clients.

## Status

Early-stage repository. Expect the layout and coverage to expand as more web and mobile skills are added.
