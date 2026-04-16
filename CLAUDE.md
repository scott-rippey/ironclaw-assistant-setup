# IronClaw Mac Mini AI Assistant Project

## What This Project Is

Planning, documentation, and build spec for setting up a dedicated M4 Mac Mini as a secure, persistent AI assistant using IronClaw (https://github.com/nearai/ironclaw). The assistant handles research, spec drafting, client prep, email communication, and background monitoring -- NOT coding.

## Key Files (public repo)

- **`README.md`** -- Public repo landing page.
- **`ironclaw-mac-mini-setup-guide-v5.1.md`** -- THE primary document. Current setup guide. When setting up the Mac Mini with Claude Code, this is the spec to follow.
- **`v5-templates/`** -- Starting templates referenced by the v5.1 guide. Copy to `~/.ironclaw/` during Phase 4. Contains `ironclaw-mac-mini.toml`, `IDENTITY.md`, `SOUL.md`, `USER.md`, `HEARTBEAT.md`, `PEOPLE-schema.md`, and a README explaining the convention.
- **`CLAUDE.md`** (this file) -- Project instructions for Claude Code during build + maintenance sessions.

## Local-Only Files (gitignored, not in public repo)

- **`archive/`** -- Prior setup guide versions (v3, v4, v4.1, v4.2, v4.3), prior features discussion docs (v0.22/0.23/0.24/0.25), and the Gemma 4 vs Qwen comparison. Kept for local reference and decision history. See `archive/README.md`.
- **`docs/SESSION_LOG.md`** -- Log of planning sessions via `/log` during development. Personal.
- **`.claude/`** -- Claude Code session state.

## Architecture Decisions (Summary -- v5.1)

- **Human-in-the-loop is a security boundary.** Agent always drafts, always asks, never sends externally without explicit approval via Discord `#approvals`. Encoded at the infrastructure level via the per-tool permission table (Phase 5.5 of the v5.1 guide). Email sends, calendar invites sent, and external DMs are hard-locked via `ApprovalRequirement::Always` and cannot be made `AlwaysAllow`.
- **`memory_write = AskEachTime` PERMANENTLY.** v0.25.0's tool permissions are per-tool-name only — no path-level granularity. Until IronClaw ships path-level permissions (or we build a custom `commitment_file_signal` wrapper), every memory write prompts for approval. This protects identity files (IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md) at the cost of silent commitment auto-capture. The security/UX tradeoff is intentional.
- **Discord for structured communication.** Private server with dedicated channels (`#general`, `#approvals`, `#research`, `#file-uploads`, `#heartbeat`, `#commitments` (v5.1 NEW), `#cost-tracking`, `#system-alerts`, `#memory`, `#router`). Per-channel notification control. Slash commands. Interactive buttons for approval flows.
- **Workspace identity files:** well-known files (IDENTITY.md, SOUL.md, USER.md, PEOPLE/, HEARTBEAT.md, MEMORY.md, commitments/) injected into every LLM call. Identity files are deletion-protected and auto-versioned by v0.25.0. The Phase 4b Identity Interview in the v5.1 guide is a ~90-min scripted intake that populates these files with real content using Claude Code.
- **Commitments system (v0.25.0):** 5 core skills installed (commitment-setup, commitment-triage, commitment-digest, decision-capture, delegation-tracker). Persona bundles (`ceo-assistant` etc.) SKIPPED — our persona is bespoke in SOUL.md. `retention_days = 365` for `commitments/resolved/`.
- **Hybrid local/API model strategy:** Qwen 3.5 27B via Ollama for 90%+ of tasks. Sonnet 4.6 via Anthropic API for quality-critical work. Haiku 4.5 as automatic fallback if Sonnet is unavailable. Gemma 4 31B is a reasonable alternative for bake-off evaluation at build time (comparison notes in `archive/gemma4-vs-qwen-comparison.md` locally).
- **Always use model aliases** (e.g., `claude-sonnet-4-6-latest`) not specific version IDs.
- **Embeddings:** OpenAI `text-embedding-3-large` (scoped API key, no generative model access).
- **Google integration via BUNDLED tools (v5.1 change):** `tools-src/gmail` and `tools-src/google-calendar` ship with IronClaw. Installed via `ironclaw tool install gmail google-calendar` or via the setup wizard. OAuth flow uses IronClaw's ephemeral callback listener at `127.0.0.1:9876` with `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` from `.env`. Desktop app OAuth client in Google Cloud Console.
- **Calendar visibility of user's calendar via Google Calendar SHARING (v5.1 change).** User shares primary calendar with `agent@yourdomain.com` at "See only free/busy (hide details)." Not an OAuth grant. Simpler, revocable with one click at Google's end, no separate secret_name.
- **Installation via Homebrew (v5.1 change):** `brew install ironclaw`. Source build is Appendix A of the v5.1 guide — for local patching only.
- **Deployment profile: `ironclaw-mac-mini.toml` inheriting from `local`.** NOT `local-sandbox`. We don't run arbitrary code. See "About Sandboxing (Two Kinds)" in the v5.1 guide for the WASM-vs-Docker distinction.
- **Engine: start on v1.** v0.25.0 ships engine v2 (Thread/Capability/CodeAct) but we start on v1 for stability. 60-day bake-off on a cloned workspace before considering v2 migration. `ENGINE_V2=false` in `.env`.
- **LLM config:** `LLM_BACKEND=ollama` for direct Ollama. `LLM_BACKEND=openai_compatible` only when pointing at the Smart Router proxy (Phase 9).
- **Ports:** 3001 gateway, 3100 Smart Router, 3200 web upload page, 9876 OAuth callback listener (ephemeral), 11434 Ollama.
- **Config precedence (v0.25.0 unified):** `DB/TOML > env > default` for ALL non-credential settings. Credentials stay env-only. **Policy: never change non-credential config via the web UI settings tab. Read-only inspection only.**
- **Backups (v5.1 expanded):** 7 days local for BOTH pg_dump AND `~/.ironclaw/` workspace tar.gz. 30 days offsite (encrypted to agent's Google Drive). Commitments, identity files, PEOPLE, and .system/ live on disk, not in Postgres — the workspace backup covers those.
- **Heartbeat:** Observation-only. Tasks defined in HEARTBEAT.md. Flags to Discord channels (`#heartbeat`, `#commitments`, `#cost-tracking`, `#system-alerts`), never acts externally. 90-day observation period.
- **Document ingestion:** Discord file uploads (`#file-uploads` channel) + web upload page (Phase 7.5). Both needed before seeding the brain.
- **Smart Router alerts:** Discord webhooks for fire-and-forget notifications to channel-specific webhook URLs (`#system-alerts`, `#cost-tracking`, `#router`). All communication encrypted via HTTPS/WSS.

## Sandboxing Posture (Important)

IronClaw has TWO sandbox layers. Don't conflate them.

- **WASM sandbox -- ALWAYS ON.** Every tool is a WASM component. Capability-based permissions, endpoint allowlisting, credential injection at the host boundary. This is the "IronClaw is secure" part. Baked in, cannot disable.
- **Docker sandbox -- OPTIONAL, WE SKIP IT.** The `local-sandbox` deployment profile. Wraps tool execution in Docker. Needed only when running arbitrary user code (shell tool, engine v2 CodeAct, coding workflows). We run neither, so we use the plain `local` profile.

**Switch to `local-sandbox` if:** we ever add a shell tool, enable engine v2 with CodeAct, or extend to coding workflows. `brew install --cask docker` and change `IRONCLAW_PROFILE`.

## What NOT to Do

- Do not use `LLM_BACKEND=openai_compatible` for Ollama directly. Use `LLM_BACKEND=ollama`.
- Do not use `text-embedding-3-small`. Always use `text-embedding-3-large`.
- Do not let the agent send emails or calendar invites without user approval.
- Do not hardcode model version IDs in config or code.
- Do not reference port 8080 for the gateway. It's 3001.
- Do not put secrets in plaintext config files. Use macOS Keychain via IronClaw's credential vault (AES-256-GCM with keychain-stored master key).
- **Do not use the IronClaw web UI settings tab (port 3001) to change non-credential config.** Read-only inspection only. All config lives in `.env`, the profile TOML, or (for credentials) macOS Keychain.
- **Do not relax `memory_write` to `AlwaysAllow`** without first building a custom wrapper tool that protects identity file paths. Security over silent UX.
- **Do not use `local-sandbox` profile** unless the switch-triggers above apply. Our setup doesn't benefit from Docker sandboxing.
- **Do not hand-roll Gmail or Calendar OAuth** — use the bundled `tools-src/gmail` and `tools-src/google-calendar` tools. This is a v5.1 correction from the v4.3 plan.
- **Do not OAuth-grant the agent access to the user's personal Google account.** The agent sees the user's calendar availability via Google Calendar SHARING (free/busy only), not OAuth. There is no second OAuth grant in v5.1.
- **Do not enable `ENGINE_V2=true`** before the 60-day retro bake-off on a cloned workspace.
- **Do not install Composio.** Data privacy — Composio routes through `backend.composio.dev`. Bundled IronClaw tools do what we need without that privacy cost.
- **Do not install persona bundles** (`ceo-assistant`, `content-creator-assistant`, `trader-assistant`). Our persona is bespoke in SOUL.md.
- **Do not commit `.env` with real values, populated identity files, or populated PEOPLE/*.md to THIS project repo.** Those go to the agent's PRIVATE GitHub repo (separate account). This repo stays template-only.

## Tech Stack

- **IronClaw** (Rust binary, v0.25.0+) -- the agent framework. Install via `brew install ironclaw`.
- **PostgreSQL 15+** with pgvector -- persistent memory and semantic search.
- **Ollama** -- local LLM inference (Qwen 3.5 27B primary, Qwen 3.5 9B for router classification).
- **Anthropic API** -- Sonnet 4.6 for quality-critical tasks, Haiku 4.5 as fallback.
- **OpenAI API** -- embeddings only (`text-embedding-3-large`).
- **Discord** -- primary communication channel (private server with structured channels, slash commands, webhooks).
- **Next.js / TypeScript** -- Smart Router proxy (Phase 9) and web upload page (Phase 7.5).
- **macOS launchd** -- process management for the router.
- **rclone + gpg** -- offsite backup pipeline (cron-driven, agent not involved).

## Repository and Build Directory Topology

**This project repo** (Mac Mini, or wherever planning happens):
- Planning docs, v5.1 setup guide, feature discussion docs.
- `v5-templates/` with profile TOML and identity file templates.
- Goes to GitHub (your main account). Public-ish per our commit rule.
- Contains NO real user data, NO secrets, NO populated workspace files.

**Mac Mini `ironclawagent` user account:**
- `~/ironclaw-setup/` (or similar) -- clone of this project repo for Claude Code to read during build and maintenance sessions.
- `~/.ironclaw/` -- live IronClaw workspace. Populated IDENTITY.md / SOUL.md / USER.md / PEOPLE/ / HEARTBEAT.md, commitments/, .system/, deployed profile TOML, real `.env`.
- Standard macOS system paths for Homebrew, Postgres, Ollama, launchd, cron.

**Agent's private GitHub repo** (separate GitHub account — `agent@yourdomain.com`):
- Backup copies of populated `~/.ironclaw/` files (identity files, PEOPLE/, HEARTBEAT.md, deployed profile TOML, `.env` template with placeholders only, allowlists, scripts).
- Commits happen during Claude Code maintenance sessions, not automatically.
- NEVER contains real `.env` values, OAuth tokens, or API keys.

**Agent's Google Drive** (accessed via rclone, cron only):
- Encrypted daily database backups and workspace tar.gz backups.
- 30-day rolling retention.
- The IronClaw agent never touches these files directly.

## CLAUDE.md Commit Rule (Scope and Exception for This Repo)

**Global rule:** "Commit CLAUDE.md and docs folder to GitHub" — applies to most projects.

**Exception for this repo:** `docs/` is gitignored here because it contains `SESSION_LOG.md` which is personal planning history. CLAUDE.md itself IS committed to the public repo. This keeps the public-facing repo clean (just CLAUDE.md, v5.1 guide, v5-templates/, README.md) while letting local planning (archive/, docs/, .claude/) stay off GitHub.

This global rule also does NOT apply to the live `~/.ironclaw/` workspace on the Mac Mini — that follows the separate selective-backup flow (agent's private GitHub repo during maintenance sessions only).

## Build Phases (v5.1)

1. Mac Mini base setup (macOS user, Homebrew, Postgres, FileVault, firewall, backup crons for DB + workspace)
2. Install IronClaw (`brew install ironclaw`)
3. Set up Ollama and pull Qwen 3.5 27B + 9B
4. Configure IronClaw with the custom profile; copy identity templates
4.5. Google Cloud Console setup (Desktop OAuth client, APIs)
4.6. Google Calendar sharing (user shares primary calendar with agent at free/busy)
4b. **Identity Interview (~90 min with Claude Code)** — populate SOUL.md, USER.md, IDENTITY.md, HEARTBEAT.md, 3-5 PEOPLE/*.md
4c. Install commitment skills
5. Discord setup; install Gmail + Calendar tools; first OAuth flow
5.5. Per-tool permission table (one-time configuration)
6. Anthropic + OpenAI API keys; offsite backup pipeline
7. Security hardening, test all approval flows and identity file write protection
7.5. Web upload page
8. First tasks, calibration, threat-model validation
9. Smart Router + custom UI
10. External knowledge base integration (skip if not applicable)

## 60-Day Retro Schedule (Ongoing Reminders)

Review with Claude Code at the 60-day mark:
1. **Engine v2 bake-off** — clone workspace, flip `ENGINE_V2=true`, run in parallel for a week.
2. **PEOPLE/ schema enforcement** — if drift causes missed context, define JSON Schema and attach via `.config`.
3. **Persona bundle decision** — after commitment signal history accumulates, decide bespoke-SOUL.md vs. authoring a custom bundle.
4. **Tool permission noise** — assess whether `memory_write = AskEachTime` is tolerable or worth building a `commitment_file_signal` wrapper tool for.

## Ongoing Watch Items (No Timer)

- IronClaw ships path-level permission granularity — relax `memory_write` when it does.
- `sanitize_auth_url()` stricter localhost behavior in a future release — revisit redirect URI if so.
