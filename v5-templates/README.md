# v5-templates/

Starting templates for the v5.1 setup guide.

## Contents

| File | Purpose | Copied to |
|------|---------|-----------|
| `ironclaw-mac-mini.toml` | Deployment profile inheriting from `local` | `~/.ironclaw/profiles/ironclaw-mac-mini.toml` |
| `IDENTITY.md` | Agent identity + human-in-the-loop rules | `~/.ironclaw/IDENTITY.md` |
| `SOUL.md` | Agent persona skeleton (populated in Phase 4b) | `~/.ironclaw/SOUL.md` |
| `USER.md` | User context skeleton with interview sections | `~/.ironclaw/USER.md` |
| `HEARTBEAT.md` | Observation-only checklist with Discord routing | `~/.ironclaw/HEARTBEAT.md` |
| `PEOPLE-schema.md` | Convention doc for PEOPLE/*.md files | Not copied (reference only) |

## How these are used at build time

The Mac Mini clones this project repo (or drops these files in alongside
CLAUDE.md and the setup guide), then Phase 4 Step 5 of the setup guide copies
the templates into `~/.ironclaw/`:

```bash
cp v5-templates/IDENTITY.md   ~/.ironclaw/IDENTITY.md
cp v5-templates/SOUL.md        ~/.ironclaw/SOUL.md
cp v5-templates/USER.md        ~/.ironclaw/USER.md
cp v5-templates/HEARTBEAT.md   ~/.ironclaw/HEARTBEAT.md

mkdir -p ~/.ironclaw/profiles
cp v5-templates/ironclaw-mac-mini.toml ~/.ironclaw/profiles/
```

Phase 4b (Identity Interview) then walks through populating IDENTITY.md,
SOUL.md, USER.md with real content, tuning HEARTBEAT.md, and seeding 3-5 PEOPLE/
files using the pattern from `PEOPLE-schema.md`.

## Populated files go where?

Once populated on the Mac Mini, the REAL versions live at `~/.ironclaw/` and get:
- Committed to the agent's private GitHub repo (separate account, private)
- Backed up daily via workspace tar.gz cron (see Phase 1 and Phase 6 of the guide)

The templates in this directory stay clean (no real names, no placeholders
filled in) so they can be reused for a rebuild or a second agent instance.

## Why not in the guide directly?

Templates in a separate directory keeps them:
- Independently version-controllable (see when a template last changed)
- Easy to `cp` into place during build without copy-paste errors
- Separate from the narrative guide text so the guide doesn't balloon

## What's NOT here

- `.env` with real API keys -- those live only on the Mac Mini and in encrypted
  offsite backup. Never in any git repo.
- Populated IDENTITY.md / SOUL.md / USER.md / PEOPLE/*.md -- those live on the
  Mac Mini and in the agent's private GitHub repo. Never in this project repo.
- Smart Router TypeScript source -- the guide's Phase 9 contains the full
  implementation inline. A future version of this templates directory could
  extract it here; for v5.1 it stays inline in the guide.
- launchd plists -- documented inline in the guide's Phase 9.
