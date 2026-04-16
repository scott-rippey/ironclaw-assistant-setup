# PEOPLE/ Schema — REFERENCE DOC, DO NOT COPY TO `~/.ironclaw/`

> This file documents the convention for `~/.ironclaw/PEOPLE/*.md` files. It is NOT a template to copy verbatim. During Phase 4b Identity Interview, Claude Code creates individual person files (e.g., `~/.ironclaw/PEOPLE/jane-smith.md`) populated with the schema below. This file stays in the project repo as documentation.

---

# PEOPLE/ Schema (Convention, Not Enforced)

PEOPLE/ is a directory of per-person contact files. Each file anchors the agent's
context about a key person (client, partner, collaborator). Semantic memories
cluster naturally around these files -- the agent pulls from here for stable
context and from memory search for recent interactions.

**v5.0 enforcement posture:** schemaless. The structure below is a CONVENTION, not
a JSON Schema. v0.25.0 supports schema enforcement via `.config` metadata, but we
defer that to the 60-day retro (we don't have data yet on what fields are
load-bearing vs. nice-to-have).

---

## File location

`~/.ironclaw/PEOPLE/firstname-lastname.md`

One file per person. Lowercase, hyphenated filename.

## Schema (convention)

```markdown
# [Full Name]

## Core Info
- Role / title: [e.g., "Partner at Groundwork AI"]
- Company: [Company name]
- Relationship: [e.g., "Business partner", "Client", "Collaborator"]
- Email: [if known]
- Communication preferences: [e.g., "Prefers short emails", "Likes detailed specs"]

## Context
[Key context the agent should know when preparing for interactions with this person.
Business history, project involvement, preferences, quirks, relevant background.]

## Active Projects
[Current projects or proposals involving this person. Update as things change.]
```

## Example (filled in during Phase 4b)

```markdown
# Jane Smith

## Core Info
- Role / title: Director of Engineering
- Company: Acme Construction Tech
- Relationship: Client (active project)
- Email: jane@acmeconstructiontech.example
- Communication preferences: Prefers short technical emails with code-level specifics.
  Hates marketing language. Responds fastest Tuesday and Thursday mornings.

## Context
Jane runs a 12-engineer team at Acme. She's technical but trusts us on architecture
for the stuff her team doesn't touch day-to-day (frontend, data pipelines). She was
burned by a prior vendor who over-promised and under-delivered on timelines; the
relationship is strong because we've always given her honest estimates even when
they're longer than she hoped.

She'll often ask "what would you do" rather than "what do I want" -- take the former
seriously; she wants a real recommendation.

## Active Projects
- Retrieval system expansion (Phase 2 scoped, starting [date])
- Dashboard refactor (paused pending Jane's Q3 budget)
```

## Maintenance

- **Initial population:** Phase 4b Identity Interview walks one contact in full
  detail (~15-20 min). Rapid-fill 2-4 more from that pattern.
- **Ongoing:** ask the agent in Discord: "Add a PEOPLE/ file for [name], they are
  [role] at [company]..." The agent uses the established pattern and prompts for
  approval on the `memory_write`.
- **Updates:** the agent can update an existing PEOPLE/ file via `memory_write`
  with patch semantics (`old_string`/`new_string`, v0.25.0 #1723). Each edit is
  approval-gated and auto-versioned.
- **Deletion:** not protected by `is_identity_document()` -- these are user data,
  not identity files. If a relationship ends and the file should be removed, do it
  via the agent (approval-gated) or directly with `rm` during a Claude Code
  maintenance session.

## Not in PEOPLE/

- Your own personal family / friends unrelated to work -- scope is work contacts
  who appear in project context.
- One-off contacts -- people you emailed once. Save for the third interaction
  (when the relationship is actually ongoing).
- Temporary collaborators -- same rule. Add when the engagement becomes recurring.
