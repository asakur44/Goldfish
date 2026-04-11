# Memory System — CLAUDE.md

This project uses a two-tier memory system: a curated wiki (L1) for fast answers and MemPalace (L2) for raw conversation retrieval.

---

## Read Path (How to Answer Questions)

**Before answering any question, follow this sequence:**

### Step 1: Load Identity
Always read `wiki/identity.md` first. It defines your role, the user's defaults, and behavioral context. If the user sounds frustrated (repeated complaints, cursing, "this is broken") without decision language, assume **Debugging Mode** — do not treat rants as architectural decisions.

### Step 2: Check the Wiki Index
Read `wiki/index.md`. Match the user's question against the trigger words listed for each article. If a trigger matches, read that article and answer from it.

### Step 3: Answer from Wiki (L1)
If you found a matching article with a clear answer — respond directly. Cite the article. If the article has no `last_verified` date or it's older than 6 months, add a footnote: "Based on wiki article, last validated {date}."

### Step 4: Fall Back to MemPalace (L2)
If the wiki has no matching article — use MemPalace search (`mempalace_search` or CLI). Answer from L2 results. If L2 also has no answer, say so honestly: "I don't have confident information on this."

### Step 5: Log Side-Effects
After answering:
- If L2 had the answer but the wiki didn't — note this as a cache miss. If this happens 3+ times for the same topic in a week, suggest adding a wiki article.
- If the answer came from a wiki article, note it was a hit.

---

## Write Path (How to Create Wiki Articles)

### Mining Conversations
When extracting knowledge from conversations (L2) to create wiki articles:

1. **Extract outcomes, not topics.** Never create an article that says "We discussed X." If no decision, outcome, or definitive insight was reached — skip it. Silence is better than noise.
2. **Only use human utterances as foundational sources.** Assistant summaries from within conversations can provide context but must never be the sole basis for a wiki claim.
3. **Every article must cite its source.** At minimum: which conversation, raw file, or MemPalace drawer the claim comes from.
4. **Check `wiki/anti-patterns.md` before creating.** If the topic is a known rejected approach, don't re-promote it.

### Article Format
All wiki articles use this structure:

```yaml
---
type: decision | pattern | entity
sources: [source references]
status: proposed | accepted | deprecated
created: YYYY-MM-DD
last_verified: YYYY-MM-DD
triggers: [keyword1, keyword2, keyword3]
---
```

Body follows in markdown with source citations.

### Updating Articles
When updating an existing article:
1. Log the change in `wiki/log/YYYY-MM.md` with: what changed, why, what the old state was.
2. If the update contradicts the current article, present both sides to the user and ask which is correct. Do not auto-update.
3. If the user says to forget/abandon a topic entirely, mark the article `status: evicted` and remove its triggers from `index.md`.

---

## Naming Rules

- Name articles by what they answer, not what they are.
  - Good: `auth-approach.md`, `why-we-chose-postgres.md`
  - Bad: `notes.md`, `misc.md`, `ideas.md`
- Never create files named `notes`, `misc`, `ideas`, or `temp`. These are routing dead ends.
- Use lowercase kebab-case for filenames.

---

## Index Rules

`wiki/index.md` is the routing table. Every wiki article must have an entry with:
- Filename
- One-line description (phrased as a question the article answers)
- 3-10 trigger words

If `index.md` is getting long, consolidate related articles or prune stale triggers.

---

## Anti-Pattern Rules

`wiki/anti-patterns.md` captures rejected approaches. Before proposing a solution, check anti-patterns first. If a past approach failed, explain why it was rejected rather than re-suggesting it. Include the original context — something rejected under one set of constraints may be viable under different conditions.

---

## Source Hierarchy

When sources disagree, prefer in this order:
1. Explicit user override (user tells you directly in chat)
2. Documents in `raw/` (immutable primary sources)
3. Human utterances in MemPalace (L2)
4. Existing wiki article
5. Assistant-generated content (lowest trust)

---

## What NOT to Do

- Do not create wiki articles from venting or frustrated debugging sessions.
- Do not infer decisions from assistant summaries in conversations.
- Do not auto-update wiki articles without user confirmation.
- Do not modify files in `raw/` — it is read-only.
- Do not commit `.mempalace/` or `wiki/.metrics/` to git.
