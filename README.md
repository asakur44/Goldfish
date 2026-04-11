# Goldfish

A memory system for AI-assisted development, built on [MemPalace](https://github.com/milla-jovovich/mempalace).

**L1 Wiki** (curated markdown) answers 80% of questions instantly. **L2 MemPalace** (ChromaDB) stores raw conversations for the other 20%. Claude Code navigates both seamlessly — wiki first, MemPalace as fallback.

```
You ask a question
    │
    ▼
Wiki (L1) ──── trigger match? ──── yes ──→ Answer from curated article
    │                                        (fast, cheap, cited)
    no
    │
    ▼
MemPalace (L2) ── semantic search ──→ Answer from raw conversations
    │                                   (19 MCP tools, knowledge graph)
    no results
    │
    ▼
"I don't have confident information on this."
```

---

## Quick Start

### 1. Install MemPalace + copy Goldfish template

**Requires Python 3.9–3.13** (ChromaDB is incompatible with Python 3.14).

```bash
# Install MemPalace
pip install mempalace

# Copy Goldfish into your project
cp -r goldfish/ ~/projects/your-project/
cd ~/projects/your-project/

# Initialize MemPalace (creates .mempalace/ with ChromaDB)
mempalace init .

# Connect MemPalace to Claude Code (19 MCP tools)
# Use the same Python that has mempalace installed:
claude mcp add mempalace -- python -m mempalace.mcp_server

# Or via the Claude Code marketplace:
# claude plugin marketplace add milla-jovovich/mempalace
# claude plugin install --scope user mempalace

# Note: if you have multiple Python versions, specify the path:
# claude mcp add mempalace -- /path/to/python3.10 -m mempalace.mcp_server
```

That's it. Claude Code now reads `CLAUDE.md` (wiki routing rules) and has access to MemPalace's 19 MCP tools. Both layers are live.

### 2. Fill in your identity

Edit `wiki/identity.md`:

```markdown
## Role
- **Primary:** Solo developer
- **Domain:** Full-stack web, TypeScript/Python

## Default Stance
- **Decision style:** Pragmatic
- **Venting pattern:** Frequent
- **Commit threshold:** High
```

This tells the LLM who you are. A frequent venter with a high commit threshold means the system won't treat your frustrated rants as architecture decisions.

### 3. Write your first real article

Delete the `example-*` files. Create your first decision:

```bash
# Delete examples
rm wiki/decisions/example-*.md
rm wiki/patterns/example-*.md
```

Create `wiki/decisions/your-first-decision.md`:

```yaml
---
type: decision
question: "What framework do we use for the API?"
chosen: "FastAPI"
rejected: ["Express (want Python ecosystem)", "Django (too heavy)"]
rationale: "Need async support, auto-generated OpenAPI docs, lightweight."
status: accepted
created: 2026-04-11
last_verified: 2026-04-11
triggers: [api, framework, fastapi, backend, server]
sources: ["Initial architecture discussion"]
---
```

### 4. Update the index

Edit `wiki/index.md` — replace the example entries:

```markdown
## decisions/

### your-first-decision.md
What framework we use for the API
Triggers: api, framework, fastapi, backend, server
```

### 5. Use it

Open Claude Code in the project directory. It reads `CLAUDE.md` automatically.

```
you: "What framework are we using for the API?"
claude: [reads index.md → matches "api" trigger → reads your-first-decision.md]
       "We're using FastAPI. We chose it for async support and auto-generated
        OpenAPI docs. We evaluated Express and Django but rejected both..."
```

That's it. The wiki is your L1 cache. Claude reads it before doing anything else.

---

## Day-to-Day Usage

### How you interact with it

You don't run commands or maintain the wiki manually. You just talk to Claude Code normally. The CLAUDE.md rules handle the rest:

**Asking questions:**
- You ask a question → Claude checks the wiki first → answers from cached knowledge
- If the wiki doesn't have it → Claude falls back to MemPalace (L2) search
- If L2 doesn't have it either → Claude says so honestly

**Making decisions:**
- You make a decision in conversation → Claude offers to write it to the wiki
- You say "yes" → Claude creates the article, updates the index, logs the change
- If you're just venting about JWT being terrible → Claude recognizes the frustration markers and does NOT create a wiki article

**Reviewing what you know:**
- Ask: "What decisions have we made about auth?"
- Ask: "What's in the wiki about databases?"
- Ask: "Is anything in the wiki stale?"

### How knowledge flows

```
Your conversations (ephemeral)
        │
        ▼
MemPalace L2 (raw archive — everything is searchable)
        │
        ▼ (only outcomes, decisions, patterns get promoted)
        │
Wiki L1 (curated — the 20% of knowledge that answers 80% of questions)
```

The wiki is not a dump of everything you've discussed. It's the **distilled outcomes** — what you decided, why, what you rejected, and what patterns you reuse.

### When to write wiki articles

Write an article when:
- You make an architectural decision and want to remember why
- You solve a problem the same way for the second time (it's a pattern now)
- You explicitly reject an approach (record it in anti-patterns so you don't re-explore it later)
- You set up infrastructure or config that you'll need to reference again

Don't write an article when:
- You're exploring options but haven't decided yet
- You're debugging and frustrated
- The information is a one-time thing you won't need again

### The anti-patterns file

`wiki/anti-patterns.md` is the most underrated file. It captures **why not** — rejected approaches with reasons. This prevents your future self (or future Claude sessions) from re-proposing ideas that already failed:

```markdown
### Auth0
- **Tried:** 2026-01
- **Failed because:** Pricing doesn't scale past 10K MAU
- **Context:** Might revisit if they change pricing tiers
- **Source:** Architecture session, Jan 2026
```

### The audit log

`wiki/log/YYYY-MM.md` tracks what changed and why. When you wonder "wait, didn't we used to use JWT?" — the log tells the full story. New month, new file. Claude only loads the current month by default.

---

## MemPalace Integration

MemPalace is not a separate tool — it's the L2 backing store. Goldfish's wiki (L1) sits on top of it.

### How they work together

| Action | Wiki (L1) | MemPalace (L2) |
|--------|-----------|----------------|
| Quick answer | Read `index.md` → read article | — |
| Deep retrieval | — | `mempalace_search(query, wing)` |
| Save a decision | Create wiki article | `mempalace_add_drawer` + `mempalace_kg_add` |
| Track a fact | — | `mempalace_kg_add(subject, predicate, object)` |
| Find connections | Wikilinks between articles | `mempalace_traverse` / `mempalace_find_tunnels` |
| Check history | `wiki/log/` | `mempalace_kg_timeline(entity)` |
| Auto-save session | — | Save hook fires every 15 messages |

### The 19 MCP tools Claude uses

**Read from palace:** `mempalace_status`, `mempalace_search`, `mempalace_list_wings`, `mempalace_list_rooms`, `mempalace_get_taxonomy`, `mempalace_check_duplicate`

**Write to palace:** `mempalace_add_drawer`, `mempalace_delete_drawer`

**Knowledge graph:** `mempalace_kg_query`, `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_kg_timeline`, `mempalace_kg_stats`

**Navigation:** `mempalace_traverse`, `mempalace_find_tunnels`, `mempalace_graph_stats`

**Agent diary:** `mempalace_diary_write`, `mempalace_diary_read`

**Dialect:** `mempalace_get_aaak_spec`

### Naming alignment

Wiki articles and MemPalace rooms use the same names:
- `wiki/decisions/auth-approach.md` → MemPalace `room_auth-approach` in `wing_your-project`
- `wiki/patterns/error-handling.md` → MemPalace `room_error-handling`

This means `mempalace_search(query, room="auth-approach")` returns the raw conversations that the wiki article was distilled from. L1 gives you the curated answer; L2 gives you the full context.

### Auto-save hooks

MemPalace's Claude Code hooks automatically capture conversations into L2:

```json
// .claude/settings.local.json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "path/to/mempalace/hooks/mempal_save_hook.sh",
        "timeout": 30
      }]
    }],
    "PreCompact": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "path/to/mempalace/hooks/mempal_precompact_hook.sh",
        "timeout": 30
      }]
    }]
  }
}
```

Every 15 messages, the hook fires and Claude saves key topics, decisions, and quotes to MemPalace. Before context compression, an emergency save captures everything important. L2 fills up automatically — you just need to promote the best stuff to L1 wiki articles.

### Mining existing conversations

```bash
# Mine conversation exports (Claude, ChatGPT, Slack)
mempalace mine ~/chats/ --mode convos

# Mine project files (code, docs)
mempalace mine ~/projects/your-project

# Split mega-files first if needed
mempalace split ~/chats/ --dry-run
```

---

## File Reference

| File | Purpose | You edit it? |
|------|---------|-------------|
| `CLAUDE.md` | Rules for the LLM — how to read the wiki, when to fall back, mining rules | Rarely (only to tune behavior) |
| `wiki/identity.md` | Who you are, your role, defaults, current context | Yes — update when your focus changes |
| `wiki/index.md` | Trigger-word routing table | Yes — Claude updates it, you review |
| `wiki/decisions/*.md` | Architectural decisions with rationale | Yes — Claude drafts, you approve |
| `wiki/patterns/*.md` | Reusable solutions | Yes — Claude drafts, you approve |
| `wiki/anti-patterns.md` | Rejected approaches | Yes — Claude drafts, you approve |
| `wiki/log/YYYY-MM.md` | What changed and why (rolling monthly) | No — Claude maintains this |
| `raw/` | Source documents (PDFs, specs, articles) | Yes — drop files in, system reads only |
| `.mempalace/` | MemPalace raw storage (ChromaDB) | No — managed by MemPalace CLI |
| `.gitignore` | Keeps `.mempalace/` and `.metrics/` out of git | No |
| `L1-L2-ARCHITECTURE.md` | Full reference architecture (1,800+ lines) | Only when adding advanced features |

---

## Month 3+ Optimizer: Progressive Enhancement

The MVM is deliberately minimal. As you use it, specific pain points will emerge. Each one has a solution in the full reference architecture (`L1-L2-ARCHITECTURE.md`). Add mechanisms only when real pain demands them.

### Stage 1: Shadow Wiki
**Trigger:** You notice Claude creating wiki articles that are wrong or half-baked, and you're spending time fixing them.

**What to add:** A `wiki/shadow/` staging directory. New articles land in shadow first and earn confidence points through usage. Only articles that prove helpful (+5 points from explicit user confirmation) get promoted to the real wiki.

**How:**
1. Create `wiki/shadow/` directory
2. Add to CLAUDE.md: "New articles go to `wiki/shadow/` first, not directly to `wiki/decisions/` or `wiki/patterns/`."
3. Add scoring rules: user says "correct"/"thanks" = +1, user says "wrong"/"actually" = -1, silence = 0
4. At score 5, Claude asks: "This draft article has been helpful 5 times. Promote to permanent wiki?"
5. At score -3, flag for deletion

See: `L1-L2-ARCHITECTURE.md` → "The Shadow Wiki (L1-Drafts)"

### Stage 2: Typed Schemas
**Trigger:** Your wiki has 20+ articles and quality varies wildly. Some decisions are missing rationale, some patterns have no anti-patterns section, some entities are outdated.

**What to add:** Required fields per article type. The lint pass validates them.

**How:**
1. Define required fields: Decisions must have `question`, `chosen`, `rejected`, `rationale`. Patterns must have `problem`, `solution`, `anti_patterns`.
2. Add a pre-commit lint check: `wiki-lint` script validates frontmatter against type schemas.
3. Articles missing required fields get flagged, not blocked.

See: `L1-L2-ARCHITECTURE.md` → "Article Type Schemas"

### Stage 3: Probe Caching
**Trigger:** Claude is checking MemPalace for contradictions on every question, even for well-established topics. It's slow and burning tokens.

**What to add:** A `wiki/.metrics/wiki_state.json` file that pre-computes which articles are stable (skip L2 probe) vs. stale (probe needed). Claude reads labels, not numbers.

**How:**
1. Create `wiki/.metrics/wiki_state.json`
2. Background script computes: `probe_state: FRESH | STALE | FORCED` per article
3. CLAUDE.md reads: "If probe_state is FRESH, answer from L1 without checking L2"
4. Probe states refresh weekly or on explicit user request

See: `L1-L2-ARCHITECTURE.md` → "Read Path" → Step 4, and "Routing"

### Stage 4: Volatility & Evidence Classes
**Trigger:** Claude keeps flagging your venting as contradictions, or real decisions aren't being caught because they were expressed casually.

**What to add:** Evidence classification (A: explicit decisions, B: implementation evidence, C: repeated references, D: venting) with different weights. Plus a volatility score that tracks how contested a topic is.

**How:**
1. Add evidence class labels to L2 fragments during mining
2. Pre-compute `volatility_state: STABLE | WARMING | HOT` per article
3. CLAUDE.md reads the label and adjusts: STABLE = answer confidently, HOT = present both sides
4. Class D evidence (venting) alone never triggers invalidation

See: `L1-L2-ARCHITECTURE.md` → "Evidence Classes" and "Semantic TTL (Volatility Scoring)"

### Stage 5: Claim-Level Provenance
**Trigger:** You have 50+ articles and find a claim that's wrong. You can't tell which source it came from because the article only has article-level citations.

**What to add:** Inline source IDs per claim: `[s1]`, `[s2]`, etc. Each with a source entry at the bottom of the article showing the exact drawer/file, speaker, date, and verification status.

**How:**
1. Switch article format from per-article `sources:` to inline `[sN]` citations
2. Add a `### Sources` section at the bottom of each article
3. Lint verifies that primary claims have valid sources (sampling-based, not exhaustive)

See: `L1-L2-ARCHITECTURE.md` → "Claim-Level Provenance"

### Stage 6: Multi-Scope (Global + Project)
**Trigger:** You're working on 3+ projects and keep re-specifying the same standards (default database, coding conventions, deployment patterns).

**What to add:** A `~/.global/wiki/` directory for cross-project standards. Claude checks project wiki first, then falls back to global wiki, then to L2.

**How:**
1. Create `~/.global/wiki/` with its own `index.md` and `identity.md`
2. Move cross-project standards there (e.g., "Always use Postgres unless justified")
3. Add cascade rule to CLAUDE.md: "If project wiki misses, check `~/.global/wiki/` before falling back to L2"
4. Project-specific overrides take priority (specificity wins)

See: `L1-L2-ARCHITECTURE.md` → "Multi-Scope Architecture"

### Stage 7: Resolution Log
**Trigger:** You've resolved several contradictions ("keep JWT, the venting wasn't a real decision") and you want the system to remember those resolutions so it doesn't re-flag the same thing.

**What to add:** `wiki/resolutions/` directory with one markdown file per resolved conflict. Plus a lint negation registry so resolved contradictions don't resurface.

**How:**
1. Create `wiki/resolutions/` directory
2. When resolving a conflict, Claude writes the reasoning as a markdown file (git-tracked)
3. Add `.mempalace/lint_ignore.json` with hashes of resolved L2 sources
4. Lint skips ignored sources in future runs

See: `L1-L2-ARCHITECTURE.md` → "Resolution Workflow" and "Lint Negation Registry"

### Stage 8: Full Lint
**Trigger:** Wiki has 50+ articles. Quality is drifting — stale claims, broken links, inconsistent schemas.

**What to add:** Tiered lint: fast checks on every commit, deeper checks weekly, expensive checks on-demand.

**How:**
1. Pre-commit (fast): broken wikilinks, frontmatter schema, missing TTL
2. Weekly cron: contradictions, staleness, citation speaker checks, state recalculation
3. On-demand (`mempalace lint --deep`): scope fragmentation, deep citation verification

See: `L1-L2-ARCHITECTURE.md` → "The Lint Pass (Tiered)"

### Stage 9: Curator Agent
**Trigger:** You're running multiple AI agents concurrently and they're creating conflicting wiki updates.

**What to add:** A dedicated L1 Curator Agent that owns the Write Path. Other agents submit changes for review rather than writing directly.

See: `L1-L2-ARCHITECTURE.md` → "Known Limitations" → #10

---

## The Full Reference Architecture

`L1-L2-ARCHITECTURE.md` (1,800+ lines) contains:

- Complete read/write path state machines with exact steps and thresholds
- Routing scoring model (pre-computed into labels the LLM reads)
- 4-component confidence model (source, freshness, routing, conflict)
- 5 answer mode templates (direct, hedged, contested, partial+expand, cannot-answer)
- Evidence classes (A/B/C/D) with weights
- Source precedence ladder (6 tiers)
- Article type schemas (Decision, Pattern, Entity, Synthesis)
- Ingestion lifecycle (dedup, tombstones, retraction propagation)
- Multi-scope cascade (Global/Project/Module)
- 14 hard constraints and invariants
- 10 known limitations and accepted tradeoffs
- 7 rounds of critique evaluation with responses

It was designed through adversarial critique — each section was stress-tested against operational realism, LLM execution capabilities, scaling limits, and curation fatigue. The MVM above is the starting point it recommends.
