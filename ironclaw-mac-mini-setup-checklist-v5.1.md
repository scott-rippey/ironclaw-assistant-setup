# Build Checklist (v5.1)

Progress tracker for the v5.1 setup guide. Check items as you complete them.

**For Claude Code:** update this file as each step finishes so we know where to pick up next session. If resuming mid-build, read this file FIRST to find the last checked item.

---

## Phase 1: Mac Mini Base Setup
- [ ] Fresh macOS setup on Mac Mini
- [ ] Create `ironclawagent` user account (admin, local-only, no Apple ID)
- [ ] Prevent sleep (System Settings > Energy > prevent sleeping when display off)
- [ ] Enable auto-restart after power failure
- [ ] Enable FileVault -- save recovery key to secrets manager
- [ ] Enable Firewall (stealth mode ON, allow built-in + signed software)
- [ ] Install Homebrew
- [ ] Install PostgreSQL 15 + pgvector via Homebrew
- [ ] Create `ironclaw` database + vector extension
- [ ] Verify PostgreSQL is localhost-only (`listen_addresses = 'localhost'`)
- [ ] Set up database backup cron (pg_dump daily 2am, 7-day retention)
- [ ] Set up workspace backup cron (tar daily 2:15am, 7-day retention)

## Phase 2: Install IronClaw
- [ ] `brew install ironclaw`
- [ ] Verify: `ironclaw --version` shows v0.25.0+

## Phase 3: Ollama + Local Model
- [ ] `brew install ollama`
- [ ] `brew services start ollama` (persistent, survives reboots)
- [ ] `ollama pull qwen3.5:27b` (~17GB download)
- [ ] Test model with a sample prompt
- [ ] `ollama pull qwen3.5:9b` (router classifier)
- [ ] Verify API running: `curl http://localhost:11434/v1/models`

## Phase 4: Configure IronClaw
- [ ] Copy `ironclaw-mac-mini.toml` to `~/.ironclaw/profiles/`
- [ ] Run `ironclaw onboard` with `IRONCLAW_PROFILE=ironclaw-mac-mini`
- [ ] Configure `~/.ironclaw/.env` (all env vars per guide)
- [ ] Create `heartbeat_log` table in PostgreSQL
- [ ] Copy identity file templates from `v5-templates/` to `~/.ironclaw/`

## Phase 4.5: Google Cloud Setup
- [ ] Decide agent email address
- [ ] Create agent's Google account (Gmail, Calendar, Drive)
- [ ] Create Google Cloud project ("IronClaw Agent")
- [ ] Enable Gmail API + Google Calendar API
- [ ] Configure OAuth consent screen (Internal or External Testing)
- [ ] Create Desktop app OAuth 2.0 Client ID (redirect: `http://127.0.0.1:9876`)
- [ ] Copy Client ID + Secret to `~/.ironclaw/.env`

## Phase 4.6: Google Calendar Sharing
- [ ] Share your personal calendar with agent email at "See only free/busy (hide details)"

## Phase 4b: Identity Interview
- [ ] Start interview: tell CC "Let's do the Phase 4b Identity Interview"
- [ ] Step 1: Agent-Naming Moment (name, temperament, formality)
- [ ] Step 2: SOUL.md Deep Interview (tone, values, writing prefs, pet peeves)
- [ ] Step 3: USER.md Deep Interview (role, clients, stack, work patterns, priorities)
- [ ] Step 4: IDENTITY.md Personalization Pass (fill all placeholders)
- [ ] Step 5: HEARTBEAT.md Review (timing, tasks, channel routing)
- [ ] Step 6: PEOPLE/ Pattern-Setting (walk 1 contact in detail, rapid-fill 2-4 more)
- [ ] Step 7: Handoff Note reviewed
- [ ] Commit populated files to agent's private GitHub repo

## Phase 4c: Install Commitment Skills
- [ ] Install: commitment-setup, commitment-triage, commitment-digest, decision-capture, delegation-tracker
- [ ] Run `/commitment-setup` to bootstrap workspace structure
- [ ] Verify `commitments/open/` and `commitments/resolved/` directories exist
- [ ] Verify `retention_days=365` in commitments `.config`

## Phase 5: Discord + Gmail/Calendar Tools
- [ ] Create Discord Application at Developer Portal
- [ ] Create Bot user, copy bot token
- [ ] Create private Discord server
- [ ] Set up channel structure (Operations / Monitoring / Management)
- [ ] Invite bot with permissions (Send Messages, Embed Links, etc. + `applications.commands` scope)
- [ ] Configure Gateway Intents (MESSAGE_CONTENT, bitmask 37377)
- [ ] Verify slash commands auto-registered (~30s after IronClaw connects)
- [ ] Install Gmail tool: `ironclaw tool install gmail`
- [ ] Install Google Calendar tool: `ironclaw tool install google-calendar`
- [ ] Complete first OAuth flow (Screen Share into Mac Mini, click URL in browser there)
- [ ] Set up file upload handler in #file-uploads
- [ ] Set per-channel notification preferences (All / Mentions Only / Muted)
- [ ] Create Discord webhooks for Smart Router channels

## Phase 5.5: Per-Tool Permissions
- [ ] Configure full permission table per v5.1 guide
- [ ] Verify `shell` = Disabled
- [ ] Verify `file_write` / `file_execute` = Disabled
- [ ] Verify `memory_write` = AskEachTime
- [ ] Verify Gmail send / Calendar write = `ApprovalRequirement::Always` hard-lock

## Phase 6: API Keys + Offsite Backup
- [ ] Create Anthropic Workspace ("Mac Mini Agent") + API key with spend cap
- [ ] Create OpenAI project scoped to `text-embedding-3-large` only
- [ ] Generate backup passphrase: `openssl rand -base64 48 > ~/.ironclaw/backup-passphrase && chmod 600 ~/.ironclaw/backup-passphrase`
- [ ] Save backup passphrase to secrets manager
- [ ] Install rclone: `brew install rclone`
- [ ] Configure rclone with agent's Google Drive OAuth
- [ ] Add GPG encryption + rclone upload cron jobs
- [ ] Test: verify first encrypted backup appears in agent's Google Drive

## Phase 7: Security Hardening + Agent Identity
- [ ] Review endpoint allowlist (only needed hosts)
- [ ] Review installed Gmail/Calendar tool scopes (tighten if broader than needed)
- [ ] Verify agent's Google account config (correct APIs, Calendar sharing working)
- [ ] Create agent's GitHub account (separate free account, agent's email)
- [ ] Create private backup repo on agent's GitHub
- [ ] Create fine-grained PAT scoped to backup repo only (Contents: Read+Write)
- [ ] Store GitHub token in macOS Keychain via `git credential-osxkeychain`
- [ ] Review + finalize all workspace identity files
- [ ] Test leak detection (credentials not exposed in tool outputs)
- [ ] Test email draft/approve/send flow via Discord #approvals
- [ ] Test calendar invite draft/approve/send flow
- [ ] Test identity file write approval (IDENTITY.md, SOUL.md, USER.md)
- [ ] Test commitment capture (obligation language -> approval prompt -> file in commitments/open/)
- [ ] Set up log rotation cron (14-day retention)

## Phase 7.5: Web Upload Page
- [ ] Build web upload page (Next.js/plain HTML at port 3200)
- [ ] Test drag-and-drop upload from another device on LAN
- [ ] Verify ingestion via `/memory recent`

## Phase 8: First Tasks + Validation
- [ ] Start IronClaw in foreground first: `ironclaw` (verify it runs clean)
- [ ] Create IronClaw launchd plist (`com.ironclaw.agent.plist`)
- [ ] `mkdir -p ~/logs && launchctl load ~/Library/LaunchAgents/com.ironclaw.agent.plist`
- [ ] Verify not restart-looping: `launchctl list | grep ironclaw`
- [ ] Test: send message via Discord #general
- [ ] Test: file upload via Discord #file-uploads
- [ ] Test: web upload page from another machine
- [ ] Test: heartbeat first tick in #heartbeat
- [ ] Test: `/memory` commands (search, recent, stats, forget, restore)
- [ ] Test: human-in-the-loop email approval flow
- [ ] Test: calendar availability check
- [ ] Test: commitment capture end-to-end + `/commitments digest`
- [ ] Test: IDENTITY.md write approval prompt
- [ ] Test: SOUL.md write approval prompt
- [ ] Test: USER.md write approval prompt
- [ ] Test: MEMORY.md append approval prompt
- [ ] Test: shell tool is Disabled (ask a task that tempts shell -- agent should refuse)
- [ ] Test: Google Calendar sharing working (free/busy, not OAuth error)
- [ ] Test: Smart Router fallback (stop Ollama, verify #system-alerts alert)
- [ ] Test: backup + restore cycle (GPG encrypt -> decrypt -> restore to test DB)
- [ ] Verify engine v1 + commitments compatibility
- [ ] Run Model Calibration Protocol (10 tasks, local vs API, score 1-5)
- [ ] Save calibration results to config repo

## Phase 9: Smart Router (Build When Ready)
- [ ] Set up Node.js project for Smart Router
- [ ] Deploy router code at localhost:3100 (127.0.0.1 bind)
- [ ] Generate `ROUTER_AUTH_TOKEN` and add to `.env`
- [ ] Create router launchd plist (`com.ironclaw.router.plist`)
- [ ] Switch IronClaw `.env` to `LLM_BACKEND=openai_compatible` + `LLM_BASE_URL=http://localhost:3100/v1`
- [ ] Test routing: Tier 1 keyword, Tier 2 heuristic, Tier 3 local LLM, Tier 4 user override
- [ ] Test Haiku fallback on Sonnet 429/5xx
- [ ] Test Discord webhook alerts (#system-alerts, #cost-tracking)
- [ ] Test `/router` slash commands (status, mode changes)

## Phase 10: External KB (Skip If Not Applicable)
- [ ] Create restricted INSERT-only Postgres role in external Supabase
- [ ] Verify permissions (INSERT succeeds, SELECT fails)
- [ ] Store connection string in IronClaw credential vault
- [ ] Build push tool
- [ ] Test end-to-end push with approval flow

## Post-Setup: Seed the Brain
- [ ] Upload initial business context (proposal templates, pricing, client notes, project summaries)
- [ ] Run validation tasks against seeded context
- [ ] Establish ongoing feeding cadence (post-client-call notes)

---

## Scheduled Reviews

### 30-Day Check
- [ ] Review heartbeat outputs (weekly during first month)
- [ ] Re-calibrate model quality (local vs API delta with growing context)
- [ ] First monthly restore test (DB + workspace, full GPG round-trip)
- [ ] First monthly installed-skills audit
- [ ] First monthly endpoint allowlist + OAuth scope review

### 60-Day Retro
- [ ] Engine v2 bake-off (clone workspace, `ENGINE_V2=true`, parallel run)
- [ ] PEOPLE/ schema enforcement evaluation (is drift causing missed context?)
- [ ] Persona bundle decision (bespoke SOUL.md vs custom bundle?)
- [ ] Tool permission noise assessment (`memory_write` AskEachTime tolerable?)
- [ ] Cost projections vs actuals review
- [ ] Discord channel usage audit (are channels used as designed?)

### 90-Day Observation Period End
- [ ] Decide which heartbeat actions to keep, adjust, or remove
- [ ] Transition from weekly heartbeat review to monthly
