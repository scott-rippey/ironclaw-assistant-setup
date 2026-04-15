# IronClaw Mac Mini AI Assistant: Setup Guide v5.0

## Project Overview

**Goal:** Set up a dedicated M4 Mac Mini as a secure, persistent AI assistant using IronClaw. A proper AI teammate that handles research, spec drafting, client prep, email communication, and background monitoring with persistent memory that gets smarter over time. This is NOT a coding agent. If you want AI-assisted development, use Claude Code, Cursor, or similar tools.

The approach prioritizes security (WASM sandboxing, credential isolation), privacy (everything local, nothing leaves your machine except explicit API calls), cost efficiency (local model for most tasks, cloud API only when quality demands it), and long-term learning (semantic search over a growing knowledge base).

You'll need to be comfortable with terminal commands and basic server configuration. If you've set up a database, deployed a web app, or configured an API integration, you have the skills for this. **v5.0 simplifies installation:** IronClaw installs via Homebrew; source-build is now an appendix for the rare case of local patching.

**Why IronClaw over alternatives:**

- Persistent memory with PostgreSQL + pgvector (hybrid full-text + vector semantic search)
- Heartbeat system for proactive background work
- WASM sandbox for security (capability-based permissions, endpoint allowlisting, credential injection at host boundary, leak detection)
- AES-256-GCM encryption with macOS Keychain integration
- Model-agnostic (Anthropic API, OpenRouter, Ollama, etc.)
- Multi-channel communication (Discord, Slack, HTTP webhooks, Web UI)
- Routines engine (cron schedules, event triggers, webhook handlers)
- Identity files for consistent personality across sessions
- **Commitments system** for in-conversation obligation tracking (v0.25.0+)
- **Bundled Gmail and Google Calendar tools** with built-in OAuth (v0.25.0+)
- Rust binary, single process, no bloat
- MIT / Apache 2.0 licensed

**GitHub:** [https://github.com/nearai/ironclaw](https://github.com/nearai/ironclaw)
**Stars:** 11K+ | **Latest Release:** v0.25.0 (April 11, 2026) | Active development with multiple releases per week

---

## What's New in v5.0 (Major Deltas from v4.3)

- **Install via Homebrew** (`brew install ironclaw`). Source build is now an appendix.
- **Bundled Gmail + Google Calendar tools** replace the v4.3 hand-rolled OAuth plan. IronClaw ships `tools-src/gmail`, `tools-src/google-calendar`, and the rest of the Google suite. First-class OAuth via ephemeral listener on `127.0.0.1:9876`.
- **Google Calendar sharing** replaces the v4.3 "read-only OAuth to your calendar" approach. Share your personal calendar with `agent@yourdomain.com` at "See only free/busy." One OAuth grant, simpler, more secure.
- **Commitments system** (v0.25.0) adds 5 core skills for in-conversation obligation tracking. Complements HEARTBEAT.md (both stay).
- **Phase 4b: Identity Interview** — new ~90-minute scripted intake where Claude Code walks you through populating SOUL.md, USER.md, IDENTITY.md, PEOPLE/, HEARTBEAT.md with real content.
- **Phase 4c: Install commitments skills** — post-Identity-Interview step.
- **Custom deployment profile** (`ironclaw-mac-mini.toml`) inheriting from `local` (not `local-sandbox`). Documents our opinionated defaults.
- **Config precedence policy:** "never change non-credential config via web UI settings tab." Broader than v4.3's LLM-only warning; `DB/TOML > env > default` now applies to all non-credential settings (v0.25.0's unified config).
- **Permanent `memory_write` approval gate.** Path-level permission granularity does not yet exist in IronClaw. Until it does (or we build a custom wrapper), every memory write prompts for approval. Identity file protection beats silent UX.
- **Expanded backup strategy:** the `~/.ironclaw/` workspace directory (including identity files, PEOPLE/, commitments/, .system/) is now backed up alongside pg_dump. Commitments live on disk, not in the DB.
- **Engine v1 start, v2 bake-off at 60 days.** v0.25.0 ships a new execution engine (Thread-Capability-CodeAct). We start on v1 (default) and evaluate v2 on a cloned workspace at the 60-day retro.
- **Full per-tool permission table** (Phase 5) encoding our human-in-the-loop policy at the infrastructure level.

---

## About Sandboxing (Two Kinds -- Read This Early)

IronClaw has two distinct sandbox layers. The v5.0 guide uses one and skips the other. Understanding the difference matters for the deployment profile choice, the permission table, and what tools we can/can't run.

**1. WASM Sandbox -- ALWAYS ON.** Every IronClaw tool (Gmail, Calendar, web search, Slack, GitHub, Discord, etc.) is a WebAssembly component running inside IronClaw's `sandboxed-tool` runtime. This isolates tool code from the host: no arbitrary filesystem access, HTTP restricted to an explicit allowlist per tool, credentials injected at the host boundary so tool code never sees raw tokens. Baked into IronClaw's architecture and cannot be turned off. It is "how tools are isolated from each other and from your machine."

When this guide talks about "WASM sandbox" as a security feature, that is this layer. It is always active in our setup.

**2. Docker Sandbox -- OPTIONAL, WE SKIP IT.** An extra containerization layer that wraps tool *execution* in a Docker container. It matters when a tool itself runs *untrusted code* -- for example:
- A `shell` tool that executes arbitrary shell commands
- Engine v2's CodeAct primitive, where the LLM writes Python that runs inside IronClaw's embedded Monty VM
- Any tool we might add later that runs user-supplied scripts or code

IronClaw exposes this as the `local-sandbox` deployment profile. The alternative is the plain `local` profile (what we use) without Docker.

**Why we use `local` (not `local-sandbox`):** our threat model is API calls + workspace I/O. Every tool we run (Gmail, Calendar, Discord, web search, memory_read/write) is an API call or disk read/write -- none run arbitrary user code. The permission table in Phase 5.5 explicitly **Disables** the shell tool (not even offered to the LLM). We start on engine v1 (no CodeAct). Docker adds a daemon, a license nag, and maintenance burden without protecting anything in our setup.

**When to switch to `local-sandbox` (documented for future):**
- We add a `shell` tool or any tool that runs user-supplied code
- We enable engine v2 with CodeAct (Python VM execution)
- We extend the project to coding workflows
- In all three cases: `brew install --cask docker`, change `IRONCLAW_PROFILE=local-sandbox`, review `SANDBOX_*` config

**TL;DR on security posture:** we are fully sandboxed for our threat model because layer 1 (WASM) is always on. Layer 2 (Docker) is defense-in-depth against code execution we don't do. Adding layer 2 now would cost maintenance without improving security.

---

## Why NOT These Alternatives

- **OpenClaw** -- Security nightmare. Estimated 430K+ lines of TypeScript (some sources report 500-600K). Plaintext credentials. Community skills with hidden prompt injections. Known CVEs. Creator left for OpenAI. For comparison, IronClaw achieves more with ~25K lines of Rust.
- **NanoClaw** -- Great security (container isolation), but memory is just a flat CLAUDE.md file per group. No semantic search, no real learning over time. Too thin for what we need.
- **ZeroClaw** -- Lean Rust binary, good security defaults, but no persistent memory/vector search layer yet.
- **Nanobot** -- 4K lines of Python. Great for learning agent patterns, too bare-bones for a real assistant.
- **Perplexity Comet** -- Consumer product, vendor lock-in, privacy concerns, currently in legal fights (Amazon injunction). Not self-hosted.
- **Composio** -- Provides 250+ third-party app integrations, but data transits `backend.composio.dev`. For a personal AI assistant handling business email and client correspondence, routing through third-party SaaS middleware is a non-starter on privacy grounds alone. IronClaw's bundled tools do what we need without that privacy cost.

---

## TOS and Legal Compliance

**The rules are clear as of February 2026:**

- **OAuth tokens from Claude Free/Pro/Max subscriptions CANNOT be used in any third-party tool, including the Agent SDK.** This is explicitly stated in Anthropic's updated Legal and Compliance documentation.
- **API keys under the Commercial Terms are the clean path.** No automation restrictions, no ambiguity.
- **IronClaw uses direct Anthropic API calls, not Claude Code CLI or OAuth tokens.** You provide an API key, IronClaw makes standard Messages API calls. Fully compliant.
- **Running Claude Code CLI itself on your own machine (including a Mac Mini) is still allowed.** It's Anthropic's own product. But IronClaw doesn't use Claude Code. It has its own agent loop.

**Bottom line:** Anthropic API key in a dedicated Workspace = fully legit. No gray areas.

---

## Cost Control Strategy

### Anthropic Console Setup

Create a dedicated Workspace in the Anthropic Console for the agent's API usage. This keeps spend isolated and trackable.

**Recommended Anthropic Console configuration:**
- Create a dedicated Workspace called "Mac Mini Agent" (or similar)
- Set spend limit based on your comfort level ($50/month is a reasonable starting point)
- Set email notification at 50% of your spend limit
- Monitor via the daily cost digest from the heartbeat (see Heartbeat Configuration) and the Anthropic Console Usage page

The Smart Router (Phase 9) also tracks all routing decisions and API token usage locally.

### Realistic Cost Estimates

**Your use case (discrete tasks, not coding loops):**

- **Quick research summary:** $0.05 - $0.15
- **Detailed spec draft (multi-step):** $0.30 - $0.50
- **Client prep one-pager:** $0.15 - $0.30
- **Heartbeat tick (brief check + update):** $0.01 - $0.03

**Monthly projections (hybrid local + API with 27B local model):**

- **Light** (3-5 tasks/day, mostly local): 90% local / 10% API -- $3 - $10/month
- **Moderate** (10+ tasks/day, mixed): 85% local / 15% API -- $10 - $20/month
- **Heavy** (lots of polished deliverables): 70% local / 30% API -- $20 - $40/month

Note: With Qwen 3.5 27B locally, the API percentage is lower than it would be with a 14B model. The 27B handles most drafting and research at a quality level that doesn't need cloud polish.

### Why Your Costs Stay Low

Unlike coding agents that re-inject entire codebases on every turn (40 turns = 40 copies of every file), your tasks are short conversations (5-15 turns) with modest context. No extended thinking burning output tokens. No massive file trees. Each task finishes and the conversation closes.

### Model Strategy: Hybrid Local + API

**Hardware:** M4, 32GB unified memory (~24GB available for models after macOS overhead)

**Primary: Qwen 3.5 27B via Ollama (free, private, no rate limits)**

With ~24GB available for model inference, Qwen 3.5 27B (Q4_K_M) is the best local model that fits. At ~17GB on disk, it leaves comfortable headroom for context and runs at strong token-per-second throughput on M4. The 3.5 generation brings a newer architecture, a 256K context window, and native multimodal input (text + image) -- meaning the agent can ingest screenshots, PDFs with figures, and scanned documents directly. Despite fewer raw parameters than Qwen 3 32B, Qwen 3.5 27B matches or beats it on research, drafting, and analysis thanks to improved training and architecture. Google's Gemma 4 31B is a reasonable alternative worth bake-off evaluation at build time if you want to comparison-shop; both fit comfortably in the 32GB envelope.

**Context window safety:** If the model hits its context boundary mid-tool-call, IronClaw automatically discards the truncated call rather than attempting to execute a half-formed operation. Framework-level, no config needed.

Install:
```bash
brew install ollama
ollama serve
ollama pull qwen3.5:27b
```

Optionally pull a fast lightweight model for the Smart Router's Tier 3 classifier:
```bash
ollama pull qwen3.5:9b
```

IronClaw config for local (this points directly at Ollama; Phase 9 changes this to point at the Smart Router instead):
```
LLM_BACKEND=ollama
OLLAMA_MODEL=qwen3.5:27b
# OLLAMA_BASE_URL=http://localhost:11434   # default, no need to set
```

**Secondary: Anthropic API (for quality-critical tasks)**

For polished client proposals, complex multi-step reasoning, or anything where quality is the priority over cost.

- **Sonnet 4.6** (`claude-sonnet-4-6-latest`) -- $3 / $15 per 1M tokens. Primary API model.
- **Haiku 4.5** (`claude-haiku-4-5-latest`) -- $1 / $5 per 1M tokens. Automatic fallback if Sonnet is unavailable.

**Always use model aliases** (e.g., `claude-sonnet-4-6-latest`) instead of specific version IDs. Specific versions get deprecated and will break the agent.

IronClaw config for API fallback:
```
LLM_BACKEND=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

**Embedding Model: OpenAI `text-embedding-3-large`**

Use OpenAI's `text-embedding-3-large` (3072 dimensions), not a local embedding model. The quality gap matters for catching semantic connections across different phrasings. Cost is negligible ($0.13 per million tokens). Create a dedicated OpenAI API key scoped to `text-embedding-3-large` ONLY -- if leaked, the worst someone can do is generate embeddings.

**Why this works:**
- Most daily tasks run locally at zero cost
- Only the small percentage needing polish or deep reasoning go through the API
- Monthly API spend realistically $3-20 depending on usage
- The persistent memory (pgvector) learns regardless of which model answered

**Additional cost levers for API usage:**
- Prompt caching (90% savings on repeated context)
- Batch API (50% discount for non-urgent tasks with 24-hour turnaround). Use the `/batch` Discord slash command.
- OpenRouter as a middle layer for per-key budget controls

---

## The "Siloed Teammate" Architecture

### Core Principle

The agent is a teammate with its own identity and accounts. It does NOT get access to your personal credentials, accounts, or data (with one controlled exception: the agent sees your availability via a Google Calendar share at "free/busy only" permission -- NOT an OAuth grant to your account). You interact through defined interfaces.

### Human-in-the-Loop: A Security Boundary

This is not just a preference. It's a hard architectural constraint enforced in the agent's `IDENTITY.md` and in the **per-tool permission table** (Phase 5, new in v5.0).

**What the agent does autonomously (no approval needed):**
- Research a topic
- Read the agent's own Gmail inbox (including anything you forward to `agent@yourdomain.com`)
- Check your calendar availability (via Google Calendar free/busy sharing)
- Read workspace memory (`memory_read`, `memory_tree`)

**What the agent drafts but needs your approval to send (preview in Discord #approvals first):**
- Email drafts (created in agent's Gmail, posted to #approvals with approval buttons, sent only after you click Approve)
- Calendar invite proposals (checks your availability, proposes a time, sent only after you approve)

**What always requires your explicit command:**
- Pushing content to an external database
- Any action that affects systems outside IronClaw

**What the agent never does, even with approval:**
- Heartbeat taking any external action (observation and Discord notification only, routed to #heartbeat channel)

**What prompts for approval every time (never "AlwaysAllow"):**
- Any `memory_write` (see Identity File Protection below)
- Routine/skill installation or modification
- Tool permission changes

The agent always drafts, always asks, never sends externally without explicit approval.

**Why approval buttons matter beyond UX:** Discord's interactive buttons (Approve / Edit / Reject) deliver a structured interaction event with a known action — no parsing, no guessing, no edge cases where a casual reply gets misinterpreted as an approval. The Edit button auto-creates a thread for contained revision, and the agent knows exactly what it's iterating on. Structured interaction events beat natural language parsing at a security boundary.

### Identity File Protection (v5.0)

**The problem:** The agent writes to its workspace via `memory_write` for commitments, MEMORY.md updates, PEOPLE/ additions, and more. v0.25.0's permission system is per-tool-name — we cannot grant `memory_write` different permissions for `/IDENTITY.md` vs `/commitments/*.md`. And `.config` metadata does not support an approval flag.

**Our decision:** `memory_write = AskEachTime` permanently. Every memory write prompts for approval. Commitment captures prompt. PEOPLE/ updates prompt. MEMORY.md appends prompt.

**Why this trade-off:**
- Identity files (IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md) define the agent's behavioral contract. If silently overwritten via prompt injection, they could weaken human-in-the-loop boundaries.
- Auto-versioning (v0.25.0's `memory_document_versions` table) would let us recover from a bad write, but we might not notice for days. Approval at write time is better than detection after the fact.
- Noise is actually useful for the first 60 days of calibration — you see everything the agent tries to write.
- This is **permanent** until either IronClaw ships path-level permission granularity OR we build a custom `commitment_file_signal` wrapper tool that exempts `commitments/` writes only.

**What DOES protect identity files automatically:**
- **Deletion protection** (v0.25.0 #1723): `is_identity_document()` blocks any attempt to delete IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md, MEMORY.md, AGENTS.md, TOOLS.md, BOOTSTRAP.md during hygiene cleanup.
- **Auto-versioning** (v0.25.0 #1723): every `write()`, `append()`, `patch()` saves prior content to the `memory_document_versions` table, hash-deduplicated. `memory_read` with `version`/`list_versions` params for recovery.
- **Credential pattern + sensitive path blocklist** (v0.25.0 #1675): blocks reads/writes on `~/.env`, `~/.ssh/`, `~/.aws/`, etc.

### Agent Identity Setup

Create dedicated accounts for the agent:

- [ ] **Anthropic API Key** in its own Workspace with spend cap
- [ ] **Free GitHub account** (tied to agent's email, e.g., `agent@yourdomain.com`) for version-controlled configuration backup. You commit via Claude Code during maintenance sessions.
- [ ] **Google Workspace / Gmail account** (see details below)
- [ ] **Discord Application + Bot** (via Discord Developer Portal)
- [ ] **macOS user account** on the Mac Mini (not your personal account)
- [ ] **OpenAI API key** scoped to `text-embedding-3-large` only

### Agent Google Account Setup

Give the agent its own Google identity. This is the cornerstone of the "teammate, not you" model.

**Account:** `agent@yourdomain.com` (on a domain you own), or a free Gmail like `yourname.agent@gmail.com` if you want to keep it simple to start.

**What to enable on the agent's Google account:**
- [ ] **Gmail** -- the agent can draft emails for your review and send them after you approve via Discord #approvals
- [ ] **Google Calendar** -- the agent can create invites on ITS calendar and propose them to you
- [ ] **Google Drive** -- used for encrypted offsite database backups (automated via cron, not the agent itself)

**What to enable via Google Calendar sharing (NOT a second OAuth grant):**
- [ ] **Your personal calendar, shared with the agent's account at "See only free/busy (hide details)"** -- the agent sees your availability through Google's native delegation, not through an OAuth grant to your account. Simpler, more secure, and revocable with one click in Google Calendar settings. This replaces v4.3's "read-only OAuth to your calendar" plan.

**How the connection works:** IronClaw ships Gmail and Google Calendar tools with built-in OAuth. The agent authenticates to the agent's Google account via one OAuth grant. Through your calendar share, it sees your free/busy as if it were its own calendar. IronClaw's credential vault stores tokens encrypted (AES-256-GCM) in the `secrets` table; the master encryption key lives in macOS Keychain.

**What this enables:**
- Agent drafts a spec email from `agent@yourdomain.com`, posts it to Discord #approvals, you click Approve and it sends
- Agent checks your calendar, proposes a follow-up invite at a time you're free, you accept or adjust
- If someone replies to the agent's email, IronClaw can process that reply through the agent loop
- Everything is transparent: recipients see it's from your AI assistant, not from you pretending to be human

**What this does NOT enable:**
- The agent cannot read YOUR email
- The agent cannot modify YOUR calendar (free/busy read-only via calendar sharing, no OAuth grant)
- The agent cannot browse YOUR Google Drive
- The agent cannot send any email or calendar invite without your explicit approval via Discord #approvals

**Google Workspace tip:** If you have a Workspace domain, creating a user like `agent@yourdomain.com` costs ~$7/month. Worth it for professional appearance when the agent emails clients on your behalf.

### What the Agent CAN Do

- Send you messages, documents, summaries via Discord (routed to appropriate channels)
- Draft emails from its own Google account (you approve before sending via #approvals)
- Check your calendar availability and propose calendar invites (you approve before sending)
- Research topics on the web
- Draft specs, one-pagers, client prep docs
- Ingest documents you send via Discord file uploads (#file-uploads channel) or the web upload page
- Run scheduled background monitoring tasks (heartbeat, observation only)
- Build context and memory over time in its local database
- Track commitments, decisions, delegations, and follow-ups extracted from conversation (v0.25.0 commitments system)
- Surface conflicting information and offer to deprecate stale entries

### What the Agent CANNOT Do

- Access your personal email or iCloud
- Modify your calendar (free/busy read-only via calendar sharing)
- Send any email or calendar invite without your explicit approval
- Write to any identity file (IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md, MEMORY.md) without your per-write approval
- Use your social media accounts
- Make purchases or financial transactions
- Access your personal files outside its designated workspace
- Talk to any host not on the explicit allowlist
- Send emails as you (it always sends as its own identity)
- Write production code (you use Claude Code for that)
- Execute arbitrary shell commands (the `shell` tool is **Disabled** in our permission table — not even offered to the LLM)

---

## Threat Model

Before building anything, document what could go wrong and how each risk is handled.

### Why Context Rot Is Not a Threat Here

Other agent frameworks (particularly OpenClaw) suffer from "context rot" where agent instructions get lost as conversation history grows and older system prompts get compressed. The community fix is to "re-inject your rules every run" as a band-aid. IronClaw's workspace identity files (IDENTITY.md, SOUL.md, USER.md, etc.) are structurally injected into every LLM call via `workspace.system_prompt()`. They're re-applied fresh every time. The database is injected as context alongside, but never replaces them.

### Threats and Mitigations

**1. Credential leak via WASM tool**
- **Risk:** A tool in the sandbox attempts to exfiltrate API keys or OAuth tokens.
- **Likelihood:** Low | **Impact:** High
- **Mitigation:** IronClaw's credential injection stores secrets encrypted (AES-256-GCM) in the `secrets` table with the master key in macOS Keychain. Credentials are injected at the host boundary, never exposed to tool code. Endpoint allowlist restricts which hosts tools can contact. Leak detection catches credential patterns in output (v0.25.0 #1675 expanded this to cover OpenRouter, Anthropic OAuth, Telegram, Groq patterns plus sensitive paths: `~/.env`, `~/.ssh/`, `~/.aws/`, `~/.netrc`, `~/.pgpass`, `~/.npmrc`, `~/.docker/`, `~/.kube/`, `~/.gnupg/`, `/etc/shadow`).
- **Validate (Phase 8):** Attempt to access credentials from a test WASM tool. Verify leak detection and allowlist blocking.

**2. RAG poisoning via ingested documents**
- **Risk:** A document contains hidden instructions that alter agent behavior when retrieved via semantic search.
- **Likelihood:** Low | **Impact:** Medium
- **Mitigation:** Workspace identity files (IDENTITY.md, SOUL.md) are injected at higher priority than retrieved context. Retrieved memories are context, not instructions. Human-in-the-loop means the agent can't act on bad context without your approval. Most ingested docs are user-created.
- **Validate (Phase 8):** Create a test document with prompt injection. Ingest it, query related topics, verify IDENTITY.md boundaries hold.

**3. Heartbeat acts on stale context**
- **Risk:** Heartbeat retrieves outdated info and flags something incorrect. User acts on it without verifying.
- **Likelihood:** Medium | **Impact:** Medium
- **Mitigation:** 90-day read-only period. Heartbeat flags to Discord only (#heartbeat channel), never acts externally. Conflicting context surfacing with `/memory` commands. Weekly log review for first month.
- **Validate (Phase 8):** Seed contradictory info, trigger heartbeat, verify it surfaces both entries with dates.

**4. Google OAuth scope creep**
- **Risk:** OAuth credential has broader scopes than needed.
- **Likelihood:** Low | **Impact:** High
- **Mitigation:** The bundled Gmail/Calendar tools declare scopes in their capabilities.json manifest. Review after install; tighten if the defaults are broader than `gmail.send` + `gmail.readonly` + `calendar.events` + `calendar.readonly`. No Drive scope (not needed by agent; Drive is only used by cron for encrypted backups via rclone).
- **Validate (Phase 7):** Document scopes, test that out-of-scope API calls fail, verify agent can't modify your calendar.

**5. WASM tool supply chain risk**
- **Risk:** Third-party tool contains malicious code.
- **Likelihood:** Low | **Impact:** Medium
- **Mitigation:** Prefer first-party IronClaw tools (Gmail, Calendar, Slack, GitHub etc.). Audit source before installing anything else. Minimal permissions. Endpoint allowlist blocks unauthorized hosts.
- **Validate (Phase 7):** Review allowlist, verify blocking of non-listed hosts.

**6. Database compromise**
- **Risk:** Attacker accesses PostgreSQL and reads business context.
- **Likelihood:** Very Low | **Impact:** High
- **Mitigation:** Localhost only. Separate macOS user. FileVault disk encryption. Strong database password.
- **Validate (Phase 1):** Verify `listen_addresses = 'localhost'`. Verify FileVault. **(Phase 7):** Check macOS user permissions.

**7. Identity file overwrite via prompt injection (v5.0)**
- **Risk:** A prompt injection attack convinces the agent to rewrite IDENTITY.md / SOUL.md / HEARTBEAT.md to weaken human-in-the-loop boundaries.
- **Likelihood:** Low | **Impact:** High
- **Mitigation:** `memory_write = AskEachTime` permanently (see Identity File Protection). Every identity file write prompts for approval. Auto-versioning means even an approved-then-regretted write is recoverable. Deletion of identity files is blocked entirely by v0.25.0's `is_identity_document()` check.
- **Validate (Phase 8):** Ask the agent to "update IDENTITY.md with the following..." -- verify it prompts for approval before writing.

### API Key Scoping

- **Anthropic API Key** -- Scoped to a dedicated Workspace with spend cap. Max exposure: your spend limit.
- **OpenAI Key (embeddings)** -- Project scoped to `text-embedding-3-large` only. Minimal cost exposure if leaked.
- **Google OAuth** -- Agent's account only: Gmail send/readonly, Calendar events/readonly. If leaked: attacker can draft/send from agent's email and create events on agent's calendar. Cannot access your personal account. Revoke and rotate immediately.
- **Google Calendar sharing** -- Your personal calendar shared with `agent@yourdomain.com` at "free/busy only." Revocable with one click in Google Calendar settings. No tokens involved; sharing is at Google's account level.

---

## Prerequisites

### Hardware
- **M4 Mac Mini, 32GB unified memory, 512GB SSD** (~$1,199)
- Internet connection
- External display for initial setup (can go headless after)

**Why 32GB M4 over 24GB M4 Pro:** For local LLM inference, RAM determines which models fit in memory. 32GB gives ~24GB available for models after macOS overhead. The M4 Pro at 24GB only gives 16-18GB.

### Software Dependencies (v5.0 -- simpler than v4.3)
- **Homebrew** (package manager for macOS)
- **PostgreSQL 15+** with pgvector extension (via Homebrew)
- **Ollama** for local model inference (via Homebrew)
- **IronClaw** (via Homebrew -- no source build needed in v5.0)

Rust toolchain is NOT required for v5.0 install. It was required in v4.3 because we built from source. Appendix A covers source-build only for users who need to patch IronClaw locally.

---

## How the Agent Learns

The PostgreSQL + pgvector database is the brain. Every interaction gets stored and indexed both as full-text and as vector embeddings. This means:

- **Direct memory:** It remembers what you told it. "We quoted Client X at $55K with three tiers of maintenance" is stored and retrievable.
- **Semantic retrieval:** Finds relevant context even when you don't use the exact same words.
- **Cross-referencing:** After feeding it project notes, client briefs, and research tasks, it connects dots across them.
- **Correction learning:** When you correct something, the corrected context outweighs the old one in retrieval. The agent will offer to deprecate the old entry.

The model is stateless. The database is the brain. **Back it up.** (See Backup Strategy.)

### Built-in Chunking and Embedding Pipeline

IronClaw handles document chunking automatically:
- **Chunk size:** 800 words per chunk (configurable)
- **Overlap:** 15% between consecutive chunks
- **Minimum fragment size:** 50 words
- **Embedding generation:** Asynchronous, with a 10,000-entry LRU cache
- **Hybrid search:** Combines full-text search with vector similarity via Reciprocal Rank Fusion (RRF)

When you send the agent content, it writes to the workspace via `memory_write` (prompting for approval per our permission policy), chunking and embedding happen automatically.

### Context Hygiene

- When retrieving context, the agent surfaces related entries sorted by date
- Conflicting entries on the same topic are presented with timestamps so you can see the contradiction
- The `/memory forget` Discord slash command lets you deprecate stale entries
- When you correct information, the agent offers to deprecate the old entry

### How Learning Affects Model Routing Over Time

**Short answer:** Yes, significantly, but it never goes to zero.

**Why it improves:** Most tasks that feel like they need a smarter model actually need better context. When the database is empty, the local model is working from just your prompt. After consistent use, the same request pulls your proposal template, your pricing framework, three similar past proposals, etc. — the local model has everything it needs.

**The practical shift:**
- **First few sessions** (empty database): ~80/20 local/API
- **After seeding core business context:** ~90/10
- **After a month of real usage:** ~95/5
- **Steady state:** ~95-98/2-5

**Why it doesn't go to zero:** The local model's raw intelligence is a fixed ceiling. Better context narrows the gap but doesn't close it entirely. Nuanced strategic analysis and polished client-facing prose still benefit from Sonnet.

---

## Workspace Identity Files (System Prompt)

IronClaw uses a set of well-known workspace files that are automatically injected into every LLM call as the system prompt. Multiple markdown files in the workspace, each serving a distinct purpose. Re-injected fresh every call, never compressed or lost.

Templates are in the `v5-templates/` directory of this project repo. **They are starting templates -- Phase 4b (Identity Interview) walks you through populating them with real content.**

### Well-Known File Structure

- `IDENTITY.md` -- Who the agent is, its role, and hard behavioral rules
- `SOUL.md` -- The agent's personality, values, and communication style
- `USER.md` -- Information about you (the user) that the agent should know
- `PEOPLE/` -- Structured contact files, one per person
- `MEMORY.md` -- Long-term curated knowledge (the agent appends here over time)
- `HEARTBEAT.md` -- Periodic task checklist (what the heartbeat does each tick)
- `commitments/` -- Markdown files tracking obligations, decisions, delegations (v0.25.0 commitments system; auto-populated)
- `AGENTS.md` -- Multi-agent coordination (skip unless running multiple agents)
- `daily/` -- Daily log directory (auto-populated by the agent)
- `.system/` -- System state (settings, extensions, skills state; hidden, managed by IronClaw)

All of these (except `.system/`) are editable markdown files. Changes take effect immediately on the next interaction. **Identity documents (IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md, MEMORY.md, AGENTS.md, TOOLS.md, BOOTSTRAP.md) are protected from deletion by IronClaw.**

### PEOPLE/ Directory (Contact Identity Files)

The agent has IDENTITY.md and SOUL.md. You have USER.md. Key contacts each get their own structured file.

**Structure:** `PEOPLE/firstname-lastname.md` (one file per person)

```markdown
# [Full Name]

## Core Info
- Role / title: [e.g., "Partner at Groundwork AI"]
- Company: [Company name]
- Relationship: [e.g., "Business partner", "Client", "Collaborator"]
- Email: [if known]
- Communication preferences: [e.g., "Prefers short emails", "Likes detailed specs"]

## Context
[Key context the agent should know when preparing for interactions with this person.]

## Active Projects
[Current projects or proposals involving this person.]
```

**Why this matters:** Semantic memories attach to these anchors. When the agent preps you for a meeting with a client, it pulls from their people file for stable context and from memory search for recent interactions.

**v5.0 note -- schemas deferred:** v0.25.0 supports JSON Schema enforcement for workspace documents, but we ship v5.0 schemaless. Templates serve as convention. Revisit at 60-day retro if PEOPLE/ drift causes the agent to miss context — at that point, define a JSON Schema and attach via `.config`.

### Commitments Directory (v0.25.0 -- NEW)

`commitments/` is populated automatically by the `commitment-triage` skill (installed in Phase 4c). The agent detects obligation language during conversation ("I'll review this by Friday", "need to call the dentist") and files a signal to `commitments/open/`. Resolved commitments move to `commitments/resolved/`.

**Retention:** `retention_days = 365` for `commitments/resolved/`. Markdown files are tiny; 1 year preserves project-cycle recall without disk pressure.

**Approval:** every commitment signal capture prompts for approval (because `memory_write = AskEachTime`). This is noisier than the default silent-capture design from PR #1736 -- see Identity File Protection above for the rationale. Revisit if/when path-level permissions are available.

### Templates

Starting templates for IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md live in `v5-templates/` in this project repo. They are scaffolds with placeholders. Phase 4b (Identity Interview) populates them.

Abridged IDENTITY.md template shown here as a reference for the security posture the Interview must preserve:

```markdown
# Identity

You are the AI assistant for [Your Name] at [Your Business].
Your name is [pick a name or just "Agent"].
You communicate from your own accounts, never impersonating [Your Name].
Your email is agent@yourdomain.com.

# Role

You are a business assistant and research partner, NOT a coding agent.
[Your Name] uses Claude Code for development work.

# Human-in-the-Loop (ENFORCED)

- You NEVER send an email without [Your Name]'s explicit approval via Discord (#approvals)
- You NEVER send a calendar invite without [Your Name]'s explicit approval via Discord (#approvals)
- You always draft first, post to #approvals with an approval embed, and wait for [Your Name] to click Approve

# Boundaries

- You can SEE [Your Name]'s calendar availability (via Google Calendar sharing at free/busy-only permission)
- You CANNOT modify [Your Name]'s calendar
- You do NOT have access to [Your Name]'s personal email, Drive, or iCloud
- You ONLY send from your own accounts
- You do NOT make purchases or financial commitments
- You do NOT write production code ([Your Name] uses Claude Code for that)
- You do NOT execute shell commands (this capability is Disabled in the tool permission table)

# Tools

- Gmail (your account): Draft emails, read replies (agent's inbox) (send only after [Your Name] approves via #approvals)
- Google Calendar (your account): Create and propose invites (send only after approval)
- Google Calendar availability (via sharing): See [Your Name]'s free/busy
- Web search: Research tasks
- Workspace memory: Always check before asking [Your Name] to repeat context
- Commitments: Auto-tracked from conversation (captures prompt for approval)
```

---

## Setup Steps

### Phase 1: Mac Mini Base Setup

1. **Fresh macOS setup** on the Mac Mini
2. **Create a dedicated macOS user account** for the agent (e.g., "ironclawagent"). This is the only account we use from this point on.
3. **Prevent sleep:** Install Amphetamine from the Mac App Store and configure a persistent session with "Start session at launch" enabled. Add to Login Items. Alternatively, configure System Settings > Energy > "Prevent automatic sleeping when the display is off."
4. **Enable auto-restart:** System Settings > General > Login Items & Extensions > "Start up automatically after a power failure"
5. **Enable FileVault** (full-disk encryption) in System Settings > Privacy & Security
6. **Enable Firewall:** System Settings > Network > Firewall: ON. Options: Stealth mode ON. Allow built-in and signed software.
7. **Install Homebrew:**
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
8. **Install PostgreSQL with pgvector:**
   ```bash
   brew install postgresql@15
   brew services start postgresql@15
   brew install pgvector
   ```
9. **Create the database:**
   ```bash
   createdb ironclaw
   psql ironclaw -c "CREATE EXTENSION IF NOT EXISTS vector;"
   ```
10. **Verify PostgreSQL is localhost-only:** Check `postgresql.conf` for `listen_addresses = 'localhost'`
11. **Set up the database backup directory and cron job:**
    ```bash
    mkdir -p /Users/ironclawagent/backups/db

    # Add to crontab (crontab -e):
    # Daily pg_dump at 2am
    0 2 * * * pg_dump ironclaw | gzip > /Users/ironclawagent/backups/db/ironclaw-$(date +\%Y\%m\%d).sql.gz
    # Clean up local backups older than 7 days
    0 3 * * * find /Users/ironclawagent/backups/db -name "ironclaw-*.sql.gz" -mtime +7 -delete
    ```
12. **Set up the workspace backup directory and cron job (v5.0 -- NEW):**
    ```bash
    mkdir -p /Users/ironclawagent/backups/workspace

    # Add to crontab (crontab -e):
    # Daily workspace snapshot at 2:15am (after pg_dump)
    15 2 * * * tar czf /Users/ironclawagent/backups/workspace/ironclaw-workspace-$(date +\%Y\%m\%d).tar.gz -C /Users/ironclawagent .ironclaw
    # Clean up local workspace backups older than 7 days
    15 3 * * * find /Users/ironclawagent/backups/workspace -name "ironclaw-workspace-*.tar.gz" -mtime +7 -delete
    ```
    Rationale: commitments/, identity files, PEOPLE/, and .system/ live on disk in `~/.ironclaw/`, not in Postgres. pg_dump alone would lose them if the disk fails.

### Phase 2: Install IronClaw (v5.0 -- simpler)

```bash
brew install ironclaw
```

That's it. IronClaw is installed and ready to configure.

**Verify install:**
```bash
ironclaw --version   # should report v0.25.0+
```

If you need to patch IronClaw locally (rare), see Appendix A for source-build instructions.

### Phase 3: Set Up Local Model via Ollama

1. **Install and start Ollama:**
   ```bash
   brew install ollama
   ollama serve
   ```
2. **Pull your primary local model:**
   ```bash
   ollama pull qwen3.5:27b
   ```
   This will download ~17GB. Test it:
   ```bash
   ollama run qwen3.5:27b "Summarize the key benefits of a multi-tenant SaaS architecture"
   ```
3. **Pull the fast lightweight model for the Smart Router classifier (Phase 9):**
   ```bash
   ollama pull qwen3.5:9b
   ```
4. **Verify the local API is running:**
   ```bash
   curl http://localhost:11434/v1/models
   ```

### Phase 4: Configure IronClaw

**1. Copy the custom deployment profile (v5.0 -- NEW):**

Copy the template `v5-templates/ironclaw-mac-mini.toml` from this project repo into `~/.ironclaw/profiles/` and set the profile env var:

```bash
mkdir -p ~/.ironclaw/profiles
cp v5-templates/ironclaw-mac-mini.toml ~/.ironclaw/profiles/
```

The profile inherits from `local` (no Docker sandbox -- see rationale below) and bundles our opinionated defaults: ports (3001 gateway, 3100 router, 3200 upload), heartbeat 30-min interval, TUI mode, parallelism, commitments retention_days=365.

**Why `local` and not `local-sandbox`:** our threat model is API calls + workspace I/O. Every tool we run (Gmail, Calendar, Discord, web search, memory) is an API call or disk read/write -- none benefit from Docker sandboxing. We are NOT a coding agent (no `shell` tool, no CodeAct). The `local` profile is simpler: one fewer daemon, no Docker Desktop, less to fail.

**Switch to `local-sandbox` triggers (document for future reference):** if we ever add a shell tool, run arbitrary code, enable engine v2 with CodeAct, or extend to coding workflows, switch: `brew install --cask docker`, change `IRONCLAW_PROFILE=local-sandbox`, review `SANDBOX_*` config.

**2. Run the onboard wizard:**
```bash
export IRONCLAW_PROFILE=ironclaw-mac-mini
ironclaw onboard
```

The wizard walks through setup steps; accept the profile defaults for most:
- Database connection (point to your local PostgreSQL)
- Security setup (uses macOS Keychain for secret master key)
- Inference provider (select Ollama for local)
- Model selection (choose qwen3.5:27b as default)
- Embeddings (select OpenAI, configure with your scoped `text-embedding-3-large` key)
- Channels (Discord -- configure in Phase 5)
- Extensions/tools (install Gmail + Calendar here or in Phase 5; see Phase 4.5 for Google Cloud prerequisites)
- Heartbeat (enable, 30-min interval, Discord notification)

**3. Configure `~/.ironclaw/.env`:**
```
# Deployment profile
IRONCLAW_PROFILE=ironclaw-mac-mini

# Primary: Local model via Ollama (free, private)
LLM_BACKEND=ollama
OLLAMA_MODEL=qwen3.5:27b

# Embeddings (use OpenAI for best retrieval quality)
EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-large

# Gateway
GATEWAY_ENABLED=true
GATEWAY_PORT=3001
GATEWAY_AUTH_TOKEN=your-secure-token

# Heartbeat
HEARTBEAT_ENABLED=true
HEARTBEAT_INTERVAL_SECS=1800

# Communication -- Discord
DISCORD_BOT_TOKEN=your_discord_bot_token
DISCORD_APPLICATION_ID=your_discord_application_id
DISCORD_GUILD_ID=your_private_server_id

# Engine: start on v1 (default). v2 bake-off at 60-day retro.
ENGINE_V2=false

# Google OAuth (used by bundled Gmail + Calendar tools -- Phase 4.5 covers the Google Cloud setup)
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
# Callback URL default is http://127.0.0.1:9876 (ephemeral listener) -- override via IRONCLAW_OAUTH_CALLBACK_URL if needed

# Keep your Anthropic API key available for quality-critical tasks
# ANTHROPIC_API_KEY=sk-ant-...
```

**v0.25.0 config precedence warning (v5.0 -- broader than v4.3):**

> **Config precedence in v0.25.0:** `DB/TOML > env > default` for **all non-credential settings** (Agent, Channels, Tunnel, Heartbeat, Embeddings, Sandbox, WASM, safety, builder, transcription, Routines, Skills, Hygiene, Search). Credentials (API keys, auth tokens) stay env-only. **Policy: never use the web UI settings tab (port 3001) to change non-credential config. Read-only inspection only.** If config seems to not take effect, check for a DB-level override via `ironclaw doctor` or the web UI before blaming `.env`.

This is broader than v4.3's LLM-only inversion warning. v0.25.0 unified precedence across subsystems.

**4. Create the heartbeat log table:**
```sql
CREATE TABLE heartbeat_log (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    action_type VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    summary TEXT,
    context_retrieved JSONB,
    notification_sent BOOLEAN DEFAULT FALSE,
    user_response TEXT,
    execution_time_ms INTEGER
);
```

**5. Create the workspace identity files from templates:**
```bash
cp v5-templates/IDENTITY.md ~/.ironclaw/IDENTITY.md
cp v5-templates/SOUL.md ~/.ironclaw/SOUL.md
cp v5-templates/USER.md ~/.ironclaw/USER.md
cp v5-templates/HEARTBEAT.md ~/.ironclaw/HEARTBEAT.md
mkdir -p ~/.ironclaw/PEOPLE
```

**These files contain only placeholders right now.** Phase 4b populates them.

### Phase 4.5: Google Cloud Setup (for bundled Gmail + Calendar tools)

1. **Create a Google Cloud project:** Go to console.cloud.google.com, create "IronClaw Agent" tied to the agent's Google account (`agent@yourdomain.com`).
2. **Enable APIs:**
   - Gmail API
   - Google Calendar API
   - Do NOT enable APIs you don't need.
3. **Configure OAuth consent screen:**
   - Google Workspace: set to Internal (no Google review needed)
   - Free Gmail: set to External in Testing mode (limited to 100 test users, fine for this)
   - App name: "IronClaw Agent"
4. **Create OAuth 2.0 Client ID:**
   - Credentials > Create Credentials > OAuth client ID
   - **Application type: Desktop app**
   - Name: "IronClaw Agent Desktop"
   - **Redirect URI: `http://127.0.0.1:9876`** (exact match required by Google; this is IronClaw's ephemeral callback listener)
5. **Copy the Client ID and Client Secret** into `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` in `~/.ironclaw/.env`.

### Phase 4.6: Google Calendar Sharing (replaces v4.3 read-only OAuth to your calendar)

1. **Open Google Calendar** signed in as YOU (not the agent).
2. **Settings for your primary calendar** → "Share with specific people or groups."
3. **Add `agent@yourdomain.com`** with permission **"See only free/busy (hide details)"**.
4. Done. The agent's Calendar tool (installed in Phase 5) will see your availability as if it were its own calendar, through Google's native delegation.

No OAuth grant to your account is involved. Sharing is revocable with one click in the same settings page. This is simpler and more secure than v4.3's planned read-only OAuth approach.

### Phase 4b: Identity Interview (v5.0 -- NEW, ~90 min)

This is the heart of the "make it yours" phase. Structural templates are in place; now Claude Code on the Mac Mini walks you through populating them with real content. **Fire up Claude Code now if it isn't already running.** Point it at this setup guide so it has the scripts below in context.

**Structure (~90 min total):**

#### 1. Agent-Naming Moment (~5 min)

Claude Code asks:
- What do you want to call your agent? (name)
- One-line temperament: (e.g., "dry, efficient, mildly sardonic" / "warm, helpful, curious" / "no-nonsense, concise, dry")
- Formality level: (casual / professional / formal)

Output: updates the `your name is [pick a name or just "Agent"]` line in IDENTITY.md and seeds the top of SOUL.md.

#### 2. SOUL.md Deep Interview (~15-20 min)

Claude Code walks through open-ended questions:
- When you write, what do you avoid? (em dashes, emojis, exclamation points, corporate-speak, hedging language, etc.)
- When someone gives you unnecessary context or over-explains, what's your reaction? (so the agent matches your tolerance for verbosity)
- What tone should the agent use when it disagrees with you? (push-back directly / gentle correction / ask-first)
- What tone should it use with external recipients? (formal / friendly / matter-of-fact)
- What makes a reply feel "right" to you vs. "AI-flavored"? (specific examples welcome)
- What does "being concise" look like for you? (3 bullets / single paragraph / structured sections)
- When the agent is uncertain, how should it communicate that? (flag explicitly / caveat softly / ask for clarification)
- How should the agent handle pet peeves in incoming messages? (mirror them / neutralize them / flag them)

Output: SOUL.md populated with sections for Communication Style, Values, Writing Preferences, Handling Uncertainty.

#### 3. USER.md Deep Interview (~15-20 min)

Scripted sections. Claude Code walks each:
- **Role / business:** What do you do, what does your business look like, who do you work with?
- **Current clients / partnerships:** Who are they, what's the relationship, what kinds of work?
- **Tools / stack:** What do you build with? (Next.js, TypeScript, Supabase, etc.)
- **Work patterns:** Deep-focus blocks / interrupt-driven / what times of day are you most productive / when do you do admin?
- **Time zone and availability:** Where are you, what hours do you want the agent to assume for scheduling?
- **Communication preferences:** How do you like to be reminded vs. asked vs. briefed?
- **Current priorities:** Top 3-5 things on your plate right now
- **Projects in flight:** Brief descriptors of what's active

Output: USER.md with fully populated sections replacing the placeholder template.

#### 4. IDENTITY.md Personalization Pass (~5 min)

Claude Code goes line-by-line:
- Fill `[Your Name]` → your name
- Fill `[Your Business]` → your business name
- Fill `agent@yourdomain.com` → your actual agent email
- Confirm role list matches what you actually want (add/remove bullets in the "Your job is" section)
- Confirm boundaries match your expectations

Output: IDENTITY.md with no bracketed placeholders remaining.

#### 5. HEARTBEAT.md Review (~10 min)

Claude Code walks the concrete checklist:
- Timing adjustments (is daily 9am the right "Project Context Review" time for your schedule?)
- Add or remove heartbeat tasks that don't fit
- Confirm channel routing (does `#heartbeat` vs `#system-alerts` vs `#cost-tracking` mapping match your Discord setup?)
- Confirm observation-only posture

Output: HEARTBEAT.md tuned to your actual schedule and concerns.

#### 6. PEOPLE/ Pattern-Setting (~15-20 min)

- Claude Code asks: "Who are your 3-5 most important ongoing contacts right now?"
- Walk the first contact in full detail: discuss what "Context" should actually capture (history, preferences, quirks, useful background), what level of detail is useful, what to leave out. This sets the pattern.
- Rapid-fill 2-4 more contacts using the established pattern. These go quickly now that the depth question is answered.

Output: 3-5 populated `PEOPLE/firstname-lastname.md` files. **This is the pattern reference for all future contacts.**

#### 7. Handoff Note

Future contacts: after this phase, you never need Claude Code to add a PEOPLE/ file again. Just tell the agent in Discord: "Add a PEOPLE/ file for Jane Smith, she's a [role] at [company]..." The agent uses the pattern established here via `memory_write` (prompting for your approval, as per the permission policy).

**Commit the populated files to the agent's private GitHub repo after Phase 4b completes** (selective commit: IDENTITY.md, SOUL.md, USER.md, PEOPLE/, HEARTBEAT.md; NOT `.env`, NOT `.system/`).

### Phase 4c: Install Commitments Skills (v5.0 -- NEW)

Install the 5 core commitment skills:

```bash
# Install from ClawHub registry (via ironclaw CLI or in-chat slash commands)
ironclaw skill install commitment-setup
ironclaw skill install commitment-triage
ironclaw skill install commitment-digest
ironclaw skill install decision-capture
ironclaw skill install delegation-tracker
```

Or install via Discord: `/skill install commitment-setup` etc.

**Run `commitment-setup` to bootstrap the workspace:**
```
/commitment-setup
```
This creates `~/.ironclaw/commitments/open/`, `~/.ironclaw/commitments/resolved/`, and initial `.config` metadata (including `retention_days=365` for `resolved/`).

**We skip:** persona bundles (`ceo-assistant`, `content-creator-assistant`, `trader-assistant`) because our persona is bespoke, authored in SOUL.md. We also skip `idea-parking` -- redundant with general note-taking.

**Noise note:** signal capture prompts for approval per our permission policy (`memory_write = AskEachTime`). Expect noise during first weeks. This is actually useful for calibration -- you see what the agent considers a "signal."

### Phase 5: Set Up Discord Communication

**Why Discord over Telegram:** Discord provides a structured server with channels, categories, threads, and per-channel notification control. Each communication stream gets its own channel with independent notification settings, tasks get their own threads, and channels carry semantic meaning for the agent (a message in `#approvals` = decision point, a file in `#file-uploads` = ingest-this, a message in `#general` = open-ended).

**Connection model:** Discord WASM channel module handles the persistent Gateway WebSocket as part of the IronClaw process. All communication encrypted via WSS/HTTPS.

1. **Create a Discord Application** at the [Discord Developer Portal](https://discord.com/developers/applications)
2. **Create a Bot user** under the application, copy the bot token
3. **Create a private Discord server** for the assistant (just you and the bot)
4. **Set up the server structure:**

   ```
   IRONCLAW ASSISTANT (Private Server)
   |
   +-- OPERATIONS (Category)
   |   +-- #general              -- Ad-hoc task requests, quick questions
   |   +-- #approvals            -- Email drafts, calendar proposals awaiting your approval
   |   +-- #research             -- Completed research summaries, spec drafts
   |   +-- #file-uploads         -- Document ingestion (drag-and-drop .md, .txt, .pdf, .docx)
   |
   +-- MONITORING (Category)
   |   +-- #heartbeat            -- Heartbeat tick updates (muted, check when needed)
   |   +-- #commitments          -- Commitment digest (v5.0 -- NEW, weekly + on-demand)
   |   +-- #cost-tracking        -- Daily cost digests, spend alerts, API usage
   |   +-- #system-alerts        -- Errors, model fallbacks, router alerts, cooldown notices
   |
   +-- MANAGEMENT (Category)
       +-- #memory               -- Memory operations, search results, forget confirmations
       +-- #router               -- Routing mode changes, /api /local /batch overrides
   ```

5. **Invite the bot to the server** with required permissions:
   - Send Messages, Embed Links, Attach Files, Read Message History
   - Manage Threads, Create Public Threads, Create Private Threads
   - Use Application Commands

6. **Configure Gateway Intents:**

   In the Discord Developer Portal, go to your application > Bot > Privileged Gateway Intents and enable the **Message Content Intent** toggle. No approval process needed for private servers under 100 bots.

   Then update the intent bitmask in IronClaw's `channels-src/discord/discord.capabilities.json`. Default is `4609` (GUILDS + GUILD_MESSAGES + DIRECT_MESSAGES); our channel-based interaction model needs MESSAGE_CONTENT too.

   Change `intents` in the `capabilities.websocket` block:
   ```json
   "intents": 37377
   ```

   Bitmask: GUILDS (1) + GUILD_MESSAGES (512) + DIRECT_MESSAGES (4096) + MESSAGE_CONTENT (32768) = **37377**

   Without MESSAGE_CONTENT, regular channel messages arrive with empty `content` fields. DMs and @mentions still work regardless — but that's not how we interact with this bot.

   Gateway WebSocket activates automatically on startup (`connect_on_start: true` default).

7. **Register guild-specific slash commands** (instant deployment, no propagation delay):

   **Memory commands:**
   - `/memory search <query>`
   - `/memory recent [count]`
   - `/memory stats`
   - `/memory forget <id>` / `/memory restore <id>`
   - `/memory forget-all <query>`

   **Heartbeat commands:**
   - `/heartbeat off` / `/heartbeat on` / `/heartbeat status`

   **Commitment commands (v5.0 -- NEW):**
   - `/commitments list [open|resolved]`
   - `/commitments digest` -- on-demand digest (otherwise weekly)
   - `/commitments resolve <id>` -- mark resolved
   - `/commitments reopen <id>`

   **Router commands:**
   - `/router status` / `/router mode <auto|api-only|local-only|haiku>`
   - `/batch <task>`

   **Cost commands:**
   - `/cost today` / `/cost month`

   **Reasoning commands:**
   - `/reasoning [N|all]`

8. **Install Gmail + Calendar tools (v5.0 -- replaces v4.3 hand-rolled OAuth):**

   ```bash
   ironclaw tool install gmail
   ironclaw tool install google-calendar
   ```

   Or via the web gateway's Extensions tab, or via the `ironclaw onboard` multi-select if you deferred from Phase 4.

   **On first use** (e.g., the agent tries to send an email or read the calendar):
   - IronClaw spins up the ephemeral OAuth listener on `127.0.0.1:9876`
   - OAuth URL surfaces in Discord (or chat)
   - You click the URL, log into **the agent's Google account** (`agent@yourdomain.com`), grant the scopes
   - Callback hits the listener; tokens land encrypted in the `secrets` table; listener shuts down (5-min timeout if you stall)
   - Agent resumes. Refresh tokens auto-renew; no re-auth unless revoked.

9. **Set up the file upload handler** in #file-uploads:
   - Bot watches for attachments in `#file-uploads`
   - Accepts .md, .txt, .pdf, .docx (up to 25MB per file)
   - Extracts text (PDF and DOCX need text extraction libraries)
   - Writes content to workspace via `memory_write` (prompts for approval per policy)
   - IronClaw's chunking/embedding pipeline handles the rest
   - Confirms ingestion with a rich embed showing word count, chunk count, and tags

10. **Set per-channel notification preferences** on your phone and desktop:

    **All Messages (buzzes your phone):**
    - `#approvals` -- Never miss a draft needing your approval
    - `#general` -- Your primary interaction channel
    - `#research` -- Know when research results are ready
    - `#system-alerts` -- Errors and failures need attention

    **Mentions Only (check when convenient):**
    - `#file-uploads` -- Confirmation of ingestion
    - `#commitments` -- Weekly digest
    - `#cost-tracking` -- Daily digest
    - `#memory` -- Background operations

    **Nothing / Muted (check on your own schedule):**
    - `#heartbeat` -- Routine tick updates
    - `#router` -- Informational routing mode changes

11. **Create Discord webhooks** for the Smart Router (Phase 9):
    - `#system-alerts` webhook URL
    - `#cost-tracking` webhook URL
    - `#router` webhook URL (optional)

### Phase 5.5: Configure Per-Tool Permissions (v5.0 -- NEW)

v0.25.0's per-user tool permission system (#1911) makes our human-in-the-loop posture explicit at the infrastructure level. Set these at first boot via the web UI (one-time), the CLI (`ironclaw tool permission set <tool> <state>`), or by accepting defaults and adjusting in-chat.

**v5.0 Permission Table:**

| Tool | State | Rationale |
|------|-------|-----------|
| `memory_read` / `memory_tree` | AlwaysAllow | Read-only workspace |
| `memory_write` | **AskEachTime (permanent)** | Identity file protection -- see Identity File Protection section |
| `time` / `json` / `echo` | AlwaysAllow | Trivial |
| `web_search` | AlwaysAllow | Core capability, read-only external |
| `shell` / file execution | **Disabled** | Not a coding agent -- tool not offered to the LLM at all |
| Gmail read (agent's own inbox, incl. forwarded mail) | AlwaysAllow | Agent's own mail |
| Gmail send (from agent's account) | `ApprovalRequirement::Always` hard-lock | External action |
| Calendar read (your calendar, via Google sharing) | AlwaysAllow | Free/busy read via Google delegation |
| Calendar read (agent's own) | AlwaysAllow | Agent's own schedule |
| Calendar write (send invite to external recipients) | `ApprovalRequirement::Always` hard-lock | External action |
| Discord send -- our private server channels | AlwaysAllow | Internal comm on our own server |
| Discord DMs to external users | `ApprovalRequirement::Always` hard-lock | External |
| `skill_install` | AskEachTime (forced `Always` on chain installs) | |
| `tool_permission_set` | AskEachTime | Prevent silent privilege escalation |
| `routine_create` / `routine_edit` | AskEachTime | Changes ongoing behavior |

**Tools we explicitly Disable** so they are not even presented to the LLM:
- `shell` and any file-execution tools -- we are NOT a coding agent
- Any Composio tools -- we skipped Composio on privacy grounds

### Phase 6: Configure API Keys and Offsite Backup

1. **Anthropic Console:**
   - Create a new Workspace called "Mac Mini Agent" at console.anthropic.com
   - Generate an API key scoped to this Workspace
   - Set spend limit and email notification threshold
   - Use this key in your IronClaw `.env` config

2. **OpenAI (embeddings only):**
   - Create a dedicated OpenAI Project scoped to `text-embedding-3-large` model access ONLY
   - Set a spend limit on the project
   - Use this key in your IronClaw embedding configuration

3. **Offsite backup to agent's Google Drive:**

   v5.0 backs up both the database AND the workspace directory.

   ```bash
   # Install rclone
   brew install rclone

   # Configure rclone with agent's Google Drive OAuth (interactive setup)
   rclone config

   # Add to crontab after pg_dump + workspace tar:
   # 2:30am -- Encrypt both database and workspace backups
   30 2 * * * gpg --symmetric --batch --passphrase-file /path/to/backup-passphrase \
     --cipher-algo AES256 \
     /Users/ironclawagent/backups/db/ironclaw-$(date +\%Y\%m\%d).sql.gz
   35 2 * * * gpg --symmetric --batch --passphrase-file /path/to/backup-passphrase \
     --cipher-algo AES256 \
     /Users/ironclawagent/backups/workspace/ironclaw-workspace-$(date +\%Y\%m\%d).tar.gz

   # 4am -- Upload both to agent's Drive
   0 4 * * * rclone copy \
     /Users/ironclawagent/backups/db/ironclaw-$(date +\%Y\%m\%d).sql.gz.gpg \
     agent-gdrive:ironclaw-backups/db/
   5 4 * * * rclone copy \
     /Users/ironclawagent/backups/workspace/ironclaw-workspace-$(date +\%Y\%m\%d).tar.gz.gpg \
     agent-gdrive:ironclaw-backups/workspace/

   # Monthly cleanup -- offsite backups older than 30 days
   0 5 1 * * rclone delete agent-gdrive:ironclaw-backups/db/ --min-age 30d
   5 5 1 * * rclone delete agent-gdrive:ironclaw-backups/workspace/ --min-age 30d
   ```

   Backups are OS-level cron jobs. The IronClaw agent is not involved in creating or uploading backups. The agent's Google Drive is just storage.

### Phase 7: Security Hardening + Agent Identity

1. **Review endpoint allowlist** -- only allow hosts the agent actually needs (`*.googleapis.com`, `accounts.google.com`, `discord.com`, `gateway.discord.gg`, specific API endpoints).
2. **Review installed tool permissions** -- capabilities.json for Gmail and Calendar tools declares HTTP allowlist and OAuth scopes. Audit the manifests after install; tighten scopes if broader than minimum (`gmail.send` + `gmail.readonly` + `calendar.events` + `calendar.readonly`).
3. **Review the agent's Google account configuration:**
   - OAuth consent screen app name "IronClaw Agent"
   - Only Gmail API and Calendar API enabled (no extras)
   - Free/busy calendar sharing to `agent@yourdomain.com` is set up and working
   - No Drive API enabled for the agent (Drive is rclone-only for backups, separate from agent's OAuth)
4. **Create the agent's GitHub account:**
   - Free account tied to agent's email
   - Create a private repo for configuration backup: workspace files (IDENTITY.md, SOUL.md, USER.md, PEOPLE/, HEARTBEAT.md), deployed profile TOML, `.env` template (placeholders only; real values NEVER committed), allowlists, scripts
   - You commit via Claude Code during maintenance sessions
5. **Review and finalize workspace identity files.** Phase 4b populated these; now verify boundaries, communication style, and heartbeat checklist are correct with all accounts set up.
6. **Test the leak detection** -- verify credentials aren't exposed in tool outputs (v0.25.0's expanded blocklist catches OpenRouter / Anthropic OAuth / Telegram / Groq patterns plus sensitive paths).
7. **Send yourself a test email from the agent** -- verify draft/approve/send flow via Discord `#approvals`.
8. **Send yourself a test calendar invite** -- verify availability checking (via Google Calendar sharing) and draft/approve/send flow.
9. **Test identity file write approval** -- ask the agent to "update USER.md to add a note that..." and verify it prompts for approval before writing.
10. **Test commitment capture** -- say "I need to call the dentist on Thursday" in chat. Verify the agent prompts for approval to capture a signal; approve; verify file appears in `commitments/open/`.
11. **Set up log rotation:**
    ```bash
    # Clean up router logs older than 14 days
    0 4 * * 0 find /Users/ironclawagent/logs -name "*.log" -mtime +14 -delete
    ```

### Phase 7.5: Build the Web Upload Page

A minimal web page running on the Mac Mini for uploading documents from desktop machines over LAN. Not the full Phase 9 custom UI, just a focused upload tool.

**What it does:**
- Drag-and-drop file upload (.md, .txt, .pdf, .docx)
- Tag uploads with project name and category
- Process files (extract text from PDF/DOCX, read MD/TXT as-is)
- Write content to workspace via `memory_write` (prompts for approval per policy)
- IronClaw's chunking/embedding pipeline handles the rest
- Show recent uploads with chunk counts

**Stack:** Next.js + TypeScript + Tailwind, or even a plain HTML page.

**Access:** Runs on the Mac Mini (e.g., `http://ironclaw-mini.local:3200`). Accessible from any device on your LAN. Protected with `GATEWAY_AUTH_TOKEN` for auth.

Both this upload page and the Discord file upload channel should work before you start seeding the brain. This page gets absorbed into the Phase 9 custom UI when that's eventually built.

### Phase 8: First Tasks, Calibration, and Validation

1. **Start IronClaw:**
   ```bash
   ironclaw start
   # or if using launchd:
   launchctl load ~/Library/LaunchAgents/com.ironclaw.agent.plist
   ```
   With debug logging:
   ```bash
   RUST_LOG=ironclaw=debug ironclaw start
   ```
2. **Send a test message via Discord `#general`:** "Hello, confirm you're running and tell me your current configuration."
3. **Test file upload:** Drop a markdown file into `#file-uploads`, verify ingestion via `/memory recent`.
4. **Test the web upload page:** Upload from another machine, verify ingestion.
5. **Verify heartbeat:** Wait for first scheduled tick, confirm update in `#heartbeat`.
6. **Check the database:** `/memory stats`.
7. **Test `/memory` commands:** search, recent, stats, forget, restore.
8. **Test human-in-the-loop:** Ask the agent to draft an email. Verify it posts to `#approvals` with approval buttons and waits.
9. **Test calendar availability:** Ask agent to suggest a meeting time. Verify it checks your calendar (via Google sharing) and proposes a time you're free.
10. **Test commitment capture end-to-end:** Have a conversation that includes obligation language. Verify the agent prompts for approval, you approve, signal appears in `commitments/open/`. Run `/commitments digest` to see it surfaced.
11. **Test threat model validations:**
    - Attempt to access credentials from a WASM tool (should fail)
    - Ingest a document with prompt injection (agent behavior should stay within IDENTITY.md boundaries)
    - Seed contradictory information, verify the agent surfaces both entries with dates
    - Ask the agent to modify IDENTITY.md — verify approval prompt
12. **Verify engine v1 + commitments compatibility:** confirm commitment-triage is firing under engine v1. If the skill silently no-ops, consult the Engine v2 Opt-In Appendix and re-test with `ENGINE_V2=true`.

13. **Run the Model Calibration Protocol:**

    Run 10 representative tasks through both Qwen 3.5 27B and Sonnet. Score 1-5 on quality, completeness, tone.

    **Suggested test tasks:**
    1. Quick factual summary
    2. Client meeting prep based on seeded context
    3. Rough draft project scope
    4. Polished client-ready proposal
    5. Competitive research synthesis
    6. Email draft (from agent's account)
    7. Cross-project comparison
    8. Follow-up extraction from meeting notes
    9. Technical research summary
    10. Strategic/business analysis

    **Save the results** in the config git repo. Re-calibrate at 30 days and 90 days to measure how the growing database narrows the quality gap.

### Phase 9: Custom Management UI + Smart Router

Build after you've used IronClaw through Discord and the web gateway enough to know what the interface actually needs to do.

**Stack:** Next.js + TypeScript + Tailwind

**How it connects:** IronClaw exposes an OpenAI-compatible `/v1/chat/completions` endpoint with Bearer token authentication (`GATEWAY_AUTH_TOKEN`) and SSE streaming. Your UI is just another client hitting these APIs.

---

### Smart Model Router (The Auto-Switching Layer)

**The problem:** IronClaw's `LLM_BACKEND` is set at the config level. It doesn't auto-detect whether a task needs local or cloud. Out of the box, you'd manually swap configs.

**The solution:** Build a lightweight proxy that sits between IronClaw and the models.

**IronClaw config change to enable the router:**
```
LLM_BACKEND=openai_compatible
LLM_BASE_URL=http://localhost:3100/v1
LLM_API_KEY=local-router-token
LLM_MODEL=auto
```

> **v5.0 precedence reminder:** if you ever changed the LLM provider via the web UI settings tab (you shouldn't have, per our Phase 4 policy), those DB settings take precedence over `.env`. Clear any DB provider overrides before switching to the Smart Router, or the `.env` change won't take effect.

**Router Classification Logic (tiered approach):**

**Tier 1: Keyword/Pattern Rules (instant, zero cost)**
Fast regex/keyword matching catches the obvious stuff before any model gets involved.

```typescript
// router/classify.ts

interface ClassificationResult {
  backend: 'local' | 'api'
  model: string
  reason: string
  confidence: number
}

const API_TRIGGER_PATTERNS = [
  /\b(polish|finalize|client-ready|deliverable|proposal)\b/i,
  /\b(compare and contrast|analyze the tradeoffs|evaluate .+ against)\b/i,
  /\b(write a (detailed|comprehensive|full) (spec|proposal|report|scope))\b/i,
  /\b(investor|pitch|contract|legal|pricing strategy)\b/i,
]

const LOCAL_TRIGGER_PATTERNS = [
  /\b(summarize|tldr|quick overview|brief)\b/i,
  /\b(what do (you|we) know about|remind me|what was)\b/i,
  /\b(organize|format|list|outline|rewrite this as)\b/i,
  /\b(rough draft|first pass|sketch out|brainstorm)\b/i,
]

function classifyByPattern(prompt: string): ClassificationResult | null {
  for (const pattern of API_TRIGGER_PATTERNS) {
    if (pattern.test(prompt)) {
      return {
        backend: 'api',
        model: 'claude-sonnet-4-6-latest',
        reason: `Pattern match: ${pattern.source}`,
        confidence: 0.85
      }
    }
  }
  for (const pattern of LOCAL_TRIGGER_PATTERNS) {
    if (pattern.test(prompt)) {
      return {
        backend: 'local',
        model: 'qwen3.5:27b',
        reason: `Pattern match: ${pattern.source}`,
        confidence: 0.85
      }
    }
  }
  return null
}
```

**Tier 2: Context-Aware Heuristics (instant, zero cost)**
```typescript
function classifyByHeuristics(messages: Message[], prompt: string): ClassificationResult | null {
  const totalTokenEstimate = messages.reduce((sum, m) => sum + m.content.length / 4, 0)
  if (totalTokenEstimate > 8000) {
    return { backend: 'api', model: 'claude-sonnet-4-6-latest',
      reason: 'Large context window (>8K tokens estimated)', confidence: 0.7 }
  }
  if (prompt.length < 200 && messages.length <= 2) {
    return { backend: 'local', model: 'qwen3.5:27b',
      reason: 'Short direct query', confidence: 0.8 }
  }
  return null
}
```

**Tier 3: Lightweight Local Classification (fast, free)**
```typescript
async function classifyByLocalModel(prompt: string): Promise<ClassificationResult> {
  const classificationPrompt = `You are a task router. Classify this user request as either LOCAL or API.

LOCAL = simple tasks: summaries, lookups, formatting, brainstorming, rough drafts, context retrieval.
API = complex tasks: polished client deliverables, multi-step analysis, strategic reasoning, detailed proposals.

Respond with ONLY "LOCAL" or "API".

User request: "${prompt.slice(0, 500)}"`

  const response = await fetch('http://localhost:11434/v1/chat/completions', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'qwen3.5:9b',
      messages: [{ role: 'user', content: classificationPrompt }],
      max_tokens: 10,
      temperature: 0
    })
  })

  const data = await response.json()
  const decision = data.choices[0].message.content.trim().toUpperCase()
  return {
    backend: decision === 'API' ? 'api' : 'local',
    model: decision === 'API' ? 'claude-sonnet-4-6-latest' : 'qwen3.5:27b',
    reason: 'Local model classification',
    confidence: 0.65
  }
}
```

**Tier 4: User Override (always wins)**
```typescript
function checkUserOverride(prompt: string): { override: ClassificationResult | null, cleanPrompt: string } {
  if (prompt.startsWith('/api ') || prompt.startsWith('/sonnet ')) {
    return {
      override: { backend: 'api', model: 'claude-sonnet-4-6-latest',
        reason: 'User override: /api', confidence: 1.0 },
      cleanPrompt: prompt.replace(/^\/(api|sonnet)\s+/, '')
    }
  }
  if (prompt.startsWith('/local ')) {
    return {
      override: { backend: 'local', model: 'qwen3.5:27b',
        reason: 'User override: /local', confidence: 1.0 },
      cleanPrompt: prompt.replace(/^\/local\s+/, '')
    }
  }
  if (prompt.startsWith('/batch ')) {
    return {
      override: { backend: 'batch', model: 'claude-sonnet-4-6-latest',
        reason: 'User override: /batch', confidence: 1.0 },
      cleanPrompt: prompt.replace(/^\/batch\s+/, '')
    }
  }
  return { override: null, cleanPrompt: prompt }
}
```

**Main router with error handling, Haiku fallback, cost tracking, and Discord webhook alerts** -- implementation unchanged from v4.3. See `v5-templates/router/` for the full code (same logic as v4.3 Phase 9 router code).

**Run the router under launchd** for auto-restart on failure (plist in `v5-templates/launchd/`).

**Emergency override:** Edit `.env` to point IronClaw directly at Ollama or the API. **Caveat:** if you ever set LLM provider via the web UI (you shouldn't have), clear that DB setting first.

**Router Discord slash commands:** `/router status`, `/router mode auto|haiku|api-only|local-only`.

**What this gives you:**
- IronClaw points at `localhost:3100`, has no idea routing is happening
- Memory, heartbeat, tools, everything works normally
- 90%+ of tasks silently go to Ollama (free)
- Complex/polished work auto-routes to Sonnet
- Error handling with Discord webhook alerts
- Full logging for classifier tuning

---

**Core UI Features to Build (Phase 9 Custom UI):**

1. **Document Upload + Indexing Zone** (absorbs the Phase 7.5 upload page)
2. **Project Context Manager** -- view/organize project-level context
3. **Chat Interface for Complex Work** -- SSE streaming, document attachments, search
4. **Memory Explorer** -- browse workspace, semantic search with RAG transparency, bulk deprecate
5. **Commitments Explorer (v5.0 -- NEW)** -- browse open/resolved commitments, resolve/reopen, history view
6. **Spend + Usage Dashboard** -- Router stats, spend trajectory, cap tracking

**Network Architecture:**
- IronClaw runs on the Mac Mini (`http://ironclaw-mini.local:3001`)
- Custom UI runs on the Mini alongside IronClaw, or on a desktop machine hitting the Mini over LAN
- For remote access: Cloudflare Tunnel
- Auth: `GATEWAY_AUTH_TOKEN`

---

### Phase 10: External Knowledge Base Integration (Skip If No Secondary KB Exists)

One-way write pipe from IronClaw into an external Supabase database. Only relevant if you have a separate application with its own knowledge base the agent should feed into.

**Architecture:**
```
  You: "Research [topic]"
        |
        v
  IronClaw Agent (does the work, sends you the result)
        |
        v
  You review
        |
        v
  You: "Push this to the knowledge base"
        |
        v
  IronClaw writes to Supabase via INSERT-only credentials
```

**Step 1: Create a restricted Postgres role in your external Supabase instance**
```sql
CREATE ROLE ironclaw_writer WITH LOGIN PASSWORD 'your-secure-password-here';
GRANT CONNECT ON DATABASE postgres TO ironclaw_writer;
GRANT USAGE ON SCHEMA public TO ironclaw_writer;
GRANT INSERT ON public.chat_memories TO ironclaw_writer;
GRANT USAGE ON SEQUENCE chat_memories_id_seq TO ironclaw_writer;
-- No SELECT, UPDATE, DELETE
```

**Step 2: Verify restrictions** (INSERT succeeds, SELECT fails with permission denied).

**Step 3: Store the connection string in IronClaw's encrypted credential vault.**

**Step 4: Build the push tool** (see v4.3 for full TypeScript example — unchanged).

**What this does NOT do:** agent cannot READ, modify, or delete from external KB. One-way only.

**Cost per push:** ~$0.00035 for the OpenAI embedding call on a typical 2,000-word summary.

---

## Post-Setup: Seed the Brain

Once IronClaw is running (Phases 1-8) with file upload working, start feeding it your business context.

### Initial Context to Upload
- Your standard proposal template and pricing frameworks
- Partnership structures and business relationships
- Your tech stack overview
- Brief summaries of active projects and clients
- Client call notes from recent meetings
- Project decision logs
- Scope changes and why they happened
- Any training materials, course outlines, or content you produce
- Competitive research you've already done

### Validation Tasks
- "Based on what you know about our stack and pricing, give me a rough complexity estimate for a client who needs..."
- "Research [prospect company], draft a preliminary scope using our standard approach"
- "How does this new prospect compare to what we built for [previous client]?"

### Ongoing: What to Feed It
- Client call notes after every meeting
- Project decision logs and scope changes
- New research or competitive analysis

### Ongoing: What to Ask It
- "Prep me for tomorrow's call with [client]"
- "[Partner] sent a new lead. Draft a preliminary scope"
- "Draft an email to [partner] with the preliminary scope" (agent drafts, posts to `#approvals`, you approve)
- "Based on client notes I uploaded yesterday, suggest a follow-up date and send me a calendar invite"

### Heartbeat Configuration

Observation-only system. See HEARTBEAT.md template. Flags things on Discord; never acts externally. 90-day observation period.

### Commitment Digest (v5.0 -- NEW)

Weekly on Monday at 7am, the `commitment-digest` skill runs and posts to `#commitments`:
- Open commitments (with age)
- Resolved this week
- Overdue (age > explicit deadline)
- Delegations awaiting follow-up

On-demand via `/commitments digest`.

### Daily Cost Digest

Every evening, cost summary in `#cost-tracking` (via Smart Router stats once Phase 9 is live).

```
Daily Cost Report - March 15, 2026

Tasks today: 12
  Local (Qwen 3.5 27B): 11
  API (Sonnet): 1

API usage:
  Input tokens: 4,230
  Output tokens: 1,870
  Estimated cost: $0.04

Month to date:
  Total API cost: $3.82 / $50.00 cap
  Projected monthly: ~$8.50

All systems normal.
```

---

## Backup Strategy (v5.0 -- Expanded)

The database is the agent's accumulated semantic memory. The workspace directory contains identity, PEOPLE, commitments, and system state. Research and project context may only live in these. Back both up.

**Local (7-day retention):**
- Daily `pg_dump` at 2am
- Daily workspace tar.gz at 2:15am (`~/.ironclaw/` directory)

**Offsite (30-day rolling retention), encrypted to agent's Google Drive:**
- DB: `rclone copy ... agent-gdrive:ironclaw-backups/db/` at 4am
- Workspace: `rclone copy ... agent-gdrive:ironclaw-backups/workspace/` at 4:05am
- Both GPG-encrypted with AES256 before upload

Zero additional cost (included in Google Workspace). OS-level cron; the agent itself never touches backup files.

**Configuration version control:** Private GitHub repo on the agent's GitHub account. Workspace files (IDENTITY.md, SOUL.md, USER.md, PEOPLE/, HEARTBEAT.md), deployed profile TOML, `.env` template (placeholders only), allowlists, router config, scripts. Commit via Claude Code during maintenance sessions. **Never commit real `.env` values.**

**Monthly restore test:** Restore latest database and workspace backups to a test environment. Takes ~5 minutes.

```bash
# Database restore test
createdb ironclaw_restore_test
psql ironclaw_restore_test -c "CREATE EXTENSION IF NOT EXISTS vector;"
gunzip -c /Users/ironclawagent/backups/db/ironclaw-$(date +%Y%m%d).sql.gz | psql ironclaw_restore_test
psql ironclaw -c "SELECT COUNT(*) FROM memories;"
psql ironclaw_restore_test -c "SELECT COUNT(*) FROM memories;"
dropdb ironclaw_restore_test

# Workspace restore test
mkdir -p /tmp/ironclaw-restore-test
tar xzf /Users/ironclawagent/backups/workspace/ironclaw-workspace-$(date +%Y%m%d).tar.gz -C /tmp/ironclaw-restore-test
diff -r /Users/ironclawagent/.ironclaw /tmp/ironclaw-restore-test/.ironclaw | head -20
rm -rf /tmp/ironclaw-restore-test
```

---

## Ongoing Maintenance

- **Daily cost digest** via Discord `#cost-tracking` (automated by heartbeat)
- **Weekly commitment digest** via Discord `#commitments` (v5.0 -- NEW)
- **Review heartbeat outputs** weekly for the first month, then monthly
- **Monthly restore test** (DB + workspace) to verify backup integrity
- **Monthly endpoint allowlist audit** and OAuth scope review
- **Update IronClaw** periodically (`brew upgrade ironclaw`)
- **Prune stale context** via `/memory forget` or Memory Explorer
- **Feed the agent after every client call** to keep project context fresh
- **Log rotation** handled by cron (14-day retention)
- **Re-calibrate model quality** at 30 and 90 days, then only when models change

### 60-Day Retro (v5.0 -- NEW)

At 60 days of live operation, do a retrospective with Claude Code:

1. **Engine v2 bake-off:** clone the workspace, flip `ENGINE_V2=true` on the clone, run in parallel for a week, compare behavior. Decide whether to switch.
2. **PEOPLE/ schema enforcement:** review drift across PEOPLE/ files. If the agent is missing context because fields are inconsistent, define a JSON Schema and attach via `.config`.
3. **Persona bundle decision:** after 60 days of commitment signal history, decide if any of the official persona bundles (`ceo-assistant`, etc.) fit better than our bespoke SOUL.md, or if we should author a custom `scott-assistant` bundle.
4. **Tool permission noise:** assess whether `memory_write = AskEachTime` noise is tolerable or if a custom `commitment_file_signal` wrapper tool is worth building (exempts commitments/ writes while keeping identity files gated).

### Ongoing Watch Items (No Timer)

- **IronClaw ships path-level permission granularity.** When it does, revisit the `memory_write = AskEachTime` policy.
- **`sanitize_auth_url()` localhost behavior.** Expected to work throughout v5.0. If a future IronClaw release gets stricter, revisit redirect URI strategy.

### Claude Code Maintenance Pattern

Use Claude Code on the Mac Mini for periodic maintenance. Point it at this guide and the CLAUDE.md for full context on decisions and policies.

**When Claude Code is active (maintenance mode):**
- Analyze heartbeat logs and reasoning traces (`/reasoning all`) to understand *why* the agent chose specific tools
- Run restore tests (DB + workspace)
- Audit configuration and security
- Build or modify custom tools, commands, or router logic
- Commit workspace changes to agent's private GitHub repo
- Review commitment digest trends -- are signals noise or signal?

**When Claude Code is inactive (normal operation):**
- IronClaw runs independently as the agent
- Heartbeat ticks on configured schedules
- You interact via Discord and (eventually) the custom UI

**Suggested cadence:**
- Weekly for the first month
- Biweekly for months 2-3
- Monthly after that

---

## Architecture Diagram

```
  You (Phone/Desktop)          You (Desk)
      |                          |
      | Discord App              | Custom Next.js UI / Web Upload
      | (private server)         | (document uploads,
      |                          |  project management,
      | #general -- tasks        |  complex conversations,
      | #approvals -- drafts     |  memory explorer,
      | #research -- results     |  commitments explorer,
      | #file-uploads -- docs    |  router stats dashboard)
      | #heartbeat -- monitoring |
      | #commitments -- digest   |
      | #cost-tracking -- spend  |
      | #system-alerts -- errors |
      | #memory -- search/manage |
      | #router -- mode control  |
      | Slash commands + buttons |
      v                          |
  +----------+                   |
  | Discord  |                   |
  | WASM     |                   |
  | Channel  |                   |
  +----+-----+                   |
       |                         |
       |    +--------------------+
       |    |
       v    v
  +-------------------+       +-------------------+
  | IronClaw Binary   |<----->| PostgreSQL +      |
  | (brew install,    |       | pgvector          |
  |  v0.25.0+)        |       | (Local on Mini)   |
  |                   |       +-------------------+
  | - Agent Loop (v1) |            |
  | - Routines Engine |   Semantic memory:
  | - Heartbeat       |   - Conversation history
  | - Commitments     |   - Vector embeddings
  | - WASM Sandbox    |   - `secrets` table (encrypted tokens)
  | - Web Gateway     |   - `memory_document_versions` (auto-versioning)
  | - Gmail tool      |   - Heartbeat logs
  | - Calendar tool   |       ~/.ironclaw/ on disk:
  +--------+----------+       - IDENTITY/SOUL/USER/HEARTBEAT.md
           |                  - PEOPLE/*.md
           | LLM              - commitments/open|resolved/*.md
           v                  - MEMORY.md, .system/
  +-------------------+       - profiles/ironclaw-mac-mini.toml
  | Smart Router      |
  | (Node.js/Express) |
  | (launchd)         |       Backups (cron):
  |                   |       - pg_dump -> encrypted -> Drive
  | Tier 1: Keywords  |       - ~/.ironclaw/ tar -> encrypted -> Drive
  | Tier 2: Heuristics|       Config: agent's private GitHub repo
  | Tier 3: Local LLM |       Embeddings: OpenAI text-embedding-3-large
  | Tier 4: /api force|
  +---+----------+----+
      |          |
      v          v
  +--------+  +------------+
  | Ollama |  | Anthropic  |
  | Local  |  | API        |
  | (free) |  | (Sonnet)   |
  | 90%+   |  | ~5-10%     |
  +--------+  +------------+
```

**Channel Philosophy:**
- **Discord** = Structured communication across dedicated channels. Slash commands with autocomplete, approval buttons at security boundaries.
- **Web Upload Page** = Document ingestion from desktop machines (Phase 7.5, absorbed into Phase 9 UI)
- **Custom UI** = Structured inputs, document management, complex work, memory explorer, commitments explorer, router stats
- **Smart Router** = Transparent proxy that auto-classifies every LLM request with error handling and Discord webhook alerts
- **All channels hit the same agent, same memory, same database**

---

## Appendix A: Building IronClaw from Source (Rare)

Only needed if you want to patch IronClaw locally before the patch lands upstream.

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Clone and build
git clone https://github.com/nearai/ironclaw.git
cd ironclaw
cargo build --release

# Verify
cargo test

# Binary at target/release/ironclaw
# You can symlink into PATH or use brew link --overwrite if replacing a brew install
```

For most users: stick with `brew install ironclaw`.

---

## Appendix B: Engine v2 Opt-In (for 60-Day Bake-Off)

v0.25.0 ships engine v2 alongside v1. v2 uses unified primitives (Thread, Capability, CodeAct with embedded Python VM) instead of v1's session/job/routine/sub-agent abstractions. **v5.0 starts on v1** because v2 is new and we already have too many moving parts in the first 60 days.

At the 60-day retro, evaluate v2 on a clone:

```bash
# Clone the workspace to a test dir
cp -r ~/.ironclaw ~/.ironclaw-v2-test

# Run IronClaw pointed at the clone with v2 enabled
IRONCLAW_HOME=~/.ironclaw-v2-test ENGINE_V2=true ironclaw start

# Compare behavior against live v1 instance for a week
# Key things to verify:
# - Does the commitments system still work (signals file correctly)?
# - Does approval UX feel the same or better?
# - Any new tool errors or regressions?
# - Cost differences (CodeAct may generate more tokens)
```

**To enable v2 globally** (after bake-off passes):
```
ENGINE_V2=true  # in ~/.ironclaw/.env
```

**Trace with:** `IRONCLAW_RECORD_TRACE=true` (renamed from `ENGINE_V2_TRACE` in v0.25.0).

**Migration triggers to revisit v1:** if v2 introduces regressions we can't work around, or if our custom router/tool integrations break under v2 semantics, revert by unsetting `ENGINE_V2`.

---

## Appendix C: TUI for Remote Debugging

v0.25.0 ships a Ratatui terminal UI in the default binary. Useful when SSH'd into the Mac Mini.

```bash
ironclaw tui
```

Shows live agent state, recent tool calls, heartbeat status. Read-only display -- no state changes from the TUI.

---

## Key Links

- **IronClaw GitHub:** https://github.com/nearai/ironclaw
- **IronClaw Releases:** https://github.com/nearai/ironclaw/releases
- **IronClaw CHANGELOG:** https://github.com/nearai/ironclaw/blob/staging/CHANGELOG.md
- **LLM Providers Guide:** https://github.com/nearai/ironclaw/blob/main/docs/LLM_PROVIDERS.md
- **Tools Reference:** https://github.com/nearai/ironclaw/tree/main/tools-src
- **Discord Developer Portal:** https://discord.com/developers/applications
- **Anthropic Console:** https://console.anthropic.com
- **Anthropic Rate Limits Docs:** https://docs.anthropic.com/en/api/rate-limits

---

## Notes

- IronClaw does NOT use Claude Code or Claude Code CLI. It has its own agent loop in Rust.
- IronClaw does NOT use OAuth tokens for LLM access. It uses standard Anthropic API keys. Fully TOS-compliant.
- **v5.0 installs via Homebrew.** Source build is appendix-only now.
- **v5.0 uses bundled Gmail + Google Calendar tools** (not hand-rolled OAuth).
- **v5.0 uses Google Calendar sharing** (not a second OAuth grant) to give the agent visibility into your availability.
- The local PostgreSQL database holds semantic memory; `~/.ironclaw/` on disk holds identity files, PEOPLE, commitments, and system state. Back up BOTH.
- The WASM sandbox means even if a tool is compromised, it can only access what you've explicitly allowed. Credentials are injected at the host boundary, never exposed to tool code.
- Workspace identity files (IDENTITY.md, SOUL.md, USER.md, HEARTBEAT.md) are re-injected into every LLM call as the system prompt. v0.25.0 adds deletion protection and auto-versioning for these files.
- **Human-in-the-loop is a security boundary.** The agent always drafts, always asks, never sends externally without explicit approval. Encoded in the per-tool permission table (Phase 5) at the infrastructure level.
- **Identity file writes prompt for approval permanently** until IronClaw ships path-level permission granularity or we build a custom wrapper tool.
- **Embeddings use OpenAI `text-embedding-3-large`** for best retrieval quality at negligible cost.
- **Heartbeat is observation-only.** Flags on Discord, never acts externally. 90-day observation period.
- **Commitments system (v5.0)** tracks obligations from conversation, capture prompts for approval, weekly digest in `#commitments`. Persona bundles skipped -- bespoke persona in SOUL.md.
- **Two-channel document ingestion:** Discord `#file-uploads` (mobile/quick) and web upload page (desktop/bulk).
- **Backups:** 7 days local (DB + workspace), 30 days offsite encrypted to agent's Google Drive. Monthly restore tests cover both.
- **Claude Code is the maintenance tool.** Used for setup (Phase 4b Identity Interview especially), periodic auditing, and building custom components. Turned off during normal agent operation.
- **The custom UI is Phase 9, not Phase 1.** Use Discord and the built-in web gateway first.
- **Hybrid local/API model strategy:** Qwen 3.5 27B via Ollama handles 90%+ at zero cost. Sonnet via Anthropic API for quality-critical. Smart Router auto-classifies with error handling and Discord webhook alerts.

---

*Document created: April 15, 2026 (v5.0 baseline)*
*Previous versions: v4.3 (March 26), v4.2, v4.1, v4.0, earlier*
*Originally created by Scott Rippey at PowerYourProcess.ai*
*v5.0 deltas: Homebrew install, bundled Google tools, Google Calendar sharing, commitments system, Identity Interview (Phase 4b), permanent memory_write approval gate, expanded backup strategy, deployment profiles, engine v1 with v2 bake-off*
*Purpose: IronClaw Mac Mini AI Assistant setup reference*
*Feel free to share and adapt for your own setup.*
