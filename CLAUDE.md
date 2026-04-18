# Goldfish Memory System — CLAUDE.md

> **HARD RULE:** NEVER call `mempalace_search` or any `mempalace_` tool to answer a question before checking `wiki/index.md`. Wiki is L1 (instant, curated). MemPalace is L2 (fallback only). If the wiki answers it — stop. Do not redundantly search L2.

This project uses a two-tier memory system built on MemPalace:

- **L1 Wiki** (`wiki/`) — curated markdown articles for fast, cheap answers
- **L2 MemPalace** (ChromaDB) — raw verbatim conversations for deep retrieval

**Roles:**
- **Wiki (L1) = Reference.** Factual primers on the domain. What-things-are. Public-facing knowledge.
- **MemPalace (L2) = Institutional memory.** Decisions, rationale, strategy, deep-dives. Why-we-chose-X-over-Y.

L1 is the cache. L2 is the backing store. Check the cache first.

**Bidirectional citations:**
- L1 cites L2 for rationale (e.g., a wiki article links to `mempalace://` for the decision reasoning)
- L2 cites L1 for domain context (e.g., a decision references a wiki article for background)

---

## Wake-Up Protocol

On session start:

1. Read `wiki/identity.md` (L0 — who the user is, their role, behavior modes)
2. Call `mempalace_status` to load palace overview and confirm L2 is connected
3. Read `wiki/index.md` to load the routing table into context

You are now ready. L0 identity loaded, L1 index loaded, L2 connected.

---

## Read Path (How to Answer Questions)

### Step 1: Route via Wiki Index
Scan `wiki/index.md`. Match the user's question against trigger words. If triggers match, read that wiki article and answer from it.

**If the wiki answers the question — stop here.** Cite the article. If `last_verified` is older than 6 months, footnote: "Based on wiki article last validated {date}. Want me to verify this is still current?"

### Step 2: Search MemPalace (L2)
If the wiki has no match, search L2:

```
mempalace_search(query="user's question", wing="project_name")
```

Filter by wing if you know the project context. Answer from L2 results, citing the wing/room/drawer.

### Step 3: Deep Retrieval (L3)
If L2 returns fragmented or low-confidence results, and the user needs the full conversation:

```
mempalace_search(query="topic", wing="project_name", room="specific_room", limit=10)
```

Use room filtering to get contiguous context. Or use `mempalace_traverse(start_room="topic")` to follow connections across wings.

### Step 4: No Answer
If both L1 and L2 have nothing: say so honestly. "I don't have confident information on this."

### Step 5: Side-Effects (after answering)
- If L2 had the answer but L1 didn't — **cache miss.** If this happens 3+ times for the same topic, suggest creating a wiki article.
- After each session: call `mempalace_diary_write` to record what happened, what was decided, what matters.

---

## Write Path (How Knowledge Flows)

Knowledge flows **up** from L2 to L1. Conversations are captured in MemPalace automatically (via hooks). The best outcomes get promoted to wiki articles.

### Saving to L2 (MemPalace)
When the auto-save hook fires (every 15 messages), or when the user explicitly asks to save:

```
mempalace_add_drawer(
  wing="project_name",
  room="topic_name",
  content="verbatim content — exact words, never summarized"
)
```

Also update the knowledge graph for important facts:
```
mempalace_kg_add(
  subject="team", predicate="uses", object="FastAPI",
  valid_from="2026-04-11"
)
```

### Promoting from L2 to L1 (Wiki Articles)
When distilling L2 conversations into wiki articles:

1. **Extract outcomes, not topics.** Never create an article that says "We discussed X." If no decision or outcome was reached — skip it.
2. **Only use human utterances as foundational sources.** Assistant summaries from conversations can provide context but never serve as the sole basis for a wiki claim.
3. **Check `wiki/anti-patterns.md` first.** If the topic is a known rejected approach, don't re-promote it.
4. **Check for duplicates first:**
   ```
   mempalace_check_duplicate(content="the proposed article content")
   ```
5. **Every article must cite its source** — which MemPalace wing/room/drawer, or which `raw/` file.

### Updating Existing Knowledge
When facts change:

1. **Update L1:** Edit the wiki article. Log the change in `wiki/log/YYYY-MM.md`.
2. **Update L2:** Invalidate old facts in the knowledge graph:
   ```
   mempalace_kg_invalidate(
     subject="team", predicate="uses", object="JWT",
     ended="2026-04-11"
   )
   mempalace_kg_add(
     subject="team", predicate="uses", object="server-sessions",
     valid_from="2026-04-11"
   )
   ```
3. **If an update contradicts the current wiki article:** present both sides to the user and ask. Do not auto-update.
4. **If the user says to forget/abandon a topic:** mark the article `status: evicted`, remove triggers from `index.md`.

---

## Article Format

```yaml
---
type: decision | pattern | entity
sources: [mempalace://wing_project/room_auth/drawer_abc123, raw/specs/auth.md]
status: proposed | accepted | deprecated | evicted
created: YYYY-MM-DD
last_verified: YYYY-MM-DD
triggers: [keyword1, keyword2, keyword3]
---
```

Body in markdown. Sources reference MemPalace drawers or `raw/` files.

---

## Naming & File Hygiene

- Name articles by what they answer: `auth-approach.md`, `why-we-chose-postgres.md`
- Never: `notes.md`, `misc.md`, `ideas.md` — these are routing dead ends
- Use lowercase-hyphens (kebab-case), one topic per file
- Split files at ~300–500 lines — if a file grows beyond this, break it into focused subtopics
- Every file should follow the structure: **what** / **why** / **how to apply** / **open questions** / **references**
- Wiki article names map to MemPalace **rooms** — use the same naming where possible
- Decisions are ADRs: dated, numbered, template-driven. Supersession is marked explicitly, never deleted

---

## MemPalace Structure Mapping

| Wiki concept | MemPalace concept | Example |
|-------------|-------------------|---------|
| Project | Wing | `wing_cihang` |
| Topic/module | Room | `room_auth`, `room_vpc-connector` |
| Article type | Hall | `hall_facts`, `hall_decisions`, `hall_patterns` |
| Article content | Closet (summary) | Points to drawers |
| Source conversation | Drawer (verbatim) | Raw transcript chunk |
| Cross-topic link | Tunnel | Same room across wings |

When creating wiki articles, use the same naming as MemPalace rooms. `wiki/decisions/auth-approach.md` maps to `room_auth-approach` in L2. This makes cross-referencing seamless.

### Recommended MemPalace Room Structure

Organize L2 into topical rooms:

```
mempalace/
├── domain/       # Domain knowledge, industry context
├── product/      # Product decisions, features, roadmap
├── tech/         # Architecture, infrastructure, stack
├── compliance/   # Regulations, legal, standards
├── market/       # Market research, competitors
├── people/       # Stakeholders, team, contacts
└── decisions/    # ADRs — numbered, dated, never deleted
```

Each file = one topic. `index.md` at the root lists all rooms and files.

---

## Behavior Modes

Read `wiki/identity.md` for the user's configured modes. Key rules:

- **Debugging mode** (frustration detected, no decision language): Suppress commit signal detection. Do not create wiki articles from rants. Do not update the knowledge graph.
- **Exploration mode** (hypothetical questions): Do not create wiki articles. Do not add to knowledge graph. Wait for concrete outcomes.
- **Review mode** (user explicitly asks to review wiki): Surface all L2 evidence. Check knowledge graph for contradictions. Offer to update stale articles.

---

## Source Hierarchy

When sources disagree, prefer in this order:
1. Explicit user override (user tells you directly in chat)
2. Documents in `raw/` (immutable primary sources)
3. Human utterances in MemPalace drawers
4. Knowledge graph facts (`mempalace_kg_query`)
5. Existing wiki article
6. Assistant-generated content in MemPalace (lowest trust)

---

## Cross-Referencing

Use MemPalace navigation to find connections:

```
# Find how topics connect across wings
mempalace_traverse(start_room="auth-approach", max_hops=2)

# Find shared topics between two projects
mempalace_find_tunnels(wing_a="wing_cihang", wing_b="wing_nova")

# Timeline of a topic
mempalace_kg_timeline(entity="auth-migration")
```

When answering, draw from both L1 (curated knowledge) and L2 (raw history) to give complete answers with full context.

---

## Index Sync (Hard Rule)

Index files are load-bearing. When you add, rename, or remove any wiki article or MemPalace room:

1. **Update `wiki/index.md`** — add/update the entry with filename, description, and triggers
2. **Update `mempalace/index.md`** (if using structured rooms) — keep the Palace Map in sync
3. **Do both in the same commit as the file change.** Never leave indices stale.

If you realize an index is out of sync — fix it immediately, don't defer.

---

## What NOT to Do

- Do not create wiki articles from venting or frustrated debugging sessions
- Do not infer decisions from assistant summaries in conversations
- Do not auto-update wiki articles without user confirmation
- Do not modify files in `raw/` — it is read-only
- Do not commit `.mempalace/` or `wiki/.metrics/` to git
- Do not call `mempalace_delete_drawer` without explicit user permission
- Do not write to the knowledge graph during Debugging or Exploration mode
