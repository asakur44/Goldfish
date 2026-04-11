# L1 Wiki / L2 MemPalace: Architecture Strategy

## The Problem

Every AI conversation — decisions, debugging sessions, architecture debates — disappears when the session ends. Existing solutions either summarize (losing context) or store everything (creating noise). We need a two-tier system that gives fast, curated answers for the 80% case and deep verbatim retrieval for the 20%.

---

## The Core Tension

This architecture wants four things:

1. **Cheap retrieval** — answer most questions from cached curated knowledge
2. **High factual integrity** — never lie confidently, cite sources, catch contradictions
3. **Adaptive learning** — improve from use, self-correct naming, promote proven knowledge
4. **Low operational burden** — don't become a governance tax on remembering things

**You cannot maximize all four.** The full specification in this document achieves 1 and 2 at the cost of 4. Every mechanism added to guarantee truth, freshness, routing quality, conflict handling, and auditability creates curation burden. The dominant real-world failure mode is not technical collapse — it's operational drag. People stop feeding, reviewing, or trusting the system because it asks too much of them.

**The architecture has not chosen its primary job ruthlessly enough.** Memory for "fast answer retrieval" is different from "historical reasoning" or "governed organizational memory." The full spec tries to serve all of them.

### The False Precision Problem

The scoring models (routing 0–16, confidence 0–12, evidence classes A/B/C/D, volatility V, promotion at 5, eviction at -3) create an **illusion of determinism around a fundamentally fuzzy process.** Query interpretation, contradiction significance, intent strength, and scope boundaries are not this crisp. The thresholds are invented control surfaces, not empirically grounded truth. They should be treated as starting points to be tuned by real usage, not as laws.

### The Curation Fatigue Risk

The wiki must stay well-named, well-scoped, claim-cited, conflict-reviewed, promotion-filtered, and lint-clean. Once that burden rises:
- Either the wiki decays and becomes untrustworthy
- Or the human spends too much time maintaining it and stops using the system naturally

Curation fatigue is the dominant failure mode, not indexing failures or hallucination.

### The Solution: Build Minimum Viable Memory First

**Do not build the full design first.** Build the smallest honest version, then let real usage justify each additional control. The full specification below is a *reference architecture* — a map of mechanisms available when real pain demands them.

---

## Minimum Viable Memory (MVM)

Build this first. Everything else is progressive enhancement.

```
project/
├── raw/                    # Immutable sources (user-managed, system reads only)
├── wiki/
│   ├── identity.md         # L0: role, stance, 3-5 sentences
│   ├── index.md            # Simple flat index with trigger words
│   ├── log/                # Rolling monthly audit log
│   │   └── 2026-04.md
│   ├── decisions/          # Light schema: question, chosen, rationale, sources
│   ├── patterns/           # problem, solution, sources
│   └── anti-patterns.md    # "We tried X, it failed because Y"
├── .mempalace/             # L2 raw storage (gitignored)
├── .gitignore              # .mempalace/ + wiki/.metrics/
└── CLAUDE.md               # Minimal rules
```

**MVM Read Path:**
1. Load `identity.md` (L0)
2. Scan `index.md` — match triggers to query
3. Read matched article → answer
4. If no match → fall back to MemPalace L2 search → answer
5. Log hit/miss to `.metrics/` as side-effect

**MVM Write Path:**
- Mine conversations for outcomes (not topics). If no outcome detected → skip.
- New articles go directly into wiki (no shadow staging yet)
- Require per-article source citations (not per-claim yet)
- `anti-patterns.md` captures rejected approaches

**MVM Lint:**
- Pre-commit only: broken links, frontmatter present
- No contradiction detection, no volatility, no deep verification

**MVM tracks 2 metrics:** L1 hit rate and stale read rate. Nothing else until those metrics reveal specific problems.

**What MVM deliberately omits:**
- Shadow wiki (add when you notice distillation quality is too noisy)
- Volatility scoring (add when contradiction probes fire too often or too rarely)
- Evidence classes (add when venting vs. committing becomes a real problem)
- Multi-scope cascade (add when you're working on 3+ projects with shared standards)
- Typed schemas (add when article quality variance becomes painful)
- Claim-level provenance (add when per-article citations aren't granular enough)
- Probe caching (add when L2 probe cost becomes noticeable)
- Resolution log (add when you start resolving conflicts and want to remember why)

**Each mechanism in the full spec below should be added only when real pain demands it.**

---

## Full Reference Architecture

### The Memory Stack

| Layer | Source | Cost | When |
|-------|--------|------|------|
| **L0** | Identity — role, stance, behavior modes (see L0 Spec) | ~50 tokens | Always loaded |
| **L1** | **Karpathy-style Wiki** — curated markdown articles + `index.md` | ~200-2K tokens | Always loaded / on topic |
| **L2** | **MemPalace** — raw verbatim conversations in ChromaDB | ~5-10K tokens | When wiki has gaps |
| **L3** | MemPalace raw drawer retrieval | Variable | When verbatim context needed (see L3 Routing) |

### Why Wiki is L1, MemPalace is L2

- Wiki is cheap to load — plain markdown file reads, no vector DB query
- Wiki answers 80% of questions — curated articles beat searching 500 conversation fragments
- MemPalace handles the long tail — verbatim retrieval for edge cases
- Search is a fallback, not the default — at ~100 articles, `index.md` + file reads beats RAG

### L0 Specification (Identity)

L0 is not just "who is the user." It's the behavioral context that determines how every other layer interprets signals. Stored in `wiki/identity.md`, always loaded first.

```markdown
# Identity

## Role
- **Primary:** Solo developer / Founder / Tech Lead / PM
- **Domain:** Backend systems, distributed architecture

## Default Stance
- **Decision style:** Pragmatic (ships fast, revisits later)
- **Venting pattern:** Frequent frustration venting that does NOT indicate decisions
- **Commit threshold:** High — only update wiki on explicit "going with X" or implementation evidence

## Behavior Modes
- **Debugging mode:** Suppress commit signal detection. Do not treat debugging rants as architectural decisions. **Detection heuristic:** If ≥ 2 messages in the current session contain frustration markers (cursing, "this is broken", "I hate", repeated error descriptions) WITHOUT decision markers ("going with", "decided", "let's use"), assume Debugging Mode for the rest of this session.
- **Exploration mode:** When questions are hypothetical ("what if we...", "could we...", "I wonder if..."), do not create shadow articles. Wait for concrete outcomes. **Detection heuristic:** Hypothetical language + no implementation follow-up = exploration.
- **Review mode:** When user explicitly asks to review/update wiki ("update the wiki", "review my decisions", "what's stale?"), elevate all L2 evidence for consideration. **Detection heuristic:** Explicit "wiki" or "review" keywords in query.

## Context
- Active projects: [list]
- Team: [solo / team members]
- Current focus: [what they're working on this week]
```

**Why L0 matters for invalidation:** A Principal Engineer venting about JWT carries different weight than a PM relaying secondhand frustration. L0 calibrates how seriously the system takes contradiction signals. The `venting_pattern` and `commit_threshold` fields directly feed into the evidence class weighting — a user tagged as "frequent venter" gets higher thresholds before Class D evidence triggers any action.

### L3 Routing Rules

L2 (semantic search) returns relevant chunks. L3 (raw drawer retrieval) returns full, contiguous conversation transcripts. Use L3 when:

- **Temporal queries:** "What did I say on March 3rd?" — requires a specific drawer by date, not a semantic search
- **Contiguous context:** "Show me the whole debate about X" — L2 returns fragments, L3 returns the full conversation
- **Low-confidence L2 results:** When L2 returns fragmented, low-confidence chunks that need surrounding context to make sense, escalate to L3 for the full drawer containing those chunks
- **Provenance verification:** When the user doubts an L1 claim and wants to see the original conversation, not just the semantic match

### Folder Structure

```
project/
├── raw/                        # Immutable sources (articles, papers, exports)
├── wiki/                       # LLM-maintained curated knowledge (L1)
│   ├── index.md                # L1 Global Index — categories + hot triggers + Recently Active
│   ├── log/                    # Rolling monthly audit trail (see Audit Trail section)
│   │   ├── 2026-04.md          # Current month (always loaded for recent context)
│   │   └── 2026-03.md          # Historical (loaded on-demand)
│   ├── anti-patterns.md        # Global "mistakes we made" — prevents re-looping
│   ├── .metrics/               # Ephemeral machine state + observability (NOT in git)
│   │   ├── wiki_state.json     # Pre-computed: volatility, answer modes, probe states, scores
│   │   ├── query-stats.json    # MVP metrics: hit rate, stale reads, miss-but-existed
│   │   └── miss_log.json       # Cache miss aggregation for naming feedback
│   ├── decisions/              # ADR-style decision records
│   │   ├── index.md            # Wing Index — granular triggers for this domain
│   │   └── *.md
│   ├── patterns/
│   │   ├── index.md
│   │   └── *.md
│   ├── entities/
│   │   ├── index.md
│   │   └── *.md
│   ├── concepts/
│   │   ├── index.md
│   │   └── *.md
│   ├── synthesis/
│   │   ├── index.md
│   │   └── *.md
│   ├── shadow/                 # L1-Drafts — low-confidence articles awaiting promotion
│   │   ├── index.md
│   │   └── *.md
│   ├── resolutions/            # Git-tracked conflict resolution reasoning
│   │   └── *.md                # One file per contested article resolution
│   └── archive/                # Superseded/evicted articles preserved for temporal queries
│       └── *.md
├── .mempalace/                 # MemPalace data store (L2) — LOCAL ONLY, never committed
│   ├── palace/                 # ChromaDB + wings/rooms/halls
│   ├── agents/                 # Specialist agent configs
│   ├── lint_ignore.json        # Lint negation registry (resolved contradictions to skip)
│   ├── resolution_log.db       # SQLite: DERIVED CACHE from wiki/resolutions/ (rebuildable)
│   ├── config.json
│   └── wing_config.json        # Includes sensitivity tiers (green/yellow/red)
├── .gitignore                  # MUST include .mempalace/ AND wiki/.metrics/ (ephemeral state)
└── CLAUDE.md                   # Schema & rules for the LLM
```

**Cross-references use wikilinks:** `[[auth-approach]]` instead of relative paths. The index system resolves triggers to current filenames. Renaming an article updates the index; all wikilinks auto-resolve.

**Folder vs. Naming convention:** Folders are organized by *knowledge type* (decisions, patterns, entities), not by functional question format. Functional naming applies to the *index descriptions and article titles*, not folder paths. So `decisions/auth-approach.md` lives in the `decisions/` folder, but its index entry reads "How we handle authentication." The folder is for the LLM to narrow its search; the title is for routing within that narrowed set.

**CLAUDE.md portability:** `CLAUDE.md` is Claude Code's native schema file. For other tools:
- Cursor → `.cursorrules`
- Copilot → `.github/copilot-instructions.md`
- Generic → `AGENTS.md`

The content is identical; only the filename changes. Consider a build step or symlink if multi-tool support is needed.

---

## Multi-Scope Architecture (Global → Project → Module)

### The Problem

The current architecture is single-project. But real work spans multiple projects, and knowledge exists at different scopes:

- **"We use Postgres by default"** — global standard, applies everywhere
- **"cihang.app uses RDS in us-east-1"** — project decision
- **"The VPC connector uses port 5432 with TLS"** — module implementation detail

A flat wiki forces you to duplicate global standards into every project or lose them entirely. The fix: **scope inheritance with cascading reads.**

### The Triple-Tier Model

| Tier | Scope | L1 Wiki | L2 MemPalace | Example |
|------|-------|---------|-------------|---------|
| **Global** | Identity, cross-project laws, career-long patterns | "Leadership Manifesto" — golden stacks, orchestration standards, values | Raw strategy sessions, high-level decisions | "Always use Postgres unless justified" |
| **Project** | Project-specific goals, stakeholders, roadmap | "Project Bible" — tech stack, architecture, team context | Sprint planning, Slack exports, project debates | "cihang.app uses RDS on App Runner" |
| **Module** | Granular implementation details | "Implementation Docs" — code patterns, configs, how-tos | Debugging logs, module-specific conversations | "VPC connector config: port 5432, TLS required" |

### Directory Structure

```
~/.global/                          # Global tier — cross-project, career-long
├── wiki/
│   ├── identity.md                 # L0 (promoted from single-project identity.md)
│   ├── index.md                    # Global standards index
│   ├── decisions/                  # "Golden stacks," default tech choices
│   ├── patterns/                   # Cross-project reusable patterns
│   └── anti-patterns.md            # Global "never do this"
└── .mempalace/                     # Global L2 — strategy sessions, career archive

~/projects/
└── cihang-app/                     # Project tier
    ├── wiki/                       # Project L1
    │   ├── index.md                # Project-specific index
    │   ├── decisions/
    │   ├── modules/                # Module-level docs (sections, not separate wikis)
    │   │   ├── vpc-connector.md    # Module as article, not separate wiki instance
    │   │   └── auth-service.md
    │   ├── shadow/
    │   ├── archive/
    │   └── log.md
    ├── .mempalace/                 # Project L2 — project-specific conversations
    ├── raw/
    ├── .gitignore
    └── CLAUDE.md
```

**Key design decision: Modules are articles within the project wiki, NOT separate wiki instances.** A separate wiki per module (with its own index.md, shadow/, archive/, metrics/) creates massive infrastructure overhead for marginal benefit. Module-level detail belongs in `wiki/modules/vpc-connector.md` — a detailed article with its own frontmatter, triggers, and claim-level citations. If a module grows complex enough to warrant its own wiki, it's really a sub-project and should be promoted to the project tier.

Module-level L2 data lives in MemPalace **rooms** within the project wing (which is exactly what MemPalace's room architecture is designed for):

```
.mempalace/palace/
├── wing_cihang/
│   ├── room_vpc-connector/     # Module debugging logs
│   ├── room_auth-service/      # Module-specific conversations
│   └── room_general/           # Project-wide discussions
```

### The Cascade Read Path

When an agent asks "What database should I use?":

```
STEP 1: Check Module article (wiki/modules/vpc-connector.md)
  → Empty on database choice. Cache miss.

STEP 2: Check Project L1 (wiki/index.md → wiki/decisions/)
  → Found: "cihang.app uses RDS in us-east-1" [s1]
  → STOP. Answer from Project L1.

--- Only if Project L1 also misses: ---

STEP 3: Check Global L1 (~/.global/wiki/index.md)
  → Found: "Default database: Postgres unless justified" [s1]
  → Answer from Global L1 with note: "No project-specific override found. Using global default."

--- Only if ALL L1 tiers miss: ---

STEP 4: L2 Fallback (project MemPalace, then global MemPalace)
```

**The cascade is a recursive version of our existing Read Path.** Each tier uses the same routing score, confidence model, and answer modes. The only addition is: on cache miss at the current tier, escalate to the parent tier before falling back to L2.

### Conflict Priority (Specificity Wins)

| Tier | Priority | Rationale |
|------|----------|-----------|
| **Module** | Highest | Most specific — implementation details override general rules |
| **Project** | Medium | Project decisions override global defaults |
| **Global** | Lowest | Defaults and standards — overridden by any project/module-specific decision |

If Module says "Use SQLite for the embedded cache" and Global says "Always use Postgres" — Module wins. The module article should cite *why* it overrides the global standard.

### Cross-Project Insight Propagation

When a module-level discovery has broader implications:

1. **Upward signal:** If a pattern is reused in ≥ 2 projects, the lint pass flags it for potential promotion to Global L1. Example: "The VPC connector retry pattern from cihang.app was also useful in project-nova. Promote to Global patterns?"

2. **Downward propagation:** When a Global standard changes, the lint pass checks all project wikis for articles that reference the old standard. It doesn't auto-update — it flags: "Global standard changed from X to Y. Project articles referencing X: [list]. Review?"

3. **Lateral awareness:** MemPalace tunnels already connect rooms across wings. Cross-project tunnels surface when the same topic appears in different project wings. The system can note: "You solved a similar problem in project-nova — see [[nova:rate-limiting-approach]]."

### Agent Scoping

For multi-agent setups:

- **Module agents** are scoped to their module article + project wiki (read-only on project decisions, read-write on their module docs)
- **Project agents** have full access to the project wiki + read-only access to Global L1
- **Architect agents** have read access to Global L1 + all project L1s, write access to Global L1

This prevents a module agent from hallucinating global standards while working on isolated logic. The cascade read path is the enforcement mechanism — agents don't need to "know" about higher tiers; they just follow the miss → escalate flow.

### When to Use Multi-Scope

| Situation | Recommendation |
|-----------|---------------|
| Solo dev, one project | **Skip.** Single-project architecture is sufficient. |
| Solo dev, 2-3 projects | **Add Global tier.** `~/.global/wiki/` for shared standards. Projects reference but don't duplicate. |
| Team with multiple projects | **Full triple-tier.** Global for org standards, project for domain specifics, modules as articles within projects. |
| 12-agent autonomous team | **Full triple-tier + agent scoping.** Each agent jailed to its tier with cascade reads. |

---

## Runtime Specifications

### Read Path (Query → Answer)

This is the exact execution sequence. Two implementations of this doc must produce the same behavior.

**Critical design principle: The LLM reads pre-computed labels, it never calculates scores.** LLMs are unreliable at arithmetic (V = 3/45 might hallucinate as 0.6 instead of 0.067). All scoring, volatility calculation, and confidence computation happen in background scripts or cheap pre-processing passes. The LLM sees `volatility_state: STABLE` and `answer_mode: direct` — it never sums four 0-3 variables.

**State separation:** Ephemeral machine state (scores, probe dates, hit counts) lives in `.metrics/wiki_state.json`, NOT in article frontmatter. Articles contain only semantic knowledge and static routing (triggers, scope, sources). This keeps git diffs clean and prevents the LLM from corrupting article text when incrementing a counter.

```json
// .metrics/wiki_state.json — ephemeral, NOT in git
{
  "decisions/auth-approach.md": {
    "volatility_state": "STABLE",    // Pre-computed from V = contradictions/days
    "probe_state": "FRESH",          // FRESH (skip probe) | STALE (probe needed) | FORCED (negation detected)
    "answer_mode": "direct",         // Pre-computed from 4-component confidence
    "confidence_components": {
      "source": "VERIFIED",          // VERIFIED | PARTIAL | UNVERIFIED
      "freshness": "CURRENT",        // CURRENT (<30d) | AGING (30-90d) | STALE (>90d)
      "conflict": "NONE"             // NONE | LOW | HIGH
    },
    "last_l2_probe": "2026-04-10",
    "hit_count_7d": 5,
    "miss_count_7d": 0
  }
}
```

```
INPUT: User query Q

STEP 1: Load L0 (identity.md, ~50 tokens) — always present
  - Check L0 behavior mode hints in query (frustration markers → debugging mode)

STEP 2: Scan global index.md
  - LLM matches Q against category descriptions and hot triggers (semantic, not arithmetic)
  - Select best-matching category (or categories if ambiguous)

STEP 3: Load wing index for selected category(ies)
  - LLM matches Q against article triggers and descriptions
  - Select best-matching article(s)

STEP 4: Read wiki_state.json for selected article(s)
  - Check pre-computed answer_mode and probe_state
  - Check query for negation markers ("did we change...", "are we still...")
    → IF negation detected, override probe_state to FORCED

  IF answer_mode = "direct" AND probe_state = "FRESH"
    → READ ARTICLE, answer directly. Go to Step 7.

  IF answer_mode = "hedged" OR probe_state = "STALE"
    → READ ARTICLE, fire L2 contradiction probe. Go to Step 5.

  IF answer_mode = "escalate" OR no good match found
    → Skip L1, go to Step 6.

STEP 5: L2 Contradiction Probe
  - Semantic search for contradicting evidence (last 30 days)
  - Filter results against eviction log (see Post-Retrieval Filter)
  - Filter results against lint_ignore.json (resolved contradictions)
  - Join results with resolution_log for metadata injection
  - LLM reads probe results and article, selects answer mode:
    → No contradictions found → answer directly
    → Contradictions found but low-weight (Class D) → answer with footnote
    → High-weight contradictions (Class A/B) → present both sides
  - Background: update wiki_state.json with probe results

STEP 5c: Cascade to Parent Tier (if multi-scope enabled)
  - Check query for scope hints ("vpc config" → stay at Module, skip Global)
  - IF current tier is Module → escalate to Project L1, repeat Steps 2-5
  - IF current tier is Project → escalate to Global L1, repeat Steps 2-5
  - IF current tier is Global → fall through to Step 6
  - On parent hit: annotate answer with tier source

STEP 6: L2 Fallback
  - Fire MemPalace semantic search against Q (project scope first, then global if enabled)
  - Apply Post-Retrieval Eviction Filter on all L2 results
  - IF results found with high confidence → answer from L2
  - IF results are fragmented / low confidence → escalate to L3 (full drawer)
  - IF L2 answer matches content inside a poorly-routed L1 article → log cache miss
  - ANSWER MODE: direct from L2, or cannot-answer-confidently

STEP 7: Queue side-effects (non-blocking, background script)
  - Update wiki_state.json (probe results, hit/miss counters, answer mode recalculation)
  - Update .metrics/query-stats.json and miss_log.json
  - If shadow article was used and user gives explicit confirmation → +1 confidence point
  - If cache miss detected → log to miss_log (aggregate, don't notify)
  - If L2 returned promotable content → queue shadow article draft

OUTPUT: Answer + answer_mode + optional caveats/footnotes
```

### Post-Retrieval Eviction Filter (L2)

ChromaDB doesn't understand eviction. An evicted JWT article is stripped from L1 indices, but the raw JWT conversations still live in L2 with high similarity scores. Without filtering, L2 will return evicted content on every auth-related query.

**Before passing L2 results to the LLM prompt:**
1. Check each chunk's ID/metadata against the eviction log (`wiki/archive/` manifest + `wiki_state.json`)
2. If a chunk relates to an evicted topic → inject warning: `[⛔ This chunk relates to JWT auth, which was evicted on 2026-04-11. Reason: abandoned approach. See wiki/archive/auth-jwt.md]`
3. The LLM sees the warning and deprioritizes the chunk in its answer
4. If ALL top L2 results are evicted → answer mode: cannot-answer-confidently

This neutralizes evicted content at retrieval time without needing to delete from ChromaDB (which is expensive and lossy).

### Routing (LLM-Native, No Arithmetic)

The LLM does NOT calculate numerical scores. It reads pre-computed labels from `wiki_state.json` and matches semantically against index entries. The "routing score" is a specification for the **background script** that pre-computes `answer_mode` — not something the LLM evaluates at query time.

**Background script** (runs after each query as a side-effect, or on a schedule):

| Component | Range | How |
|-----------|-------|-----|
| **Title/question similarity** | 0–5 | Keyword overlap between Q and the article's description line |
| **Trigger overlap** | 0–5 | Number of trigger words present in Q (capped at 5) |
| **Scope match** | 0–3 | Q mentions terms within article's scope |
| **Freshness bonus** | 0–2 | +2 if validated < 30 days, +1 if < 90 days, +0 otherwise |
| **Prior hit success** | 0–1 | Capped at +1 as tie-breaker only (prevents rich-get-richer loop) |
| **Total** | **0–16** | |

**Output labels** (written to `wiki_state.json`, read by LLM):
- Total ≥ 9 → `answer_mode: direct`
- Total 6–8 → `answer_mode: hedged`
- Total < 6 → `answer_mode: escalate`

The LLM reads `answer_mode: direct` and acts accordingly. It never sees the number 9.

### Answer Mode Templates

The runtime chooses exactly one mode. This prevents inconsistent hedging.

| Mode | When | Template |
|------|------|----------|
| **Direct** | High confidence, no contradictions | Answer the question. No qualifiers. |
| **Hedged** | Moderate confidence or aging article | Answer + footnote: "Based on wiki article last validated {date}." |
| **Contested** | Active contradiction detected | "Wiki says X. However, recent evidence suggests Y. {Present both sides.} Which reflects current intent?" |
| **Partial + Expand** | L1 has partial answer, L2 fills gaps | Answer from L1, augment with L2 context. Note which parts came from where. |
| **Cannot Answer** | Low confidence everywhere | "I don't have confident information on this. Here's what I found: {best fragments}. Want me to search deeper?" |

### Write Path (Distillation / Update / Eviction)

```
INPUT: Trigger (periodic scan, user command, or side-effect from read path)

STEP 1: Identify candidate content in L2
  - Filter by sensitivity tier (green only by default)
  - Filter by speaker attribution (human utterances only as foundational)
  - Check L0 behavior mode: if user is in Exploration Mode, skip distillation for this session

STEP 1b: Mining extraction (strict)
  - Mining prompt MUST extract OUTCOMES, not topics:
      ✅ "Decided against JWT because refresh token logic too error-prone"
      ❌ "We talked about JWT authentication"
  - If no decision, outcome, or definitive insight is detected → return PROMOTION_SKIPPED
  - Silence is better than noise. "No change detected" is a valid and expected result.
  - The mining agent must NOT create articles that say "We discussed X" without a conclusion.

STEP 2: Check promotion predicates (ALL must pass)
  - Referenced in ≥ 2 separate sessions (durability)
  - Contains decision language OR reused in ≥ 2 contexts (reuse)
  - Source confidence above threshold (evidence)
  - Not contradicted by more recent committed evidence (currency)
  - Not present in anti-patterns.md (not a known dead end)
  - Not flagged in lint_ignore.json (not a previously dismissed contradiction)

STEP 3: Draft article (Pass 1 — LLM call)
  - Generate with claim-level citations (see Claim Granularity)
  - Apply functional naming
  - Generate trigger words
  - Assign article type schema (Decision / Pattern / Entity / Synthesis)
  - MUST include a concrete outcome statement, not just a topic summary

STEP 4: Verify (Pass 2 — sampling-based)
  - Check source file paths exist
  - Check MemPalace drawer IDs resolve
  - Check speaker attribution on cited fragments
  - Keyword match claims against source content
  - ONLY escalate to full LLM verify if cheap checks fail

STEP 5: Place in shadow/
  - confidence_score: 0
  - Update shadow/index.md with triggers
  - Log to log.md

STEP 6: Promotion (triggered by usage)
  - confidence_score ≥ 5 → move to L1 proper
  - confidence_score ≤ -3 → flag for rewrite or deletion
  - On promotion: update wing index, global index if needed, run lint

STEP 7: Update / Invalidation (triggered by read-path probes)
  - Contested → quarantine, surface on next query
  - Evicted → strip from all indices, archive, log reasoning
  - Superseded → archive old, link new, update indices

STEP 8: Lint (periodic or pre-commit)
  - Full lint pass (see Lint Pass section)
  - Auto-inject missing TTL by folder
  - Token budget check on global index
```

### Human Review Boundary

What auto-commits vs. what requires approval:

| Action | Auto | Approval Required |
|--------|------|-------------------|
| Trigger word updates (from cache-miss feedback) | ✅ | |
| Probe cache refresh (contradiction_likelihood, last_l2_probe) | ✅ | |
| TTL auto-injection by lint | ✅ | |
| Source link repairs (path correction) | ✅ | |
| Metrics updates | ✅ | |
| Shadow article creation | ✅ | |
| Shadow → L1 promotion (confidence ≥ 5) | ✅ | |
| Article rename suggestions | | ✅ |
| L1 article content update | | ✅ |
| Eviction | | ✅ |
| Conflict resolution (merge conflicts) | | ✅ |
| Sensitivity tier changes | | ✅ |
| Anti-pattern additions | | ✅ |

**Rule:** Anything that changes what the system *knows* requires approval. Anything that changes how the system *routes or measures* is automatic.

### Source Precedence Ladder

When sources disagree, this is the resolution order:

| Priority | Source | Example |
|----------|--------|---------|
| **1 (highest)** | Explicit human override / approved conflict resolution | User resolves merge conflict in chat |
| **2** | Immutable primary sources in `raw/` | Original documents, specs, exported artifacts |
| **3** | High-confidence committed L2 evidence (Class A/B) | Explicit decisions + implementation evidence from human speaker |
| **4** | Existing L1 article (if `status: accepted`) | Current wiki knowledge |
| **5** | Low-confidence L2 fragments (Class C/D) | Repeated references, sentiment, venting |
| **6 (lowest)** | Assistant-generated L2 content | AI summaries, paraphrases in conversation |

**When precedence is tied:** prefer recency. When recency is tied: prefer the source with more corroborating evidence.

### Evidence Classes

For invalidation and promotion decisions, L2 evidence is classified by strength:

| Class | Type | Examples | Weight |
|-------|------|----------|--------|
| **A** | Explicit decision artifacts | "Final decision: use sessions", "Going forward: Clerk" | 5x |
| **B** | Implementation evidence | Merged PRs, config changes, rollout notes, migration scripts | 4x |
| **C** | Repeated operational references | Multiple future-tense references to the new approach across sessions | 3x |
| **D** | Discussion / sentiment / venting | Frustration, exploration, hypotheticals | 1x |

Invalidation confidence = Σ(evidence_weight × count) / total_evidence_count. Class D evidence alone never triggers invalidation.

### Confidence Model (4 Components)

"Confidence" is decomposed into independent variables:

| Component | Measures | Range | Inputs |
|-----------|----------|-------|--------|
| **Source confidence** | Are the citations real, human-sourced, and verified? | 0–3 | Speaker attribution, file existence, lint status |
| **Freshness confidence** | Is the article current? | 0–3 | TTL, last_verified, valid_from/valid_until |
| **Routing confidence** | Did the query actually match this article well? | 0–3 | Routing score / 6 (normalized from 0–18 scale) |
| **Conflict confidence** | Is there contradicting evidence? | 0–3 | Probe results, volatility score V, evidence class of contradictions |

Total (0–12) determines answer mode (see Answer Mode Templates above).

**Freshness ≠ truth.** A 6-month-old article with zero contradictions (freshness: 1, conflict: 3) is more trustworthy than a 1-week-old article with 3 Class C contradictions (freshness: 3, conflict: 1). The 4-component model captures this.

---

## Article Type Schemas

Different article types have different required fields. The lint pass validates against these.

### Decision

```yaml
---
type: decision
question: "How should we handle authentication?"      # REQUIRED
chosen: "JWT with 24h expiry"                          # REQUIRED
rejected: ["Auth0", "roll-our-own sessions"]           # REQUIRED
rationale: "Stateless auth needed for mobile app"      # REQUIRED
scope: "All services"
owner: null                                            # Who made the call
status: accepted
decision_date: 2026-03-15
review_date: 2026-09-15
sources: [...]                                         # Claim-level (see below)
---
```

### Pattern

```yaml
---
type: pattern
problem: "Rate limiting across distributed services"   # REQUIRED
solution: "Token bucket with Redis backing"            # REQUIRED
applicability: "Any service with external API calls"   # REQUIRED
anti_patterns: ["Fixed delay retry", "No backoff"]     # REQUIRED
examples: ["wiki/decisions/api-rate-limiting.md"]
sources: [...]
---
```

### Entity

```yaml
---
type: entity
who: "Maya Chen"                                       # REQUIRED
role: "Senior Backend Engineer"                        # REQUIRED
relationships: ["assigned: auth-migration", "team: Platform"]
confidence: high
source_age: 2026-04-01
sources: [...]
---
```

### Synthesis

```yaml
---
type: synthesis
topic: "Auth landscape evaluation Q1 2026"             # REQUIRED
inputs: ["[[auth-approach]]", "[[why-we-chose-clerk]]"] # REQUIRED — articles synthesized
conclusion: "Clerk is winning but JWT still needed for mobile"
sources: [...]
---
```

The lint pass checks: Does the article's `type:` field match its folder? Does it have all REQUIRED fields for that type? Are `rejected:` options populated for decisions?

---

## Claim-Level Provenance

Article-level citations are too coarse — one bad sentence hitchhikes inside a generally sourced page. Citations are **per-claim**, using inline source IDs.

```markdown
## Auth Approach

We use JWT with 24-hour expiry for all API authentication. [s1]
Refresh tokens are stored in HttpOnly cookies to prevent XSS. [s2]
We evaluated Auth0 but rejected it due to pricing at our scale. [s3][s4]

---
### Sources
- [s1] mempalace://wing_orion/drawer/4a2f (speaker: human, 2026-03-15) | verified: 2026-04-10
- [s2] raw/decisions/auth-spec.md §3.2 | verified: 2026-04-10
- [s3] mempalace://wing_orion/drawer/3f1a (speaker: human, 2026-01-20) | verified: 2026-04-10
- [s4] raw/evaluations/auth0-pricing.md | verified: 2026-04-10
```

Each source entry includes:
- **Source ID** — MemPalace drawer or raw/ path
- **Speaker** — human or assistant (only human as foundational)
- **Date** — when the source was created
- **Verified** — when the citation was last checked against the source

The lint pass can re-verify individual claims without re-reading the entire article.

---

## Ingestion Lifecycle (Dedup / Deletion / Rebuild)

### Content Hashing for Dedup

Every item ingested into L2 gets a content hash (SHA-256 of normalized text). Before ingestion:
- Hash the content
- Check against existing hashes in the wing
- If match found → skip (log as duplicate)
- If near-match (>90% similarity via simple Jaccard on sentence set) → flag for review, don't auto-ingest

### Tombstones for Deleted Sources

Sources are never truly deleted — they get a tombstone:

```yaml
# In raw/ or MemPalace metadata
status: tombstoned
tombstoned_date: 2026-04-11
reason: "Source was incorrect / user requested removal"
```

**Retraction propagation:** When a source is tombstoned:
1. Find all L1 articles citing that source
2. Mark affected claims as `[source retracted — needs re-verification]`
3. Log to `log.md`
4. If the article has no remaining valid sources → move to `shadow/` for re-evaluation

### Archive vs. Delete

| Action | What happens | When to use |
|--------|-------------|-------------|
| **Archive** | Move to `wiki/archive/`, strip from indices, preserve content | Superseded, evicted, or deprecated articles |
| **Tombstone** | Mark as dead, propagate retractions, preserve metadata | Source material that was wrong or retracted |
| **Hard delete** | Remove from disk entirely | **Never automatic.** User-initiated only, for privacy/legal (GDPR right to erasure) |

### Rebuild Strategy

If the L2 ChromaDB store is corrupted:
1. Re-ingest from `raw/` (immutable source of truth)
2. Re-mine conversation exports (if preserved)
3. L1 wiki survives independently (it's just markdown in git)
4. Re-run lint to check L1 citations against rebuilt L2
5. Mark any citations that don't resolve as `[source unavailable — L2 rebuilt]`

---

## Negative Knowledge

The system must remember what is **not** true, not just what is. Three mechanisms:

1. **Anti-patterns** (`anti-patterns.md` + per-article rejected sections) — already in place
2. **Eviction** (`status: evicted`) — already in place
3. **Negative assertions in Knowledge Graph:**

```python
kg.add_triple("team", "does_not_use", "Auth0",
              valid_from="2026-01-25",
              reason="Pricing doesn't scale",
              source="mempalace://wing_orion/drawer/3f1a")
```

When L2 surfaces an old conversation mentioning Auth0 positively, the knowledge graph's negative assertion intercepts it before it reaches the distillation pipeline.

---

## Problem 1: Naming (The Routing Protocol)

### The Failure Mode

The LLM navigates by reading names in `index.md`. If a wiki article is named `oauth-impl-notes.md`, the LLM skips it when the user asks "how do we handle auth?" and unnecessarily fires an L2 search. Names are hash keys in this system — bad names cause silent cache misses.

**Death sentences:** `notes.md`, `misc.md`, `ideas.md` — the LLM can never confidently route to these. They become black holes where L1 knowledge goes to die.

**Taxonomy rot:** Names that made sense at creation become misleading as the project evolves. `backend.md` doesn't accommodate the CLI tool you added three weeks later.

### Mitigations

**1. Hierarchical Routing (Two-Tier Index)**

A single `index.md` bloats as the wiki grows past ~50 articles. Instead, use a two-tier structure:

**L1 Global Index (`wiki/index.md`)** — Always loaded. Contains only high-level categories and "hot" trigger words. Points to wing indices.

```markdown
## decisions/
Architectural choices, technology selections, trade-off reasoning
Hot triggers: decided, chose, why we, switched, migrated, approach

## patterns/
Reusable solutions across 2+ projects
Hot triggers: pattern, how to, recipe, boilerplate, template

## shadow/
Low-confidence drafts awaiting validation (see Shadow Wiki)
```

**L1 Wing Indices (`wiki/decisions/index.md`, etc.)** — Loaded on-demand when the global index routes to a category. Contains granular per-article triggers:

```markdown
## auth-approach.md
How we handle authentication
Triggers: login, signup, JWT, sessions, OAuth, passwords, identity, auth
Scope: Global (all services) | Status: Active | Last validated: 2026-04-10

## why-we-chose-postgres.md
Database selection rationale
Triggers: postgres, database, db, sql, data store, sqlite, mongo
Scope: Backend services | Status: Active | Last validated: 2026-04-08
```

**Routing flow:** User asks question → LLM reads global index (~50 tokens) → identifies category → reads wing index (~200 tokens) → reads target article. Two hops, but the always-loaded context stays lean.

The LLM matches the user's question against trigger words, not just titles. This is a hand-built embedding — zero infrastructure, fully auditable.

**Index Token Budget (hard limit):** If the global `index.md` exceeds **1,500 tokens**, the lint pass triggers a **Wiki Consolidation** warning:
- Merge related articles (e.g., `why-we-chose-postgres.md` + `postgres-migration-notes.md` → `database-strategy.md`)
- Prune trigger words to top 3 most-hit synonyms per article (based on `.metrics/` query logs)
- Archive stale articles that haven't been queried in 60+ days

The token budget is the forcing function that keeps L1 lean. Without it, index bloat silently degrades the system back to "dump everything into context."

**2. Functional Naming (Verb-Noun)**

Name articles by what they answer, not what they are:

| Bad | Good |
|-----|------|
| `payment-architecture.md` | `how-we-process-payments.md` |
| `database-decisions.md` | `why-we-chose-postgres.md` |
| `auth-notes.md` | `auth-approach.md` |

Filenames are for linking (noun-based). Index descriptions are for routing (question-shaped).

**3. Cache-Miss Feedback Loop (Aggregated)**

The self-correcting mechanism:

1. User asks about X
2. `index.md` scan misses
3. Falls back to L2 MemPalace search
4. L2 returns a chunk that actually exists in a badly-named L1 article
5. **Log the miss** to `.metrics/miss_log` (article, query, timestamp) — do NOT notify the user yet

**Aggregation threshold:** Only surface a rename suggestion when a single article is the source of truth for **≥ 3 distinct miss queries within 7 days.** Then surface one consolidated suggestion:

> "Your wiki article `project-phoenix.md` has been the real answer for 4 different queries this week (rate limiting, API throttling, request quotas, backpressure). Consider renaming or adding aliases."

**No per-query nagging.** Miss logging is silent. Suggestions are batched and threshold-gated. This prevents the system from becoming nagware when a user explores a single topic from multiple angles.

**4. Temporal Synonymy Handling**

The same concept gets called different things over time (auth → identity → IAM). The trigger-word index handles this if updated. The cache-miss detector auto-adds new synonyms: if queries for "IAM" keep routing to `authentication.md`, add "IAM" to its trigger list.

**5. Anti-Pattern Capture (Preventing Re-Loops)**

LLMs are good at following instructions but bad at remembering why we *stopped* doing something. Without explicit capture, a failed approach resurfaces months later as a "new idea."

Two mechanisms:

- **Global `wiki/anti-patterns.md`** — a living document of "we tried X, it failed because Y." Always checked during distillation to prevent re-promoting dead ideas.
- **Per-article `## Mistakes / Rejected Alternatives` section** — each major article captures what was tried and abandoned, with reasoning.

```markdown
## Rejected Alternatives
- **Auth0**: Evaluated 2026-01. Rejected — pricing model doesn't scale for our usage.
  Source: mempalace://wing_orion/drawer/3f1a
- **Roll-our-own JWT**: Attempted 2025-11. Abandoned — refresh token logic too error-prone.
  Source: raw/decisions/jwt-postmortem.md
```

This turns the Venting Problem into a feature: frustrated rants about failed approaches become source material for anti-pattern entries, ensuring the system remembers *why not* as well as *why*.

---

## Problem 2: Cache Invalidation (The Freshness Problem)

### The Core Failure Mode: The Venting Problem

L1 Wiki says: "Using JWT with 24h expiry."
L2 MemPalace has a raw fragment: "JWT is a nightmare, we're switching to sessions."

Naive contradiction detection flags this as invalidation. But the user was venting, not committing. If the system auto-updates L1, the wiki is now corrupted by transient emotion.

**The rule: Contradiction != Invalidation. Invalidation requires intent to commit.**

### Strategy 1: Lazy Invalidation (Query-Time Detection) — PRIMARY

Don't proactively scan L2 against L1. Do it at read time:

1. User asks: "How do we handle auth?"
2. System reads L1: "Using JWT."
3. System silently fires a contradiction probe against L2: "Recent evidence (last 30 days) that JWT was abandoned?"
4. L2 returns match on venting fragment
5. System injects caveat: "Wiki says JWT. Note: User expressed frustration on Oct 12, but no formal decision to change was found."

**Why this wins:** Most articles are dormant most of the time. Proactive scanning wastes compute. The query IS the invalidation trigger — you only pay for freshness when freshness matters.

### Strategy 2: Linguistic Commit Signals — SECONDARY

For optional background scanning (weekly cron), don't look for semantic contradiction. Look for speech acts of commitment:

- **Explicit reversal:** "Forget what I said about...", "Scratch that..."
- **Finality markers:** "Final decision:", "Going forward:", "Locked in:"
- **Migration verbs:** "Migrating to...", "Deprecating...", "Replacing X with Y"

High similarity to L1 article + commit signal = high confidence invalidation.

### Strategy 3: Frequency as Signal — FILTER

One frustrated message about JWT doesn't outweigh fifteen conversations using JWT without complaint. Weight contradictions by frequency:

- Contradiction appears 1x, existing position appears 15x → likely venting
- Contradiction appears 5x across multiple sessions → likely real
- Subsequent conversations reference the new position → confirmed

Frequency is a cheaper first filter than commit signal classification.

### Strategy 4: Scope Inference from Contradictions

When L1 says "Python stack" and L2 says "Using TypeScript for the dashboard" — that's not an invalidation, it's a missing scope boundary.

Don't pre-assign scopes manually. Infer them from contradictions:

- First contradiction triggers scope creation, not invalidation
- "Oh, `tech-stack.md` applies to the backend. The frontend is a different scope."
- Scopes grow organically from real usage

**Anti-fragmentation rule:** Scope inference can create `tech-stack-backend.md`, `tech-stack-frontend.md`, `tech-stack-cli.md` — a labyrinth of tiny files. Principle: **prefer sections over files until a file exceeds 200 lines or a genuine conflict arises.** The lint pass checks for scope fragmentation: if 2+ articles share >80% structural similarity (same frontmatter keys, overlapping triggers), suggest merging into a single article with scoped sections:

```markdown
# Tech Stack
## Backend (Python/FastAPI)
...
## Frontend (TypeScript/Svelte)
...
## CLI (Go)
...
```

### Strategy 5: Semantic TTL (Volatility Scoring)

Flat TTLs are too coarse. A 6-month-old architecture decision that has zero contradictions is more trustworthy than a 2-week-old decision with three conflicting L2 fragments. Replace flat TTL with a **Volatility Score**:

```
V = (Number of L2 Contradictions) / (Days since last L1 Update)
```

| V Range | Interpretation | System Behavior |
|---------|---------------|-----------------|
| V < 0.05 | Stable | Answer confidently |
| 0.05 ≤ V < 0.2 | Warming | Answer, footnote: "recent discussions suggest this may be in flux" |
| V ≥ 0.2 | Hot | Lead with: "The wiki says X, but this is actively contested" |

**Base TTL still applies** as a floor — even with V = 0, an article older than its TTL gets a staleness footnote:

| Content Type | Base TTL |
|-------------|----------|
| Mission / values | Infinite |
| Architecture decisions | 6 months |
| Current sprint goals | 2 weeks |
| Debugging notes | 30 days |

The combination means: a stable old article hedges mildly ("this is 5 months old"), but a volatile recent article hedges aggressively ("decided last week but already 3 contradictions").

**Active degradation, not silent hedging.** When an article exceeds its TTL, don't just silently reduce confidence in the answer. Make the staleness visible and actionable:

- **Visible warning:** Prepend to answer: `[Wiki article 'auth-approach.md' has not been validated in 7 months. Treating as archival context.]`
- **Offer a refresh action:** "Want me to check this against recent conversations?" — turns user annoyance into an update trigger
- **If user confirms refresh:** Fire an L2 probe, update `last_verified`, recalculate volatility. If no contradictions found, article is re-validated for another TTL cycle with zero effort.

The goal: stale articles don't just produce wishy-washy answers. They produce clear answers with an explicit staleness flag and a one-step path to re-validation.

### The Confidence Budget

The system must commit when evidence is strong. Hedging on everything destroys trust.

| Condition | Behavior |
|-----------|----------|
| Validated < 30 days, no contradictions | Answer confidently |
| Validated 30-90 days, no contradictions | Answer, footnote the age |
| Any active contradiction | Present both sides |
| Validated > 90 days + contradiction | Treat L1 as hypothesis, lead with L2 |

### Resolution Workflow (Merge Conflicts)

When invalidation is confidently detected, don't auto-update. Open a merge conflict:

1. **Quarantine:** Article marked `<status: contested>`
2. **Present the diff:**
   - Current Wiki: "Use JWT"
   - Challenger (L2): "Oct 14: 'Migrating to sessions, JWT deprecated'"
3. **Force choice:**
   - [ ] Update wiki to sessions
   - [ ] Keep JWT, mark L2 as historical
   - [ ] Merge: JWT for mobile, sessions for web
4. **Write reasoning to resolution log** — stored as git-tracked markdown in `wiki/resolutions/`, one file per contested article. SQLite (`.mempalace/resolution_log.db`) is a **derived cache** rebuilt from the markdown files for query-time joins — never the source of truth.

   ```markdown
   # wiki/resolutions/auth-jwt-2026-04-11.md
   ---
   chunk_id: mempalace://wing_prod/drawer/vent-101
   resolution: kept_l1
   l1_article: decisions/auth-approach.md
   resolved_date: 2026-04-11
   ---
   ## Reasoning
   Kept JWT. The L2 fragment was frustration venting after a debugging session,
   not an architectural decision. User confirmed JWT is still the approach.
   Three subsequent sessions referenced JWT without complaint.
   ```

   **Why markdown, not SQLite-only:** Git should track everything that explains *why* knowledge changed. If you `git checkout` to an old commit, the resolution history should match the wiki state at that point. SQLite at HEAD decoupled from markdown at an old commit creates temporal inconsistency.

   At query time, the background script rebuilds `resolution_log.db` from `wiki/resolutions/*.md` and the system joins L2 results against it to inject metadata: `[Meta: This fragment was flagged as venting and resolved as non-actionable on 2026-04-10]`.

**Conflict fatigue mitigation:** Don't batch conflicts as notifications. Surface them lazily — only when the contested article is actually queried. Most conflicts resolve themselves (the user mentions the answer in a subsequent conversation).

### The Forget Operation (Eviction)

Update and Read are well-covered. **Delete is missing.** When a user says "Forget the JWT idea entirely, we aren't doing auth that way" — this isn't an invalidation to merge. It's an eviction. But vector DBs don't un-learn, and leaving the article in L1 means the aliased index might still accidentally route to it.

**`status: evicted`** — a distinct lifecycle state:

1. Article is **not deleted** (preserved for historical audit in `wiki/archive/`)
2. All triggers are **stripped from every index** (global and wing) — the article becomes invisible to routing
3. Article markdown is prepended with: `> ⛔ EVICTED — This decision was abandoned, not superseded. Do not reference as current. See log.md for reasoning.`
4. Log entry records the eviction with reasoning
5. If the topic resurfaces in future L2 conversations, the system surfaces the eviction history: "This was previously evicted because X. Are you reconsidering?"

Eviction differs from `deprecated` and `superseded`:
- **Deprecated:** "We used to do this, now we do something else" → points to successor
- **Superseded:** "A newer version of this decision exists" → points to replacement
- **Evicted:** "This was never the right path" → points to nothing, prevents re-routing

---

## Problem 3: Factual Integrity (The Hallucination Problem)

### The Core Rule

The wiki is a **summarization layer**, never a **generation layer**. Every claim must cite a source.

### Mitigations

**1. Source Grounding as Hard Constraint**

Every factual statement must reference a `raw/` file or MemPalace drawer ID. No citation → no claim. Enforced in CLAUDE.md:

```
RULE: Every factual statement in a wiki article MUST include
a source reference. If you cannot cite a specific source,
mark the claim as [unverified] or do not include it.
```

**2. Immutable Raw Layer**

`raw/` is read-only. The LLM never modifies sources. Ground truth always exists to verify against.

**2b. Speaker Attribution in L2 (The AI Slop Filter)**

MemPalace conversations contain both human and assistant utterances. The assistant may have summarized, paraphrased, or subtly hallucinated during the original conversation. If you distill an L2 fragment where Claude said "So we decided to use JWT" but the human never confirmed — you've hardened AI slop into curated wiki truth.

**Hard rule for the distillation pipeline:**
- Only extract decisions, patterns, or facts from `speaker: human` fragments as foundational sources
- `speaker: assistant` fragments can provide context and framing, but never serve as the sole source for an L1 claim
- MemPalace metadata must tag each chunk with speaker attribution
- The `wiki-lint` citation verifier checks that cited L2 sources point to human utterances, not assistant summaries

```
# Valid L1 citation
Source: mempalace://wing_orion/drawer/4a2f (speaker: human)
"We're going with JWT because the mobile app needs stateless auth"

# Invalid as sole source — assistant paraphrasing
Source: mempalace://wing_orion/drawer/4a2f (speaker: assistant)
"So to summarize, you've decided on JWT for stateless auth"
```

This prevents the "compression of compression" problem: raw conversations → AI summary in conversation → distilled into wiki → treated as ground truth.

**3. Provenance Frontmatter**

```yaml
---
# Identity
triggers: [postgres, database, db, sql, data store]
scope: Backend services

# Provenance
sources: [raw/decisions/db-choice.md, mempalace://wing_orion/drawer/4a2f]
last_verified: 2026-04-10

# Decision lifecycle (ADR-style)
status: accepted                    # proposed | accepted | deprecated | superseded-by: <file> | evicted
decision_date: 2026-03-15
review_date: 2026-09-15            # Auto-ping for validation
valid_from: 2026-03-15
valid_until: null                   # null = currently active
supersedes: db-choice-v1.md         # Archived predecessor

# Freshness
ttl: 6 months
last_l2_probe: 2026-04-10
contradiction_likelihood: 0.02
probe_ttl: 7 days                   # Skip probes if likelihood < 0.1

# Confidence
confidence: high                    # high | medium | draft | unverified
---
```

**4. Two-Pass Distillation (Pass 2 is sampling-based)**

When promoting from L2 to L1:
- **Pass 1:** LLM drafts article with citations (full LLM call)
- **Pass 2:** Verification — but **not a full LLM call by default.** Sampling-based:
  - Check that each `sources: [...]` file path exists
  - Check that cited MemPalace drawer IDs resolve
  - Check that cited sources are `speaker: human` (not assistant slop)
  - Check that source files contain keywords from the claim (simple string match)
  - **Only escalate to full LLM semantic verification** if a source link is broken, a drawer doesn't resolve, or keyword match fails

This turns Pass 2 from "expensive LLM call" to "cheap file existence + keyword check" in the common case. Full LLM verification is the exception, not the default.

**5. Git as Audit Trail**

The wiki is markdown in git. Every edit tracked. Diffs against sources catch drift. Revert if corrupted.

**6. Structured Audit Log (`wiki/log/`)**

The log is a **rolling monthly archive**, not a single flat file. A year of operation would make a single `log.md` 50K+ tokens — unreadable by the LLM.

```
wiki/log/
├── 2026-04.md    # Current month (LLM reads this for recent context)
├── 2026-03.md    # Previous months (read on-demand for historical queries)
└── 2026-02.md
```

The LLM only loads the current month's log by default (~30 days of entries). Historical entries are loaded on-demand when the user asks "why did X change?" with a date reference.

Each entry is a structured paper trail that lets the LLM explain *why its memory changed*:

```markdown
## [2026-04-11] update | auth-approach.md — The JWT→Sessions Reversal
- **Action:** Updated `decisions/auth-approach.md` — JWT → server-side sessions
- **Source:** `mempalace://wing_prod/drawer/882a`
- **Commit Signal:** "Migrate to sessions" — explicit migration verb after 3 failed JWT debug sessions
- **Reasoning:** User stated intent to migrate, confirmed in 2 subsequent sessions referencing "the sessions migration"
- **Previous State:** Archived to `wiki/archive/auth-jwt-2026-04.md`
- **Volatility at time of update:** V = 0.3 (3 contradictions / 10 days)

## [2026-04-10] promote | rate-limiting-strategy.md — Shadow → L1
- **Action:** Promoted from `shadow/` to `decisions/`
- **Confidence Score:** 5 (5 positive confirmations, 0 corrections)
- **Source:** `mempalace://wing_orion/drawer/7b2c`
- **Reasoning:** Used in 6 queries, user confirmed helpful each time

## [2026-04-09] ingest | new-source-graphql-evaluation.md
- **Action:** Ingested raw source, created shadow article
- **Source:** `raw/articles/graphql-vs-rest-2026.md`
- **Articles touched:** `shadow/api-design-approach.md` (new), `decisions/why-we-chose-rest.md` (added cross-ref)
```

When the user asks "Wait, didn't we use JWT?" the LLM reads `log.md` and can reconstruct the full story: what changed, when, why, what the previous state was, and where the old version is archived.

---

## The Distillation Pipeline (L2 → Shadow → L1 Promotion)

### The Shadow Wiki (L1-Drafts)

Before information reaches L1, it enters `wiki/shadow/` — a staging area for low-confidence articles.

**How it works:**
1. Distillation creates a new article → it lands in `shadow/`, not L1 proper
2. Shadow articles have `confidence: draft` in frontmatter. Confidence score tracked in `wiki_state.json` (not frontmatter — keeps git clean)
3. Score changes based on **explicit linguistic markers only** — never inferred from context:

| Signal | Score change | Detection |
|--------|-------------|-----------|
| User says "correct", "thanks", "that's right", "yes", "commit this" | **+1** | Explicit positive marker |
| User pastes code/config matching the shadow article's content | **+2** | Implementation evidence (strongest signal) |
| User says "no", "wrong", "actually", "that's not right", "look at the error" | **-1** | Explicit negative marker |
| Follow-up questions, code copying, silence, topic change | **0** | Neutral — do NOT change score |

4. At **5 points** → **lazy confirmation, not auto-promote.** Next time the user queries this topic, surface: "This answer came from a draft article that's been helpful 5 times. Promote to permanent wiki?" Only move to L1 on explicit "yes."
5. At **-3 points** → flagged for deletion or major rewrite

**Why strict markers:** Users rarely say "that was helpful." They just move on. If you let the LLM infer feedback from context, the confidence score becomes noise. Neutral = 0 prevents shadow articles from being promoted (or killed) by silence. The +2 for implementation evidence is the strongest signal because it means the user actually *used* the knowledge, not just acknowledged it.

**Why lazy confirmation for promotion:** The Human Review Boundary says "anything that changes what the system knows requires approval." Auto-promotion at score 5 violates this. Lazy confirmation surfaces the approval at the natural moment (when the user is already thinking about that topic) without requiring a separate review workflow.

```yaml
# Shadow article frontmatter (static only — scores live in wiki_state.json)
---
sources: [mempalace://wing_orion/drawer/7b2c]
confidence: draft
created: 2026-04-11
triggers: [rate limiting, throttling, API limits]
---
```

### Flow

```
MemPalace (L2)
    │
    ▼
[Periodic scan: new high-value content?]
    │
    ▼
[Check anti-patterns.md — is this a known dead end?]
    │
    ▼
[Draft wiki article — Pass 1: summarize with citations]
    │
    ▼
[Verify — Pass 2: check claims against sources]
    │
    ▼
[Name the article — functional naming + generate triggers]
    │
    ▼
wiki/shadow/  (L1-Draft, confidence_score: 0)
    │
    ▼
[User queries hit shadow articles naturally]
    │
    ├── User confirms helpful → +1 point
    ├── User corrects → -1 point
    │
    ▼
[confidence_score ≥ 5]
    │
    ▼
[Move to L1 proper — update wing index.md]
    │
    ▼
[Lint — check for contradictions, duplicates, scope conflicts]
    │
    ▼
Wiki (L1)
```

### Promotion Predicates (What Gets Distilled)

Not everything in L2 deserves even a shadow article. **All predicates must pass:**

| Predicate | Test | Why |
|-----------|------|-----|
| **Durability** | Referenced in ≥ 2 separate sessions | One-off mentions aren't worth curating |
| **Reuse** | Contains explicit decision language OR reused in ≥ 2 contexts | Must have ongoing relevance |
| **Evidence** | Source confidence ≥ 2 (human speaker, verified sources) | No AI slop promotion |
| **Currency** | Not contradicted by more recent Class A/B evidence | Don't promote stale knowledge |
| **Novelty** | Not already covered by an existing L1 or shadow article | No duplicates |
| **Non-toxic** | Not present in `anti-patterns.md` | Don't re-promote known dead ends |
| **Sensitivity** | From green-tier L2 wing (or yellow with explicit consent) | Respect privacy boundaries |

**Promotion = durability + reuse + evidence.** This replaces the subjective "decisions, patterns, key learnings" with testable conditions.

What stays in L2 only:
- Fails durability (mentioned once, never revisited)
- Fails evidence (assistant-only source, unverifiable)
- Debugging logs, exploratory conversations, venting, one-off questions

### The Lint Pass (Tiered)

The full lint is too expensive to run on every commit. Tier by frequency:

**Pre-commit (fast, every save):**
- Broken wikilinks (renamed/evicted targets)
- Frontmatter schema validation (required fields per article type)
- Missing `ttl:` → auto-inject default based on folder
- File-existence check on cited sources (no content verification)

**Weekly (moderate, background cron):**
- Contradictions between articles **(skip sources in `lint_ignore.json`)**
- Stale claims superseded by newer sources
- Orphan pages with no inbound wikilinks
- Articles with high miss rates (from `.metrics/miss_log`, threshold ≥ 3 misses in 7 days)
- Citations pointing to `speaker: assistant` fragments (AI slop risk)
- Global `index.md` token count exceeds 1,500 → consolidation warning
- Recalculate `wiki_state.json` (volatility scores, answer modes, probe states)

**On-demand (expensive, user-triggered via `mempalace lint --deep`):**
- Scope fragmentation check (>80% structural similarity → merge suggestion)
- Deep citation verification: primary claims (`chosen`/`solution` fields) keyword-matched against source content. Secondary claims (rejected alternatives, rationale) only verified if primary claim fails.
- `[unverified]` claims checked against L2 for newly available sources
- Cross-project propagation checks (if multi-scope enabled)

**Staleness tiers for claim verification:** Don't deep-verify all 1,500 claims across 100 articles. Only verify the `chosen` or `solution` field (the primary claim) during weekly lint. Secondary claims are verified only if the primary claim fails or the article is flagged for a merge conflict. This keeps lint from timing out or bankrupting the context window.

**Lint Negation Registry (`.mempalace/lint_ignore.json`):**
When a user resolves a conflict as "Keep L1, ignore this L2 source," the system writes the source hash to `lint_ignore.json`. The lint pass skips these sources in all future runs. Without this, resolved contradictions resurface every lint cycle and the system becomes a spam generator.

```json
{
  "ignored_sources": [
    {
      "source_id": "mempalace://wing_prod/drawer/vent-101",
      "hash": "a3f2b1...",
      "reason": "Flagged as venting, not actionable",
      "resolved_date": "2026-04-10",
      "resolved_by": "user"
    }
  ]
}
```

---

## Implementation Path

### Phase 0: Cold Start (Reverse Pipeline)
- **Don't build the wiki first.** Start with L2 capture only.
- `pip install mempalace && mempalace init` with wings for each project
- Configure sensitivity tiers in `wing_config.json` (green/yellow/red)
- Mine existing conversation exports into L2 (using strict mining prompts — outcomes only, `PROMOTION_SKIPPED` for inconclusive conversations)
- Configure MCP server for Claude Code access
- Create `.gitignore` with `.mempalace/` excluded
- **Use for 2-4 weeks.** Build up L2 naturally through normal work.

### Phase 1: Wiki Bootstrap (Informed by L2)
- Write `wiki/identity.md` (L0) — role, default stance, venting pattern, commit threshold, behavior modes
- Run `mempalace distill --dry-run` → identify top topics from actual conversation patterns
- Create folder structure (`raw/`, `wiki/` with wing subdirs, `wiki/shadow/`, `wiki/archive/`, `CLAUDE.md`)
- Write hierarchical index structure — global `index.md` (with `## Recently Active` section) + per-wing `index.md` files
- Write CLAUDE.md schema: naming rules, source grounding, frontmatter format (including ADR lifecycle states, probe cache, temporal fields, wikilink resolution), shadow promotion rules, sensitivity tier rules, mining strictness rules
- Create `wiki/anti-patterns.md` with initial known dead ends
- Create `wiki/log.md` with structured audit format
- Initialize `wiki/.metrics/query-stats.json` and `wiki/.metrics/miss_log.json`
- Initialize `.mempalace/lint_ignore.json` and `.mempalace/resolution_log.db`
- Auto-generate shadow articles from L2 patterns (user approves which ones)
- Confidence-score shadow articles into L1 proper through natural usage

### Phase 2: The Bridge
- Build distillation prompt (L2 green wing → Shadow, with anti-pattern check + sensitivity filter + strict outcome extraction)
- Build shadow → L1 promotion logic (confidence scoring, 5-point threshold)
- Build lazy invalidation probe with **probe cache** (skip probes when `contradiction_likelihood < 0.1` and `probe_ttl` not expired; always probe on negation markers)
- Build cache-miss detector (aggregated — log to `miss_log`, surface at ≥3 misses/7 days threshold)
- Build `wiki-lint` pre-commit hook (typed schema validation, wikilink resolution, citation + speaker verification, scope fragmentation check, token budget check, lint negation registry)
- Configure auto-save hooks for ongoing capture

### Phase 3: Feedback Loops
- Cache-miss logging → automatic trigger word updates
- Shadow confidence tracking → automated L1 promotion
- Resolution reasoning → personalized invalidation priors
- Volatility scoring → dynamic confidence hedging
- Drift detection → conversation-count sentiment tracking (lightweight, no embeddings)
- Lint cron → wiki hygiene (orphans, broken wikilinks, contradictions, stale articles, anti-pattern re-loops)
- Structured audit log → explainable memory changes
- Metrics dashboard → L1 hit rate, stale read rate, shadow promotion rate, naming friction, probe skip rate

---

## Summary of Mechanisms

| Problem | Primary Strategy | Refinement |
|---------|-----------------|------------|
| **Execution determinism** | Informal component descriptions | **Read Path + Write Path state machines** with exact steps and thresholds |
| **Routing determinism** | "Names are the protocol" | **Scoring model** (0–18) with hard gates at 9/6 |
| **Answer consistency** | Ad-hoc hedging | **5 answer mode templates** chosen by 4-component confidence model (0–12) |
| **Source conflicts** | Implicit ordering | **Precedence ladder** (6 tiers) + **evidence classes** (A/B/C/D with weights) |
| **Automation boundary** | Undefined | **Human review matrix** — routing/metrics auto-commit; knowledge changes require approval |
| Index scaling | Single `index.md` | **Hierarchical two-tier index** + hot tracking + **1,500 token budget** |
| Routing accuracy | Aliased triggers | **Functional naming + cache-miss feedback + wikilinks + scoring model** |
| Knowledge re-looping | None | **Anti-pattern capture + negative knowledge in KG** |
| Promotion quality | Periodic scan → subjective | **Shadow wiki** + **promotion predicates** (durability + reuse + evidence) |
| Invalidation cost | Lazy probe on every read | **Probe cache** + negation markers + evidence class weighting |
| Invalidation accuracy | Commit signals only | **Evidence classes** (A/B/C/D) + volatility scoring + drift detection |
| Decision tracking | Binary active/contested | **ADR lifecycle** (proposed → accepted → deprecated → superseded → **evicted**) |
| Knowledge deletion | Not addressed | **Eviction + tombstones + retraction propagation** |
| AI slop in L2 | Trust all L2 equally | **Speaker attribution** — human-only foundational sources |
| Distillation cost | Two full LLM passes | **Sampling-based Pass 2** |
| Provenance granularity | Per-article | **Per-claim** inline source IDs `[s1]` with verification dates |
| Article consistency | Shared frontmatter | **Typed schemas** — Decision/Pattern/Entity/Synthesis with required fields |
| Confidence model | Single "confidence" field | **4 components:** source, freshness, routing, conflict (0–12 total) |
| Freshness vs truth | Conflated | **Separated:** old + uncontested = trustworthy; new + contested = suspect |
| Ingestion integrity | Not addressed | **Content hashing for dedup + tombstones + retraction propagation** |
| Sensitive data | None | **Sensitivity tiers** (green/yellow/red) with pipeline access control |
| Cross-references | Relative paths | **Wikilinks** `[[topic]]` resolved via index triggers |
| Cold start | Seed manually | **Reverse pipeline** — L2 first, distill after 2-4 weeks |
| L3 routing | Undefined | **Explicit rules** — temporal, contiguous, low-confidence, provenance |
| Schema enforcement | CLAUDE.md honor system | **wiki-lint** + typed schema validation + claim-level citation checks |
| Observability | None | **Metrics** — hit rate, miss rate, promotion precision, answer latency, token cost |
| Architecture justification | Token savings | **Cognitive scaffolding** — curated knowledge beats raw retrieval regardless of context size |
| Multi-project scaling | Single-project only | **Triple-tier** (Global → Project → Module) with cascade reads + agent scoping |
| Cross-project insights | Not addressed | **Upward propagation** (pattern reuse → Global), **downward flagging** (standard change → project review) |
| Tool portability | Claude Code only | **CLAUDE.md** + symlinks for `.cursorrules`, `AGENTS.md` |

---

---

## Critique Evaluation & Responses

### Critical Gaps

**1.1 Index.md Bottleneck** — ✅ Already addressed
Our hierarchical two-tier index (global + wing indices) solves this. **New addition worth taking:** "hot" article tracking. The global index should surface recently-accessed articles, not just categories. Add a `## Recently Active` section auto-updated on each query.

**1.2 Query-Time Tax (Lazy Invalidation Cost)** — ✅ Real gap. Integrate.
Firing a contradiction probe on *every* L1 read is expensive. The fix: a **probe cache** in article frontmatter.

```yaml
---
last_l2_probe: 2026-04-10
contradiction_likelihood: 0.02
probe_ttl: 7 days  # Skip probes if likelihood < 0.1 and TTL not expired
---
```

Rules:
- If `probe_ttl` hasn't expired AND `contradiction_likelihood < 0.1` → skip the probe, answer from L1 directly
- If query contains terms NOT in the trigger list (novel angle) → always probe regardless of TTL
- If query contains **linguistic negation/revision markers** → always probe: "did we change...", "wait, are we still...", "I thought we stopped...", "didn't we switch..."
- Each probe updates `contradiction_likelihood` as a rolling average

This turns lazy invalidation from "always probe" to "probe when it's likely to matter."

**1.3 Venting Classification Trap** — ⚠️ Concept yes, implementation no.
The "gradual drift" problem is real — no single commit signal, but slow erosion of a position across multiple conversations. Weekly embedding + cosine distance tracking is too heavy for a personal system.

**Lighter alternative: conversation-count drift detection.**
- Track how many L2 conversations mention a topic positively vs negatively (simple sentiment, not embeddings)
- If the ratio flips over 30 days (was 80% positive, now 60% negative), flag as "organic drift"
- Present: "Your thinking on auth seems to be shifting — 4 recent conversations express skepticism about JWT. Review?"

No vector infrastructure needed. Just counting.

**1.4 Multi-Session Concurrency** — ❌ Over-engineered for personal use.
The append-only log → compiled views architecture is elegant but solves a problem that barely exists in a personal wiki. You're not running 10 agents writing to the same file simultaneously. Git handles the rare collision fine. If this ever becomes a problem, address it then.

**Rejected:** The `log/ → compiled/` model adds significant complexity (build step, generated files, two sources of truth) for a marginal concurrency benefit.

**1.5 PII & Sensitive Data** — ✅ Real gap. Integrate (simplified).
Full Presidio-level encryption is overkill for personal use, but sensitivity awareness is important.

**Simplified approach: tag-based sensitivity tiers.**

```yaml
# In MemPalace wing config
wings:
  green: [architecture, tech-stack, patterns]     # Safe for distillation
  yellow: [performance-reviews, salary, health]   # Distill only with explicit consent
  red: [passwords, api-keys, legal]               # Never distill, auto-scrub on ingest
```

- Distillation pipeline only pulls from green wings by default
- Yellow requires explicit `--include-sensitive` flag
- Red is auto-scrubbed on ingest (regex for API keys, passwords, tokens) and never promoted
- No encryption infrastructure — just access control in the pipeline

### Architectural Improvements

**2.1 Decision Lifecycle States** — ✅ Integrate.
ADR-style states are a strict improvement over binary active/contested:

```yaml
---
status: proposed | accepted | deprecated | superseded-by: auth-v2.md
decision_date: 2026-04-01
review_date: 2026-07-01  # Auto-ping for validation
---
```

Key insight: "venting" almost always happens in `proposed` state. `accepted` requires higher evidence (commit signals). This naturally raises the bar for invalidation — you don't contest an `accepted` decision based on one frustrated comment.

**2.2 Synthesis Layer (L1.5)** — ✅ Already addressed.
This is our Shadow Wiki (`wiki/shadow/`) with confidence scoring. The critique's `.staging/` directory is the same concept renamed. No changes needed.

**2.3 Cross-Reference Integrity** — ✅ Integrate.
Wikilinks beat relative paths:

- Use `[[auth-approach]]` instead of `../decisions/auth-approach.md`
- Index system resolves triggers to current filenames
- Renaming `auth-approach.md` → `jwt-strategy.md` updates the index; all `[[auth-approach]]` links auto-resolve via trigger matching
- Broken link detection added to lint pass

**2.4 Temporal Scoping** — ✅ Integrate.
MemPalace already has temporal queries via knowledge graph. Extend to L1:

```yaml
---
valid_from: 2026-01-15
valid_until: null  # Currently active
supersedes: auth-approach-v1.md
---
```

Query: "What was our auth approach in Q1?" → System checks `valid_from`/`valid_until` across article versions, or falls back to L2 temporal search.

Combined with `wiki/archive/` (where superseded articles live), you get full temporal navigation.

### Implementation Gaps

**3.1 Cold Start Problem** — ✅ Real gap. Integrate.
You can't seed L1 without knowing what's valuable. The reverse pipeline:

1. **Week 1-4:** L2 capture only. No wiki. Just use MemPalace normally.
2. **Week 4:** Run `mempalace distill --dry-run` → suggests "Top 5 topics that keep coming up"
3. **User approves** → auto-generates shadow articles from L2 patterns
4. **Shadow articles earn their way** into L1 via confidence scoring

This eliminates the blank-page problem. The system tells you what to write based on what you actually talk about.

**3.2 Metrics & Observability** — ✅ Integrate.
Essential for knowing if the system is working:

**MVP metrics (always tracked — side-effects of the read path, zero extra cost):**

| Metric | What it measures | Healthy range |
|--------|-----------------|---------------|
| **L1 hit rate** | % of queries resolved by wiki alone | >70% |
| **Stale read rate** | % of L1 answers where article freshness was STALE or volatility was HOT | <15% |
| **Miss-but-existed** | % of L2 fallbacks where answer was in a badly-routed L1 article | <5% |

**Extended metrics (tracked if you care, all derived from read path side-effects):**

| Metric | What it measures | Healthy range |
|--------|-----------------|---------------|
| Shadow promotion rate | % of shadow articles that reach L1 | 30-60% |
| Probe skip rate | % of queries where probe cache avoided L2 round-trip | >50% |
| Contested article frequency | % of L1 articles currently in contested state | <10% |

**Diagnostic metrics (computed on-demand by `mempalace stats`, not continuously tracked):**

| Metric | What it measures |
|--------|-----------------|
| Promotion precision | % of promoted articles that stay in L1 after 30 days |
| Invalidation false positives | % of probe-triggered contests resolved as "keep original" |
| Mean article age by type | Average days since last_verified |
| Claim verification coverage | % of primary claims with verified provenance |
| Answer latency by path | Tokens consumed per answer by execution path |

Stored in `wiki/.metrics/` as simple JSON, updated as side-effects of the read path. The rule: if a metric wouldn't change your behavior, don't track it continuously.

**3.3 CLAUDE.md Enforcement** — ✅ Integrate.
LLMs are unreliable at self-enforcing schema. Add a lint layer:

- **Pre-commit hook:** `wiki-lint` checks frontmatter schema, broken wikilinks, missing citations, `[unverified]` claims
- **Citation verification:** Check that cited `raw/` files actually contain the claimed content (fuzzy match, not exact)
- **Shadow gate:** Shadow articles missing required frontmatter fields can't be promoted

This is the "trust but verify" layer. The LLM writes, the linter checks.

### Strategic Concerns

**4.1 Capture Everything Assumption** — ⚠️ Valid concern, optional mitigation.
The `#remember` tag approach fundamentally changes MemPalace's philosophy. For personal use, "capture everything" is fine — you control your own data. 

**Compromise: configurable capture mode.**
- `capture_mode: all` (default) — everything goes to L2
- `capture_mode: tagged` — only `#remember` tagged content
- `capture_mode: summary` — session summaries only, no verbatim

This lets privacy-sensitive users opt into selective capture without changing the default architecture.

**4.2 Overfitting to Context Limits** — ✅ Critical reframe. Integrate.
This is the most important strategic point. The L1/L2 distinction should NOT be justified by token savings alone. With 1M+ context windows, you could dump everything into context. But:

**L1's real value is cognitive scaffolding, not token efficiency.**
- Curated knowledge produces better reasoning than raw retrieval, regardless of context size
- A 200-token wiki article that says "we use JWT because X, Y, Z" produces better downstream reasoning than 10K tokens of raw conversation fragments about JWT
- L1 is a *compression of understanding*, not just a compression of tokens
- Even with infinite context, the LLM reasons better from structured knowledge than from a haystack

**Reframe:** L1 = pre-computed understanding. L2 = evidence archive. The architecture survives the death of context limits.

### Revised Folder Structure Evaluation

**The append-only log → compiled views model:** Rejected for core architecture (see 1.4). Too much complexity for personal use.

**Sensitivity tiers in MemPalace:** Accepted (simplified).

**The `.staging/` directory:** Already exists as `wiki/shadow/`.

**Net result:** Our current folder structure with the following additions:
- `wiki/.metrics/` for observability
- Sensitivity tags in MemPalace wing config (not separate directories — just metadata)
- `wiki/.hot-cache.md` or `## Recently Active` section in global index

### Critique Round 2

**Gap 1: Index Token Budget** — ✅ Partially addressed, now fully integrated.
We had hierarchical routing but no hard limit. Added **1,500 token budget** on global `index.md` with consolidation lint trigger. The budget is the forcing function — without it, index bloat silently degrades L1 back to "dump everything."

**Gap 2: AI Slop in L2 (Compression of Compression)** — ✅ Excellent catch. Integrated.
L2 conversations mix human and assistant utterances. Assistant summaries can be subtly wrong. New hard rule: only `speaker: human` fragments serve as foundational sources for L1 claims. Assistant fragments provide context but never sole citation. `wiki-lint` verifies speaker attribution on citations.

**Gap 3: The Forget Operation** — ✅ Real gap. Integrated.
`deprecated` and `superseded` assume a successor exists. `evicted` is for "this was never the right path." Evicted articles are stripped from all indices (invisible to routing), archived with warning banners, and flagged if the topic resurfaces in future L2 conversations.

**Friction 1: Probe Cost** — ✅ Already addressed, refined.
Probe cache was already in place. Added **linguistic negation markers** ("did we change...", "are we still...") as additional always-probe triggers alongside novel terms.

**Friction 2: L3 Undefined** — ✅ Real gap. Integrated.
Added explicit L3 routing rules: temporal queries, contiguous context requests, low-confidence L2 escalation, and provenance verification.

**Friction 3: Two-Pass Cost** — ✅ Good optimization. Integrated.
Pass 2 is now sampling-based: file existence + keyword match + speaker attribution check. Full LLM semantic verification only fires when the cheap checks fail.

**Nit: Folder vs. Naming Tension** — ✅ Clarified.
Folders = knowledge type (for narrowing search). Titles = functional questions (for routing within a narrowed set). Both serve the LLM but at different stages of the lookup.

**Nit: CLAUDE.md Portability** — ✅ Footnoted.
Added note on `.cursorrules`, `AGENTS.md`, and symlink strategy for multi-tool support.

**Nit: Auto-injected TTL** — ✅ Integrated into lint pass.
Lint auto-injects default TTL based on folder if missing from frontmatter. The human never guesses.

### Critique Round 3

**1. Runtime state machine** — ✅ The biggest structural gap. Integrated.
Added full Read Path (7-step state machine) and Write Path (8-step state machine) with exact inputs, outputs, thresholds, and transitions. Two implementations must now behave identically.

**2. Deterministic routing policy** — ✅ Integrated.
Added 5-component scoring model (0–18) with hard gates: ≥9 L1 confident, 6–8 L1 hedged, <6 L2 fallback. Tie-breaking defined.

**3. Promotion criteria too qualitative** — ✅ Integrated.
Replaced subjective categories with 7 testable predicates (durability, reuse, evidence, currency, novelty, non-toxic, sensitivity). All must pass.

**4. Commit signals incomplete** — ✅ Integrated.
Added 4 evidence classes (A: decision artifacts, B: implementation evidence, C: operational references, D: sentiment/venting) with explicit weights. Class D alone never triggers invalidation.

**5. Claim granularity underspecified** — ✅ Integrated.
Moved from per-article to per-claim provenance with inline `[sN]` source IDs. Each source entry includes speaker, date, and verification status. Lint can re-verify individual claims.

**6. No source precedence** — ✅ Integrated.
Added 6-tier precedence ladder from explicit human override (highest) to assistant-generated content (lowest). Tie-breaking: prefer recency, then corroboration.

**7. No article type schemas** — ✅ Integrated.
Added typed schemas for Decision, Pattern, Entity, and Synthesis with required fields per type. Lint validates type-specific requirements.

**8. No ingestion/dedup/deletion model** — ✅ Integrated.
Added content hashing for dedup, tombstones for deleted sources, retraction propagation to downstream L1 articles, archive vs delete semantics, and L2 rebuild strategy.

**9. Observability gaps** — ✅ Expanded.
Metrics table expanded from 5 to 12 metrics including promotion precision, invalidation false positives, claim verification coverage, answer latency by path, and token cost by path.

**10. Security/privacy/access control** — ⚠️ Partially addressed.
Sensitivity tiers (green/yellow/red) from Round 1 cover the personal-use case. Full per-project ACL and cross-scope leakage prevention is deferred until multi-user is in scope. Noted as a future concern.

**Design improvements:**
- **A. 4-component confidence model** — ✅ Integrated (source, freshness, routing, conflict → 0–12 total)
- **B. Split freshness from truth** — ✅ Integrated into confidence model (old+uncontested = trustworthy)
- **C. Negative knowledge** — ✅ Integrated (anti-patterns + eviction + negative KG assertions)
- **D. Answer mode templates** — ✅ Integrated (5 explicit modes: direct, hedged, contested, partial+expand, cannot-answer)
- **E. Human review boundary** — ✅ Integrated (explicit matrix: routing/metrics auto-commit, knowledge changes require approval)

---

## Key Principles

1. **L1 is cognitive scaffolding, not token savings.** Even with infinite context, curated knowledge produces better reasoning than raw retrieval. The architecture survives the death of context limits.
2. **Wiki is L1, MemPalace is L2.** Pre-computed understanding first, evidence archive as fallback.
3. **Names are the routing protocol.** Optimize for LLM readability with hierarchical aliased index + functional naming + wikilinks.
4. **Contradiction != Invalidation.** Require commit signals + volatility scoring, not just semantic overlap. Respect decision lifecycle states.
5. **Lazy over proactive.** Query-time checks with probe caching beat background scanning for personal-scale systems.
6. **Never lie confidently.** Degrade gracefully from "here is the answer" to "here is what I thought, but something might have changed."
7. **The wiki summarizes, it never generates.** Every claim cites a source or is marked unverified. The linter enforces what CLAUDE.md declares.
8. **Shadow before spotlight.** New knowledge earns its way into L1 through user confirmation, not LLM confidence.
9. **Remember why not.** Anti-patterns are first-class knowledge. Capturing failures prevents re-looping.
10. **Explain your changes.** Every mutation to L1 is logged with action, source, reasoning, and previous state.
11. **Start with L2, not L1.** The cold start problem is real. Let conversation patterns tell you what's worth curating.
12. **Feedback loops are the product.** Cache misses fix naming. Confidence points filter promotion. Resolution reasoning trains invalidation. Metrics reveal system health. The system improves by being used.
13. **Deterministic execution.** Two implementations of this spec must produce the same routing decisions, answer modes, and promotion outcomes. The state machines are the spec, not the prose.
14. **Separate freshness from truth.** Old + uncontested = trustworthy. New + contested = suspect. The 4-component confidence model captures this.
15. **Silence over noise.** Mining returns `PROMOTION_SKIPPED` when no outcome is detected. Notifications are aggregated and threshold-gated. Resolved contradictions are permanently suppressed. The system earns attention by being precise, not loud.
16. **Specificity wins, cascade before fallback.** In multi-scope mode, module overrides project overrides global. Exhaust all L1 tiers before hitting L2. The cascade is cheap; L2 is not.

---

## Appendix A: Known Limitations & Accepted Tradeoffs

These are not bugs to fix — they are tradeoffs the design deliberately makes.

### 1. Curation Fatigue is the Dominant Failure Mode
The system's integrity depends on the human being willing to act as memory editor-in-chief. Rename suggestions, conflict resolutions, evictions, anti-pattern additions, and sensitivity-tier choices create ambient interruption. The design mitigates this (lazy surfacing, aggregated notifications, silent logging) but cannot eliminate it. If the human stops reviewing, the wiki decays.

### 2. False Precision in Scoring
All numerical thresholds (routing gates at 9/6, promotion at 5, eviction at -3, volatility ranges) are invented starting points. They create the illusion of determinism around fuzzy judgment. Treat them as defaults to be tuned by observation, not as ground truth. The pre-computed label approach (STABLE/WARMING/HOT) is partly a mitigation — the LLM never sees the numbers, just the labels.

### 3. Taxonomy Fragility Replaces Vector-Search Fuzziness
The hand-built routing layer (triggers, aliases, wing indices) can fail quietly and repeatedly once topics become messy, overlapping, or cross-cutting. The cache-miss feedback loop and miss aggregation help, but the index is still a knowledge artifact that must be maintained. At scale (500+ articles), consider whether lightweight vector search over L1 articles (not L2) would be more resilient than hand-curated triggers.

### 4. Decisions Often Shift by Accretion, Not Announcement
The contradiction model assumes truth changes in declarative events ("Final decision: X"). In reality, decisions often drift through gradual behavioral change (increasingly working around JWT without ever announcing a switch). Volatility scoring and drift detection partially address this, but the system will still lag behind reality for decisions that happen implicitly.

### 5. Probe Cache Staleness Window
Newly ingested L2 conversations may contain contradictions that won't be detected until the article is next queried AND the probe TTL has expired. This is by design (proactive scanning is avoided). If immediate freshness is required after a known decision change, manually run `mempalace review [article]` or ask a question that triggers the probe.

### 6. Code Truth is Underweighted
The system is conversation-centric. For engineering work, what was *implemented* often matters more than what was *discussed*. Code, config, tests, migrations, and merged PRs should be privileged over conversation fragments. The `raw/` directory and Class B evidence (implementation evidence) partially address this, but the philosophical center still revolves around conversation preservation.

### 7. Anti-Patterns Can Harden Too Aggressively
Rejected alternatives recorded in `anti-patterns.md` can become sticky. Something rejected under one set of constraints may become viable later. Anti-pattern entries should include their original context/constraints so future agents can assess whether the rejection still applies.

### 8. Shadow Stagnation Risk
Shadow articles may stall at low confidence scores indefinitely if the user rarely provides explicit feedback. The strict heuristic approach (explicit markers only) prevents noise but also limits data flow. If shadow stagnation becomes a problem, consider adding a **usage-frequency fallback**: if the same shadow article is retrieved for ≥ 3 distinct queries without correction, treat that as weak implicit confirmation (+1).

### 9. The `raw/` Directory Ingestion Gap
`raw/` is user-managed; the system reads from it but never modifies it. There is no automatic ingestion workflow (no PDF chunking, no web clipper integration). New files are added manually or via external automation (Obsidian Web Clipper, download scripts). The system detects new files during distillation passes and can offer to create shadow articles summarizing them.

### 10. Multi-Agent Concurrency is Not Addressed
The architecture assumes single-agent or sequential-agent access. Concurrent writes from a 12-agent team would require a dedicated **L1 Curator Agent** that owns the Write Path (other agents submit for review, not write directly). This is out of scope for MVM and early usage.

---

## Appendix B: Progressive Enhancement Path

Add mechanisms in this order, triggered by real pain:

| Stage | Trigger | What to Add |
|-------|---------|------------|
| **MVM** | Day 1 | Flat wiki, simple index, L2 fallback, basic mining, 2 metrics |
| **+Shadow** | Distillation quality is too noisy | Shadow staging with confidence scoring |
| **+Typed schemas** | Article quality variance is painful | Decision/Pattern/Entity schemas with required fields |
| **+Probe caching** | L2 probe cost becomes noticeable | `wiki_state.json` with pre-computed probe states |
| **+Volatility** | Contradictions fire too often or too rarely | Volatility scoring, evidence classes, drift detection |
| **+Claim-level provenance** | Per-article citations aren't granular enough | Inline `[sN]` citations, tiered verification |
| **+Multi-scope** | Working on 3+ projects with shared standards | Global tier, cascade reads, agent scoping |
| **+Resolution log** | Resolving conflicts and wanting to remember why | `wiki/resolutions/` with git-tracked reasoning |
| **+Full lint** | Wiki has grown to 50+ articles and quality is drifting | Tiered lint (pre-commit / weekly / on-demand) |
| **+Curator agent** | Running 3+ agents with concurrent write needs | Centralized Write Path owner |

**The full spec below is the reference for what each stage looks like when you need it.**

---

## Appendix C: Hard Constraints & Invariants

These are non-negotiable rules that must hold across all operations.

### 1. The L0 Lock
Before any L1 or L2 operations, the system MUST read `wiki/identity.md`. L0 calibrates all downstream behavior — evidence weighting, venting detection thresholds, commit signal sensitivity. Without L0, the system has no personality and misinterprets user intent.

### 2. Mining Silence
The distillation pipeline MUST return `PROMOTION_SKIPPED` if no decision, outcome, or definitive insight is detected in the L2 source. Do not create wiki articles that say "We discussed X." Topic summaries without conclusions are noise, not knowledge.

### 3. The `.gitignore` Mandate
`.mempalace/` is local-only state (ChromaDB binaries, resolution logs, lint ignore lists). It MUST NOT be committed to version control. `wiki/` and `raw/` are the distributable source of truth. Phase 0 setup must create `.gitignore` with `.mempalace/` before any other operations.

### 4. Lint Negation Registry
When a user resolves a conflict as "Keep L1, ignore this L2 source," the system MUST write the source hash to `.mempalace/lint_ignore.json`. The lint pass MUST skip these sources in all future runs. Without this, resolved contradictions resurface on every lint cycle and the system becomes unusable.

### 5. Resolution Storage: Markdown Source, SQLite Cache
Resolution reasoning MUST be stored as git-tracked markdown in `wiki/resolutions/`. SQLite (`resolution_log.db`) is a derived cache rebuilt from markdown — never the source of truth. `git checkout` to any commit must produce a consistent view: wiki articles + their resolution history at that point in time.

### 6. Speaker Attribution Gate
L1 claims MUST NOT be founded solely on `speaker: assistant` L2 fragments. The distillation pipeline, the lint pass, and the citation verifier all enforce this independently. AI summaries from within conversations are context, never ground truth.

### 7. Notification Aggregation
The system MUST NOT surface per-query notifications for cache misses, rename suggestions, or stale article warnings. All notifications are logged silently and surfaced only when aggregation thresholds are met (≥3 distinct misses in 7 days for naming; TTL expiry for staleness). One consolidated suggestion beats ten interruptions.

### 8. Scope Merge Before Split
When scope inference creates a new scoped article, the system MUST first check if the content can be added as a section to an existing article. Prefer sections over files until a file exceeds 200 lines or a genuine conflict between scopes exists. Fragmentation is the default failure mode; merging is the default correction.

### 9. Cascade Before Fallback
In multi-scope mode, the system MUST exhaust all L1 tiers (Module → Project → Global) before falling back to L2. The cascade is cheap (markdown file reads at each tier); L2 is expensive (vector search). Never skip a parent tier to go directly to L2.

### 10. Specificity Wins
When a module-level article contradicts a global standard, the module article takes priority. The module article SHOULD cite why it overrides the global default. The lint pass flags overrides that lack justification but does not auto-resolve them.

### 11. LLM Reads Labels, Never Calculates
The LLM MUST NOT perform arithmetic (scoring, volatility calculation, confidence summation) at query time. All numerical computations happen in background scripts. The LLM reads pre-computed labels (`volatility_state: STABLE`, `answer_mode: direct`, `probe_state: FRESH`) from `wiki_state.json`. This is the boundary between what the LLM is good at (semantic routing, text synthesis) and what it's bad at (multi-variable arithmetic).

### 12. State Separation
Ephemeral machine state (scores, probe dates, hit counts, volatility) MUST live in `.metrics/wiki_state.json`, NOT in article frontmatter. Articles contain only semantic knowledge and static routing (triggers, scope, sources, type schema). This prevents: (a) git diff pollution from metric ticks, (b) LLM accidentally corrupting article text when updating a counter, (c) temporal decoupling of state from content.

### 13. Shadow Promotion is Lazy, Not Auto
Shadow articles reaching confidence ≥ 5 MUST NOT auto-promote to L1. The next time the user queries that topic, surface a lazy confirmation: "This answer came from a draft article. Promote to permanent wiki?" Only move to L1 on explicit user approval. This resolves the tension between "shadow auto-promotion" and "knowledge changes require approval."

### 14. Post-Retrieval Eviction Filter
L2 (ChromaDB) does not understand eviction. When returning L2 results, the system MUST check each chunk against the eviction log and inject warnings for evicted topics. Evicted L2 content is neutralized at retrieval time, not deleted from the vector store.

---

## Appendix D: Critique Evaluations

### Critique Round 5: Multi-Scope Architecture

**The Triple-Tier Hierarchy** — ✅ Core concept integrated. Implementation calibrated.

The proposal for Global → Project → Module scope hierarchy is the right model for multi-project work. However, the implementation needs calibration:

**Accepted:**
- Three-tier scope model (Global, Project, Module) with cascading reads
- Conflict priority (specificity wins): Module > Project > Global
- Cross-project insight propagation (upward: pattern reuse → Global; downward: standard change → project flag)
- Agent scoping (module agents jailed to their directory + parent read access)
- Cascade as a recursive extension of the existing Read Path state machine

**Modified:**
- **Modules are articles, not separate wiki instances.** The proposal suggests each module gets its own `wiki/` + `.mempalace/`. This is 3x the infrastructure per module. Instead: module-level detail lives in `wiki/modules/vpc-connector.md` (a detailed article), and module-level L2 data lives in MemPalace **rooms** within the project wing. This gives the same cascading behavior without infrastructure multiplication.
- **Global tier is optional.** Solo devs with one project don't need it. The "When to Use" matrix defines when each tier activates.

**Pushed back on:**
- **AAAK usage recommendations.** AAAK currently regresses LongMemEval by 12.4 points vs raw mode (84.2% vs 96.6%). Until AAAK demonstrates compression benefits at scale without retrieval loss, recommending "High AAAK at Global tier" is premature. Raw mode remains the default at all tiers.
- **Separate `.mempalace/` per module.** MemPalace rooms already provide module-level isolation within a project wing. A separate ChromaDB instance per module is wasteful.

**The cascade read path** is the most important addition. It extends the existing state machine with a single new step (5c) — on L1 miss, escalate to parent tier before L2 fallback. This is cheap (markdown reads), deterministic (same scoring model at each tier), and composable (works with 1, 2, or 3 tiers).

**The "Lazy Invalidation" cross-tier signal** is a good catch: if Module L2 shows a specific RDS config failed, that should propagate up to Project L1 to prevent other modules from repeating the error. This is handled by the existing upward propagation rule (pattern reuse in ≥2 modules → flag for project-level documentation).

### Critique Round 7: The Existential Critique

**The deepest criticism.** Not about missing mechanisms — about whether the full mechanism set is too heavy to survive.

**Fully accepted:**
- **Core tension acknowledged.** The architecture wants 4 things (cheap retrieval, high integrity, adaptive learning, low burden) and can't maximize all 4. This is now stated at the top of the document.
- **Curation fatigue is the dominant failure mode.** Not indexing failures, not hallucination. People stop using the system because it asks too much. Added to Known Limitations.
- **False precision.** The scoring thresholds are invented, not empirically grounded. Acknowledged explicitly. They should be treated as tunable defaults.
- **The architecture hasn't chosen its primary job.** Different products (fast retrieval, historical reasoning, governed memory) keep adding mechanisms. MVM forces a choice: fast retrieval first, everything else only when pain demands it.
- **Build minimum first.** Added MVM section as the actual build target. Full spec reframed as reference architecture.

**Partially accepted:**
- **Taxonomy fragility vs. vector fuzziness.** True at scale (500+ articles). At MVM scale (~20-50 articles), hand-curated triggers work fine. Added note that lightweight vector search over L1 should be considered if triggers become unmanageable.
- **Code truth is underweighted.** Acknowledged in Known Limitations. The `raw/` directory and Class B evidence partially address this, but the design is still conversation-centric. Future work: privilege implementation artifacts.
- **Anti-patterns can harden too aggressively.** Acknowledged. Anti-pattern entries should include their original context/constraints.
- **Shadow stagnation.** Acknowledged. Added usage-frequency fallback as a future option if strict markers aren't generating enough signal.

**Tactical items addressed:**
- **Agent concurrency** — Out of scope for MVM. Added note: future Curator Agent for multi-agent write coordination.
- **Cascade latency** — Acceptable for personal use (file reads, not network). Query scope hints already specified.
- **PII in distillation** — Sensitivity tiers (green/yellow/red) already in pipeline. Could add a regex sanitization pass between Draft and Shadow; deferred until real need.
- **Shadow confirmation detection** — Strict heuristic approach already specified. Usage-frequency fallback noted as option.
- **Probe cache staleness window** — Explicitly acknowledged in Known Limitations.
- **`raw/` ingestion gap** — Clarified: user-managed, system reads only, no automatic chunking.
- **`anti-patterns.md` format** — Simple markdown list with bullet points and source citations. No typed schema needed.

**Not accepted:**
- **"The system should privilege code/config/tests over conversations."** Directionally right, but the architecture is specifically a *conversation memory* system. Code-first knowledge management is a different product (closer to standard documentation + ADRs). The `raw/` directory can hold code artifacts, and Class B evidence (implementation evidence) gets 4x weight, but rebuilding the architecture around code rather than conversations would be a different project.

---

## Appendix E: Critique Log

| Round | Focus | Key Additions |
|-------|-------|--------------|
| **1** | Refinements | Hierarchical index, shadow wiki, anti-patterns, volatility scoring, structured audit log |
| **2** | Gap analysis | Probe cache, speaker attribution, eviction, L3 routing, sampling-based verification, typed schemas |
| **3** | Execution determinism | Read/write state machines, routing scoring model, 4-component confidence, evidence classes, source precedence, claim-level provenance, ingestion lifecycle |
| **4** | Operational realism | L0 operationalization, mining strictness, miss aggregation, .gitignore mandate, lint negation registry, resolution storage separation, scope merge rules, active TTL degradation |
| **5** | Multi-scope scaling | Triple-tier hierarchy (Global → Project → Module), cascade read path, conflict priority (specificity wins), cross-project propagation, agent scoping |
| **6** | Execution realism | Pre-computed labels (LLM reads, never calculates), state separation (wiki_state.json), post-retrieval eviction filter, strict shadow scoring heuristics, lazy promotion confirmation, tiered lint, rolling log.md, resolution log as git-tracked markdown, L0 mode detection heuristics, simplified MVP metrics |
| **7** | Existential | Acknowledged core tension (4 goals, can't max all). Added MVM (Minimum Viable Memory) as build-first target. Added Progressive Enhancement path. Added Known Limitations appendix. Reframed full spec as reference architecture, not build spec. Accepted curation fatigue as dominant failure mode. |
