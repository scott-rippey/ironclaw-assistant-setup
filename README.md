# IronClaw Mac Mini Setup

Setup guide and templates for running [IronClaw](https://github.com/nearai/ironclaw)
as a secure, persistent personal AI assistant on a dedicated M4 Mac Mini.

**Current version:** v5.1 (April 15, 2026) | **IronClaw baseline:** v0.25.0+

This is a **planning and specification** repo, not a code repo. The guide is
designed for Claude Code to walk you through the build on the Mac Mini.

## What's here

| File / folder | Purpose |
|---------------|---------|
| [`ironclaw-mac-mini-setup-guide-v5.1.md`](./ironclaw-mac-mini-setup-guide-v5.1.md) | The complete setup guide. Phase-by-phase build, architecture decisions, threat model, Smart Router implementation, maintenance. **Start here.** |
| [`CLAUDE.md`](./CLAUDE.md) | Project instructions for Claude Code. Drop this into the Mac Mini working folder alongside the guide so Claude Code has full context during build and maintenance sessions. |
| [`v5-templates/`](./v5-templates/) | Starting templates copied into `~/.ironclaw/` during Phase 4: deployment profile (`ironclaw-mac-mini.toml`), workspace identity files (`IDENTITY.md`, `SOUL.md`, `USER.md`, `HEARTBEAT.md`), `PEOPLE-schema.md` convention doc. |

## Who this is for

Someone who wants a real AI teammate -- with its own identity, accounts, and
persistent memory -- running on dedicated hardware with defense-in-depth
security. Not for coding assistance (use Claude Code, Cursor, or similar for
that). For research, spec drafting, client prep, email communication, and
background monitoring.

**You'll want this if:**
- You're comfortable with terminal commands and basic server configuration
- You want full ownership and privacy of your assistant's memory
- You want to calibrate cost via hybrid local + cloud model routing
- You value human-in-the-loop as a security boundary, not a convenience

**You might not want this if:**
- You're looking for a code-writing agent (use Claude Code)
- You don't want to maintain a dedicated machine
- You prefer consumer SaaS products over self-hosted systems

## How to use this repo

1. **Read the guide** (`ironclaw-mac-mini-setup-guide-v5.1.md`) end-to-end before touching any hardware. It's written as a spec; reading through builds the mental model.
2. **On the Mac Mini:** clone this repo to the agent's user account (e.g. `~/ironclaw-setup/`), fire up Claude Code in that directory, and follow the guide with Claude Code as your build partner.
3. **Templates in `v5-templates/`** get copied into `~/.ironclaw/` during Phase 4. They are starting scaffolds. Phase 4b (Identity Interview) is a ~90-minute scripted session where Claude Code walks you through populating them with your real content.
4. **Your populated workspace files** (IDENTITY.md with your name filled in, PEOPLE/ with real contacts, etc.) should live in a *separate private repo* under your agent's GitHub account -- never committed back to this repo. This repo stays template-only.

## Adapting for your own setup

This guide is opinionated for a specific shape: dedicated M4 Mac Mini, Ollama +
Anthropic hybrid model routing, Discord as the primary interaction channel,
Google Workspace for the agent's identity. If you want different choices, the
guide's phases are structured so you can swap modules: different local model
(any Ollama-compatible), different cloud model, Slack instead of Discord,
different email provider with OAuth support.

Things not negotiable without significant rework:
- IronClaw as the agent framework
- PostgreSQL + pgvector for persistent memory
- Human-in-the-loop for external actions (architectural, not optional)

## License

Share and adapt freely. No warranty; your setup is your responsibility. IronClaw
itself is MIT / Apache 2.0 licensed.
