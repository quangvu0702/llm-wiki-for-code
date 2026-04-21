# LLM-Wiki for Code Repositories

> Based on Karpathy's LLM-Wiki pattern: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
>
> Adapted for a single goal: give a coding agent **persistent memory of one codebase** — so it stops re-reading the same files on every session.
>
> **How to use this file:** Copy it into your repo. Ask your coding agent (Cursor, Claude Code, Codex, etc.): *"Implement this."* The agent will read your codebase and the last ~60 merged PRs, then bootstrap the wiki (Job Story JS-1). The complete `.cursor/skills/wiki-memory/SKILL.md` is included in Deliverable 1 — copy it directly for Cursor setups.

---

## 0. AI Review Checkpoints

### Review Gates
- [ ] **Gate 1 — Problem direction:** after sections **1–2**
- [ ] **Gate 2 — Deep research:** after section **3**
- [ ] **Gate 3 — Requirements:** after section **4**
- [ ] **Gate 4 — Assumptions + scope + success:** after sections **5–7**
- [ ] **Gate 5 — Implementation readiness:** after sections **8–9**
- [ ] **Gate 6 — Lessons learned + final readiness:** after sections **12–14**

---

## 0. TL;DR (assumes familiarity with Karpathy's LLM-Wiki pattern)

Same three-layer architecture (raw sources / wiki / schema). Adapted for a real production **codebase** with opinionated deviations.

**What's the same:** Agent-maintained wiki of hints (not answers). `index.md` with keyword-rich entries for navigation at small scale. Articles compiled from source code, not docs. Self-correcting through use.

**What's different:**

1. **Schema must be an explicit skill or section, not a passive rule.** In Cursor: `.cursor/skills/wiki-memory/SKILL.md` (recommended — skills fire explicitly when triggered) or `.cursor/rules/wiki.mdc` with `alwaysApply: true` (weaker — rules are passive background context that agents often ignore for behavioral workflows; see "Lessons Learned" below). In Claude Code: a section inside `CLAUDE.md`. In Codex: `AGENTS.md`. Karpathy's version is a plain file the agent must be told to read.
2. **Renamed and expanded operations.** Karpathy has 3 ops (ingest, query, lint). We have 4: **LEARN** (ingest — but only when effort savings >50%, not on every source change), **USE** (query), **SELF-CORRECT** (lint — inline during work, no separate pass), **MERGE** (new — combines articles when the 200-article cap is hit).
3. **Hard article budget (200 max, ~400 words each).** Karpathy's wiki grows unbounded. Ours caps at 200 broad-topic articles. Articles exceeding 400 words get split. This drives the "broad topics not narrow features" rule.
4. **Structured template (6 sections).** Karpathy doesn't prescribe article structure. We enforce 3 mandatory sections (Entry Points, Key Patterns, Search Shortcuts) + 3 optional (Directions, Tests, Gotchas). Multi-layered anchoring (function names + patterns + structural truths) so hints survive renames.
5. **No pre-seeding.** Wiki starts empty, grows organically from real work. Bootstrap reads ~6 months of merged PRs to seed the first ~10–20 articles — but the agent decides which, not a human.
6. **Source of truth is stricter.** Karpathy's wiki can compile from any source. Ours compiles only from **code files and merged PRs** — never from research docs, which reflect intent, not implementation.
7. **Log is write-only.** `log.md` is an append-only audit trail for humans. The agent writes to it but never reads it back.
8. **Token-efficient index entries.** Karpathy's index uses title + one-line summary per entry. Ours uses two lines: **link + pure matchable tokens** (function names, class names, grep patterns, directory paths). No prose, no prefixes — every token is either a structural element or a matchable keyword.
9. **Mandatory CLOSING STEP gate.** Karpathy's pattern has no enforcement mechanism for post-task wiki work. We add a `## CLOSING STEP` section that forces the agent to self-check two questions before ending its turn: *"Did I read 3+ source files?"* (triggers LEARN) and *"Did I mark anything STALE_HINT?"* (triggers SELF-CORRECT). Without this gate, LEARN silently fails — the agent defers wiki work to "after the task" but then stops at the turn boundary.
10. **SELF-CORRECT has known limitations.** Testing proved SELF-CORRECT only fires reliably for catastrophic staleness (wrong file path → file-not-found error). Subtle name mismatches (wrong function name in the right file) are silently resolved — the agent navigates by meaning, not string comparison. Accepted tradeoff: minor drift is harmless (agent still gets correct answers), catastrophic staleness auto-corrects, and periodic audit prompts handle the gap.

**Deliverables the agent will produce:**
1. **Schema** — For Cursor: `.cursor/skills/wiki-memory/SKILL.md` (recommended). For Claude Code: a `## Wiki Protocol` section in `CLAUDE.md`. For Codex: a `## Wiki Protocol` section in `AGENTS.md`. Contains USE / LEARN / CORRECT / MERGE / CLOSING STEP protocols.
2. **`wiki/index.md`** — Categorized catalog; each entry is link + pure matchable tokens (no prose).
3. **`wiki/log.md`** — Append-only audit trail (write-only; agent never reads).
4. **`wiki/TEMPLATE.md`** — Structure & worked example; rules deduplicated into the schema.

## 1. Background

### Current State

Most codebases accumulate hundreds of source files, specs, and research docs — but no **compiled knowledge layer**. Each new session, the AI agent re-derives understanding from scratch — reading large source files to answer questions that have been answered before.

There is no compiled, synthesized knowledge layer between the raw sources and the agent. Every session starts from zero — no hints, no memory, no shortcuts. When asked *"how does feature X work?"*, the agent re-reads the same thousand-line files from scratch, re-derives the same understanding, and produces the same answer it produced the session before. Every task is fully redone. Nothing accumulates.

**Source of truth hierarchy:** The authoritative sources are (1) code files (`.py`, `.ts`, `.go`, etc.) and (2) merged PRs. Research docs and design proposals record ideas at the time of writing, not necessarily the current implementation. The wiki must be derived from code and PRs, not from research docs.

### Trigger

Andrej Karpathy published [llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — a pattern for building persistent, agent-maintained knowledge bases. The insight: instead of re-deriving knowledge from raw sources on every query (RAG), the LLM incrementally builds and maintains a **compiled wiki** that compounds over time. This file applies that pattern to a **single code repository**.

### Domain Context

**RAG (status quo)**: Source code files → embedding → vector search at query time → LLM re-derives answer from scratch every session. No accumulation. No hints, no memory, no shortcuts.

**LLM-Wiki (proposed)**: Source code files → LLM compiles hints → persistent wiki → agent knows *where to look* and *what matters*, reads the right 50 lines instead of 1000. **The wiki is navigation context for the agent, not answers for humans.**

**Three layers (Karpathy)**:
- **Raw sources** — source code files the agent reads but never modifies
- **Wiki** — LLM-maintained markdown files; compact hints and pointers, not complete answers — tells the agent which files, which functions, which patterns matter
- **Schema** — a configuration document (e.g., `.cursor/skills/wiki-memory/SKILL.md`, `CLAUDE.md`, or `AGENTS.md`) that governs how the LLM operates on the wiki

---

## 2. Problem Statement

### What are we solving?

A human developer reads a file, researches a problem, and retains something — a mental note, a direction, a hint about where to look next time. The coding agent does the same work but retains nothing. Every session starts from zero: no memory of what was read, no hints from previous research, no accumulated sense of the codebase. The work is fully repeated, every time.

We want to give the agent memory — not full answers (humans don't write down full answers either), but the distilled, reusable hints that make the next session faster: *which files matter, what the key function is, what the pattern is called, what to avoid.* **No human reads this memory — it exists solely for the agent.**

### What properties must this memory have?

1. **Hints, not answers.** Compact pointers — function names, file references, patterns, short "why" notes — enough for the agent to know *where to look*, not *what the answer is*. This matches how developers take notes.
2. **Self-evaluatable.** Every hint must be specific enough for the agent to verify by reading the referenced source code. Vague hints ("routing is complex") are worse than no hints — the agent can't check them.
3. **Robust to trivial changes.** Hints anchored to function/class names and structural truths, not line numbers. A hint survives when someone adds logging, renames a local variable, or reformats code.

### Why does this matter?

- Every session re-reads the same large source files to answer questions already answered before
- Useful context — *"the routing entry point is `handle_request()` in `router.py`"* — is never saved; re-derived every time
- The agent cannot build on previous sessions' insights the way a developer naturally would
- Architectural decisions and "why" context live only in code comments or are re-inferred each time

### Who is affected?

- **The AI coding agent** — repeats full re-derivation work every session with no carry-over
- **Developers** using AI agents to understand and modify the codebase

---

## 3. Deep Research & Exploration

### Technical Deep Dive

#### Concept 1: How the Wiki Makes the Agent Faster

**Without wiki:** *"How does feature X work?"* → agent reads 3 large source files (say 2,400 lines total) → scans all of them → derives that the entry point is `handle_request()`, uses `ServiceClass`, dispatches via `Handler`. Total: ~2,400 lines read, mostly irrelevant.

**With wiki:** same question → agent reads `index.md` (20 lines) → finds `feature-x.md` → reads it (30 lines of hints: *"entry point: `handle_request()` in `router.py`, dispatches via `Handler` in `handler.py`, state managed by `RequestState` in `state.py`"*) → reads only the referenced functions in those files. Total: ~100 lines read, all relevant.

The wiki doesn't answer the question — it tells the agent *where the answer lives*, so it reads 100 lines instead of 2,400.

#### Concept 2: What Goes in a Wiki Page (Not Too Much, Not Too Little)

A wiki page is a *developer's notebook page*, not a manual. It stores:
- **File pointers**: which files own this concept (by function/class name, not line numbers — line numbers shift on every commit)
- **Entry points**: which function to start reading from
- **Key patterns**: architecture, design patterns, conventions used
- **Structural truths**: *"there are exactly 3 node types"*, *"all handlers inherit from `BaseHandler`"*
- **Short "why" notes**: *"retry logic exists because network calls are flaky"*
- **Cross-references**: *"see also [[other-feature]] for how it's consumed"*
- **Enough context for self-evaluation**: each hint should be specific enough that the agent can verify it by reading the referenced source

**Robustness principle:** Hints must be anchored to function names, class names, patterns, and structural truths — NOT to line numbers or implementation details that change with every minor refactor. If someone adds 5 lines of logging, the hint *"entry point is `handle_request()`"* is still valid. But *"entry point is on line 245"* would break.

It does NOT store:
- Line numbers (they shift constantly)
- Full function implementations (that's what the source code is for)
- Complete answers to questions (the agent derives answers from source code)
- Prose explanations meant for human reading (no human reads this wiki)

#### Concept 3: Self-Evaluation and Self-Correction

The wiki must carry enough information for the agent to *check its own accuracy*. When the agent reads a wiki page during query, it should be able to:

1. **Verify**: Read the referenced source file/function and confirm the hint is still true.
2. **Detect staleness**: If the function was renamed, moved, or deleted, the agent knows the hint is stale.
3. **Self-correct**: The agent updates or flags the wiki page with what it found. No human intervention needed for routine corrections.

Vague hints are worse than no hints. Every hint must point to something the agent can open and check in under 30 seconds of reading.

**Known limitation (validated through testing):** SELF-CORRECT only fires reliably for **catastrophic staleness** — when a file path is wrong and the file doesn't exist, or when the entire referenced concept is gone. For **subtle name mismatches** (e.g., a function was renamed but still exists in the same file), the agent navigates by meaning rather than string comparison and silently resolves the mismatch without correcting the wiki. This means minor drift accumulates until a periodic human audit or until drift becomes catastrophic. Accepted tradeoff: subtle drift is harmless (the agent still finds the right code), and catastrophic drift auto-heals.

#### Concept 4: The Four Wiki Operations (All Autonomous)

**LEARN (was "Ingest")** happens automatically when the agent does real work. After completing a task that required significant exploration, the agent evaluates whether a wiki article would save a future agent most of that effort. If yes, it extracts hints (file pointers, entry points, search keywords, patterns, pitfalls) and writes/updates a wiki page. No human trigger. **Enforcement:** LEARN requires a mandatory CLOSING STEP gate in the schema — without explicit enforcement, agents defer wiki work past their turn boundary and it never happens. The CLOSING STEP checks *"Did I read 3+ source files?"* and triggers LEARN if yes. **Bootstrap mode:** agent reads the last ~6 months of merged PRs to seed the initial articles.

**USE (was "Query")** is the default behavior every session. The agent reads `index.md`, scans the tokens line in each entry to find relevant articles, loads those wiki pages for hints, then reads only the referenced source files. The **tokens line** is what makes routing work — it surfaces function names, class names, grep patterns, and directory paths so the agent can match its query to the right article without opening every page.

**SELF-CORRECT (was "Lint")** happens continuously during normal work. When the agent reads a wiki hint and finds it stale (function renamed, moved, deleted), it corrects the wiki page inline. No separate "lint the wiki" command — **the wiki heals itself through use**. Enforcement: the USE step includes a `DETECT` instruction that tells the agent to mentally mark stale hints as `STALE_HINT`; the CLOSING STEP then checks for this flag and triggers SELF-CORRECT. **Known limitation:** only fires reliably for catastrophic staleness (file-not-found, concept gone). Subtle name mismatches are silently resolved by the agent without wiki correction.

**MERGE** is new. When the wiki hits 200 articles and a new one is needed, the agent consolidates two related articles into one broader article. No knowledge is deleted — only merged and compressed.

#### Concept 5: index.md Is Enough at Small Scale (With Token-Efficient Entries)

Karpathy: *"This works surprisingly well at moderate scale (~100 sources, ~hundreds of pages)."* But a title-only index forces the agent to guess which article is relevant. The bottleneck: articles contain rich, matchable content (function names, class names, grep patterns) in their Entry Points and Search Shortcuts — but a title-only index hides all of it.

The fix: every index entry is two lines — link, then pure matchable tokens:

```
- [feature-x](feature-x.md)
  handle_request, ServiceClass, Handler, RequestState, router.py, handler.py
```

No prose descriptions. No `keys:` prefix. No format instructions inside `index.md` (the schema file already explains the format). At 200 articles × ~15 tokens per entry, the full index is ~3,000 tokens — well within a single read.

#### Concept 6: The Schema File (SKILL.md / CLAUDE.md section / AGENTS.md)

The schema is the agent instruction that tells the agent: the wiki exists, check it first, here's how to use it. Self-contained: constraints, operation workflows, page conventions — all in one place. The agent's "memory bootstrap" for every session.

**Cursor-specific lesson (validated through testing):** `.cursor/rules/*.mdc` with `alwaysApply: true` provides passive context — the agent sees it but ignores behavioral workflow instructions within it. Cursor skills (`.cursor/skills/*/SKILL.md`) are actively invoked when trigger words match, making them reliable for behavioral protocols like wiki operations. **Recommendation for Cursor: use a skill, not a rule.** For Claude Code and Codex, `CLAUDE.md` and `AGENTS.md` are actively read at session start, so rules-in-file works fine.

#### Concept 7: Local Search Upgrade (Optional)

When `index.md` stops being enough (beyond ~200 articles), add a local search tool. [`qmd`](https://github.com/tobi/qmd) is a good option: BM25 + vector + LLM re-ranking over markdown files, local-only, with CLI and MCP server. Add when the wiki outgrows a single-read index.

### Constraints

| Constraint | Impact on Solution |
|------------|--------------------|
| Wiki pages store hints, not full answers | Pages must be compact — file pointers, function names, patterns, short "why" notes |
| Hints must be verifiable | Every hint references a specific function/class/file the agent can check |
| Hints must be robust to trivial changes | Anchor to function/class names and structural truths, not line numbers |
| No human reads the wiki | No need for prose, readability, or onboarding-style writing |
| No pre-seeded wiki pages | Wiki starts empty; grows organically through use |
| `index.md` is enough at < ~200 pages | Past that, add a local search engine like `qmd` |

---

## 4. Job Stories

> **Core principle:** The human never calls any wiki instruction. The agent manages its own memory transparently through rules and skills. The human just works normally — asks questions, requests changes, reviews code. The wiki operates in the background.

### Limits

- **Max 200 articles** — the wiki is compact by design, not an encyclopedia
- **Max ~400 words per article** — forces brevity; if it's longer, it's storing answers instead of hints
- **Only worth updating if the time savings are significant** — if the wiki only saves 10–20% effort, skip it; update only when reading 10 files → reading 2 files

---

### JS-1 — Bootstrap: understand what humans are working on

**When** the wiki is empty or a new major area of work begins,
**The agent** reads merged PRs from the last ~6 months, extracts the main topics and areas of active development, reviews the referenced source code to understand the main flows,
**So** it knows which topics deserve wiki articles — based on what humans actually work on, not abstract completeness.

> **Decision locked:** Bootstrap is triggered once (or when the agent detects large gaps). It produces the initial set of ~10–20 high-value articles. The agent decides which topics matter based on PR frequency and code complexity — not human instruction.

**Acceptance Criteria**
- [ ] Agent reads the last ~60 merged PRs (≈ 6 months) to identify active areas
- [ ] Agent extracts main topics from PR patterns (feature clusters, common files touched)
- [ ] Agent reviews source code for each topic to understand the main flow
- [ ] Agent creates wiki articles only for topics that would save significant effort in future sessions
- [ ] Each article stays within ~400 words: hints, not documentation
- [ ] Agent updates `wiki/index.md` and appends to `wiki/log.md`

---

### JS-2 — Learn while working: create knowledge from real tasks

**When** the agent is doing any coding task and encounters an area it hasn't memorized,
**The agent** searches the wiki first → finds nothing useful → does the work from scratch (reads many files, traces flows, figures out the structure) → after finishing, evaluates whether the effort was significant enough to justify a wiki article,
**So** the wiki grows organically from real work — not from explicit *"create a wiki page"* commands.

> **Decision locked:** The agent decides autonomously whether to write a wiki article. Threshold: if the work required reading many files and significant effort, and a wiki article would save a future agent most of that effort — write it. If the savings would be marginal (10–20%), skip it. No human instruction needed.

**Acceptance Criteria**
- [ ] Agent searches wiki index before starting any codebase exploration
- [ ] When wiki has no relevant hints, agent does the work from scratch — no different from today
- [ ] After completing the work, agent evaluates: *"Would a future agent benefit significantly from hints about this?"*
- [ ] If yes: extracts useful info (entry points, key functions, patterns, search keywords, pitfalls) and writes/updates an article
- [ ] If no (marginal savings): skips the wiki update — no noise
- [ ] Article stays within ~400 words
- [ ] Agent MUST NOT modify source code files
- [ ] Agent updates `wiki/index.md` and appends to `wiki/log.md`

---

### JS-3 — Use memory: read wiki hints before reading source code

**When** the agent needs to work on any part of the codebase,
**The agent** reads the wiki index, loads relevant wiki pages for hints and pointers, then reads only the referenced source files,
**So** the agent starts with a map instead of scanning blind — reads 100 targeted lines instead of 2,400.

> **Decision locked:** This is the default behavior for every session, governed by the schema rule. The agent always checks the wiki first. The human doesn't ask for this — it just happens.

**Acceptance Criteria**
- [ ] Agent reads `index.md` at the start of any codebase exploration
- [ ] Agent reads relevant wiki pages to get hints: which files, which functions, what patterns
- [ ] Agent reads the referenced source files to do the actual work — wiki is the map, source is the territory
- [ ] This happens transparently — the human never asks for it

---

### JS-4 — Self-maintain: correct stale hints during normal work

**When** the agent reads a wiki hint and discovers it's wrong, outdated, or no longer useful (function renamed, file moved, pattern changed),
**The agent** corrects the wiki page immediately as part of its normal workflow,
**So** the wiki stays accurate without any human maintenance or explicit *"lint"* commands.

> **Decision locked:** Self-correction is automatic and continuous. The agent fixes hints as it encounters staleness — no separate lint step needed for routine maintenance. The wiki heals itself through use.

**Acceptance Criteria**
- [ ] When agent verifies a hint against source code and finds it stale → updates the wiki page inline
- [ ] When a hint is no longer useful (concept removed, pattern changed entirely) → removes or rewrites the hint
- [ ] Self-correction happens during normal work — no separate command, no human trigger
- [ ] Agent appends a log entry to `log.md` for any wiki correction
- [ ] Agent MUST NOT modify source code files — only wiki files

---

### JS-5 — Understand the agent's own workflow, then serve it

The wiki must be designed around how the coding agent actually works — not how a human reads documentation. The agent's workflow has a repeating cycle with a clear bottleneck:

```
Agent Workflow (every task)
+---------------------------+
| 1. RECEIVE TASK           |  <- human asks something
+---------------------------+
           |
           v
+---------------------------+
| 2. ASSESS CONFIDENCE      |  <- "Do I know enough to act?"
+---------------------------+
           |
     < 95% confidence
           |
           v
+---------------------------+
| 3. EXPLORE (bottleneck)   |  <- grep, glob, semantic search, read files
|    - Which files matter?  |    THIS IS WHERE THE AGENT WASTES TIME
|    - What to search for?  |    scanning 20 files to find the 2 that matter
|    - What pattern is used?|    guessing keywords, hitting false positives
|    - Where is the entry   |    tracing call chains from scratch
|      point?               |
+---------------------------+
           |
           v
+---------------------------+
| 4. PLAN                   |  <- figure out what to change, where, how
+---------------------------+
           |
           v
+---------------------------+
| 5. IMPLEMENT              |  <- edit files
+---------------------------+
           |
           v
+---------------------------+
| 6. VERIFY                 |  <- lint, test, check
+---------------------------+
```

**Step 3 (Explore) is the bottleneck.** The agent has tools (Grep, Glob, SemanticSearch, Read) but doesn't know which files are relevant, which keywords to grep, what patterns are used, or where the entry point is. It searches broadly, hits false positives, traces from the top, and reads 1,000 lines to find the 50 that matter.

**The wiki must serve Step 3 directly.** Every wiki article should answer the questions the agent asks during exploration:
- *Which files own this concept?* → file pointers
- *What should I grep for?* → search keywords, anti-patterns (*"don't search for X, use Y"*)
- *What pattern does this code use?* → pattern name + key class/function
- *Where do I start reading?* → entry point function name
- *Which direction do I trace?* → *"start at A, follow call to B, decision happens in C"*
- *What should I NOT do?* → dead ends, pitfalls, common wrong assumptions

**Steps 1, 5, 6 don't need wiki help** — receiving a task, editing files, and running linters are mechanical.

> **Decision locked:** Wiki articles are structured to serve the agent's exploration phase. Every hint must answer one of the exploration questions above. If a hint doesn't help the agent explore faster, it doesn't belong in the wiki.

---

### JS-6 — Enrich existing articles from real work

**When** the agent is working on a task in an area that already has a wiki article, and it discovers something the article doesn't mention — a useful keyword to grep for, a direction to take, a pitfall to avoid, a related file it didn't expect,
**The agent** updates the existing article with the new hints,
**So** the wiki gets richer over time from real work, not just from the initial bootstrap.

> **Decision locked:** Enrichment follows the same threshold as creation — only update if the new info would save significant effort next time. Don't add noise.

---

### JS-7 — Store search shortcuts: keywords, grep patterns, directions

**When** the agent spends significant time figuring out which keywords to grep, which file patterns to glob, or which direction to trace through the code,
**The agent** saves those search shortcuts in the wiki article — the exact keywords, patterns, and directions that worked,
**So** a future agent doesn't waste time guessing what to search for — it knows the right `rg` pattern, the right directory to start in, the right class name to follow.

> **Decision locked:** Search shortcuts are a first-class hint type. They're often the most valuable part of a wiki article because they save the most time.

---

### JS-8 — Stay invisible to the human

**When** the human is working normally — asking questions, requesting code changes, reviewing PRs,
**The agent** uses and maintains the wiki entirely in the background, without mentioning it, asking permission, or changing the human's workflow,
**So** the human never knows the wiki exists unless they look at `wiki/`. The wiki is infrastructure, not a feature the human interacts with.

> **Decision locked:** The agent never says *"I'm checking the wiki"* or *"I'm updating the wiki."* It just works faster. The only visible effect is that the agent gives better answers and needs less exploration time.

---

### JS-9 — Merge articles when the wiki reaches capacity

**When** the wiki approaches or reaches 200 articles and the agent needs to create a new one,
**The agent** reviews existing articles, identifies two or more that cover closely related topics, merges them into a single broader article, and then creates the new article within the freed slot,
**So** the wiki stays within the 200-article budget without losing knowledge — information is consolidated, not deleted.

> **Decision locked:** The agent never deletes knowledge to make room. It merges: combines related articles into a broader one. The merged article keeps all hints from both, deduplicated and compressed. Target: ~400 words; if a merge pushes past 400 words, pick a different merge pair.

---

## 5. Assumptions

| # | Assumption | Confidence | Risk if Wrong |
|---|------------|------------|---------------|
| A-1 | The schema is actively invoked at every session start. Cursor: `.cursor/skills/wiki-memory/SKILL.md` (skill, not rule — `alwaysApply: true` rules proved unreliable for behavioral workflows). Claude Code: `CLAUDE.md`. Codex: `AGENTS.md`. | High (skill) / Medium (rule) | Wiki is invisible to the agent → no behavior change at all |
| A-2 | Source code files and merged PRs are the source of truth — research docs are ideas only | High | Wiki built from stale proposals instead of actual code |
| A-3 | The agent can read merged PRs to bootstrap topics (via `git log`, `gh pr list`, or a GitHub MCP) | High | Bootstrap falls back to scanning source directories by size/recency |
| A-4 | The agent can autonomously decide when a wiki article is worth writing (effort threshold) | Medium | Agent writes too many low-value articles (noise) or too few (wiki stays empty) |
| A-5 | The agent can self-correct wiki hints accurately when it detects staleness | Medium | Only fires reliably for catastrophic staleness (file-not-found). Subtle name mismatches are silently resolved without correction. Accepted tradeoff — minor drift is harmless. |
| A-6 | `index.md` is sufficient for navigation at 0–200 articles | High | Fallback: scanning source code (no worse than today) |
| A-7 | Wiki work adds minimal overhead; net effect is the agent is faster | High | Occasional wiki writes after a task; the rest of the time the wiki makes the agent faster |
| A-8 | Max 200 articles, ~400 words each is the right size | High | Comfortable headroom; if outgrown, add `qmd` or equivalent search |
| A-9 | Wiki hints survive trivial source changes through multi-layered anchoring | High | Pattern/structural anchors hold even if many functions rename at once |

---

## 6. Scope Definition

### In Scope
- Create a **schema file** — actively invoked instruction governing autonomous wiki behavior (learn/use/self-correct/merge/closing-step). For Cursor: `.cursor/skills/wiki-memory/SKILL.md` (recommended). For Claude Code: a section in `CLAUDE.md`. For Codex: a section in `AGENTS.md`.
- Create `wiki/TEMPLATE.md` — article template with structure, section purposes, and worked example
- Create `wiki/index.md` — empty catalog with domain categories appropriate to this codebase
- Create `wiki/log.md` — initialized with first entry
- Wiki limits: max 200 articles, ~400 words each; merge related articles when at capacity
- **Bootstrap:** agent reads the last ~60 merged PRs to seed ~10–20 initial articles
- Multi-layered anchoring: function/class names, pattern names, structural truths, directory ownership, search keywords
- Self-correction: agent fixes stale hints during normal work — no human maintenance
- Article merging: when at 200 articles, agent consolidates related articles to free slots — no knowledge deleted
- Invisible to human: no wiki commands, no wiki mentions in responses

### Explicitly Out of Scope
- Knowledge graph / entity registry / confidence scoring — overkill at current scale
- `qmd` setup — optional upgrade when wiki exceeds ~200 articles
- Migrating existing research docs into wiki — research docs are ideas, not source of truth
- Any backend code changes — this is purely schema + configuration + infrastructure files
- Human-triggered wiki commands — the wiki is fully autonomous
- Consolidation tiers (working/episodic/semantic/procedural memory) — future upgrade if needed

---

## 7. Success Criteria

| # | Criteria | How to Verify |
|---|----------|---------------|
| D-1 | Schema exists and is actively invoked | Cursor: `.cursor/skills/wiki-memory/SKILL.md` exists and fires on trigger keywords. Claude Code: `## Wiki Protocol` section in `CLAUDE.md`. Codex: section in `AGENTS.md`. |
| D-2 | `wiki/index.md` exists with category scaffolding | Read the file; confirm domain categories appropriate to this codebase |
| D-3 | `wiki/log.md` exists with init entry | Read the file; confirm first entry |
| D-4 | Bootstrap produces initial articles from merged PRs | Agent reads last ~60 PRs, creates ~10–20 articles, each ≤ ~400 words |
| D-5 | Agent checks wiki before exploring source code | Ask a codebase question; observe agent reads `index.md` first |
| D-6 | Agent learns from real work | After a complex task in an uncovered area, verify a new article was created |
| D-7 | Agent skips wiki update when effort is marginal | After a simple task, verify no article was created — no noise |
| D-8 | Agent self-corrects *catastrophically* stale hints during normal work | Rename a referenced file (not just function); ask about that area; verify wiki update. Note: subtle name mismatches within the same file may not trigger correction — this is a known limitation. |
| D-9 | Articles use multi-layered anchoring | Confirm: function/class names + pattern names + structural truths + search keywords |
| D-10 | Agent enriches existing articles | After working on a covered area and finding new info, verify the article was updated |
| D-11 | Agent stays invisible to the human | Across 5 tasks, verify agent never mentions the wiki |
| D-12 | Agent merges articles when at capacity | At 200 articles, trigger new creation; verify merge happens without knowledge loss |

### Rollback Plan

All changes are additive: new files only (`wiki/`, schema rule). Rollback = delete `wiki/` and the schema rule. No existing files are modified.

---

## 8. Existing Code Analysis

This is a completely independent feature. No existing files are modified. All deliverables are new files.

| File | Type | Purpose |
|------|------|---------|
| Schema (`.cursor/skills/wiki-memory/SKILL.md` / section in `CLAUDE.md` / `AGENTS.md`) | New | Actively invoked instruction governing autonomous wiki behavior |
| `wiki/index.md` | New | Article catalog — the agent's table of contents |
| `wiki/log.md` | New | Append-only log of learn/correct operations |
| `wiki/TEMPLATE.md` | New | Article template — structure, section purposes, and worked example |

### Dependencies

- A way for the agent to read merged PRs:
  - `git log --merges --since="6 months ago"` + `git show --stat` (universal fallback — works in any repo)
  - `gh pr list --state merged --limit 60` (if GitHub CLI is available)
  - GitHub MCP server (if configured in Cursor/Claude Code)
- Optional future: `qmd` or similar local search when the wiki exceeds ~200 articles

---

## 9. Impact Assessment

**Impact:** LOW — completely independent feature, all additive.

No existing files are modified. No existing functionality is changed. The wiki is 4 new files that add a transparent memory layer. The agent behaves differently (checks wiki first, learns from work, self-corrects) — but this is the intended behavior, not a regression. Rollback = delete the 4 files.

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|------------|--------|------------|
| R-1 | Agent reads wiki on every task, increasing token cost | Low | Low | `index.md` is ~200 lines; a single wiki page is ~400 words. Total per-session overhead: ~1k tokens. Negligible vs. the 2,400 lines the agent would otherwise scan. |
| R-2 | Agent writes low-value wiki articles (noise) | Medium | Low | Effort threshold in schema: only write when savings are 50%+, not marginal. |
| R-3 | Agent self-corrects a wiki hint incorrectly | Low | Low | Hints are evidence-based — the agent reads source code before correcting. `log.md` tracks all changes for audit. |
| R-4 | Wiki maintenance delays the human's task | Low | Low | Schema instructs: learn/correct after the task, never blocking it. |
| R-5 | Bootstrap creates too many or too few initial articles | Medium | Low | Target ~10–20 articles from ~60 PRs. If too many: agent prunes low-value ones. If too few: agent learns organically from work (JS-2). |
| R-6 | Major refactor makes many wiki hints stale at once | Low | Medium | Multi-layered anchoring: pattern/structural anchors survive; only function-name anchors need updating. |

---

## 12. Lessons Learned (from production testing)

These findings come from 6 debugging attempts validating the wiki protocol in real Cursor agent sessions. They apply primarily to Cursor but the CLOSING STEP lesson is universal.

### L-1: `alwaysApply: true` rules are passive context, not behavioral triggers

`.cursor/rules/*.mdc` with `alwaysApply: true` injects text into the agent's context window, but the agent treats it as background information — it does not follow step-by-step behavioral instructions from rules. **Fix:** Use `.cursor/skills/*/SKILL.md` in Cursor. Skills are actively invoked when trigger words match (e.g., "find code", "explore", "wiki hint doesn't match"), making them reliable for behavioral protocols.

### L-2: LEARN requires explicit enforcement (CLOSING STEP gate)

Without enforcement, agents defer wiki LEARN to "after the task" but then stop at the turn boundary. The work never happens. **Fix:** Add a `## CLOSING STEP` section that forces a self-check before ending the turn: *"Did I read 3+ source files? → execute LEARN now."* This gate is mandatory — without it, LEARN silently fails.

### L-3: SELF-CORRECT only fires for catastrophic staleness

When a wiki hint says `function_foo()` but the actual function is `function_bar()` in the same file, the agent navigates by meaning (reads the file, finds the right function) and silently resolves the mismatch — it never marks it as stale. Only when the file itself doesn't exist (catastrophic staleness) does the agent detect and correct. **Fix:** Add a `DETECT` instruction in USE step 5 that tells the agent to explicitly mark `STALE_HINT` when names don't match. Even with this, testing showed subtle mismatches are still silently resolved. **Accepted tradeoff:** Minor drift is harmless (agent still gets correct answers); catastrophic staleness auto-corrects; periodic human audit handles the gap.

### L-4: Flat folder structure prevents category confusion

Original designs used subfolders (`wiki/concepts/`, `wiki/systems/`). Testing showed agents struggle with categorization decisions and sometimes create wrong paths. **Fix:** Flat `wiki/` folder — all articles go directly in `wiki/`, no subfolders. Categories exist only as headings in `index.md`.

---

## 13. Detail Plan — The 4 Deliverables

> **For the implementing agent:** These are drafts. Adapt file paths, category names, and examples to match the target codebase. The `Connector System` example below is intentionally concrete — keep it as a reference example in `TEMPLATE.md` even if your codebase has no connectors; it shows what a good article looks like.

### Deliverable 1: The Schema

**For Cursor (recommended)** → create `.cursor/skills/wiki-memory/SKILL.md`:
**For Cursor (alternative, weaker)** → create `.cursor/rules/wiki.mdc` with `alwaysApply: true` (passive context — agent may ignore behavioral workflows).
**For Claude Code** → append this to `CLAUDE.md` under a `## Wiki Protocol` heading.
**For Codex** → append this to `AGENTS.md` under a `## Wiki Protocol` heading.

> **Implementation lesson:** Testing proved that `alwaysApply: true` rules in Cursor provide passive context — the agent sees the text but ignores step-by-step behavioral workflows. Cursor skills (`.cursor/skills/*/SKILL.md`) are actively invoked when trigger words match, making them reliable for protocols like USE/LEARN/SELF-CORRECT. For Claude Code and Codex, `CLAUDE.md`/`AGENTS.md` are actively consumed at session start, so rules-in-file works. Key additions vs. the original draft: DETECT instruction (USE step 5), CLOSING STEP gate, flat folder constraint.

**Cursor Skill version (recommended — copy to `.cursor/skills/wiki-memory/SKILL.md`):**

````markdown
---
name: wiki-memory
description: >-
  Persistent agent memory stored in wiki/. USE: read wiki/index.md before exploring source code — match tokens to find relevant articles. LEARN: after significant exploration, write or enrich a wiki article. SELF-CORRECT: when a wiki hint is wrong, fix it in place. Use when starting any task that involves finding code, after completing a task that required deep exploration, or when a wiki hint doesn't match the actual source code.
---

# Wiki Memory

The wiki at `wiki/` is persistent agent memory. It stores hints and pointers — not answers.

## USE — before exploring source code

1. Read `wiki/index.md`
2. Scan the **tokens** line in each entry — match against the function/class/file/pattern you need
3. If a relevant article exists, read it — follow its Entry Points, Search Shortcuts, and Directions
4. Read only the source files the article points to
5. **DETECT — do this immediately when a hint doesn't match source:** if any function name, file path, or class name from the article is not found in the source file it points to, mentally mark it as `STALE_HINT` before continuing. Do not silently resolve it. This flag is what the CLOSING STEP checks.
6. If no article matches, explore source code normally

## LEARN — after significant exploration

Only write if future agent saves >50% effort:

1. Read `wiki/TEMPLATE.md` for structure
2. Draft article: 3 mandatory sections (Entry Points, Key Patterns, Search Shortcuts) + optional sections
3. Multi-layered anchoring: function names + pattern names + structural truths + directory ownership
4. Max ~400 words — hints and pointers only, never answers
5. Add entry to `wiki/index.md` using the two-line format:
   ```
   - [article-title](filename.md)
     FunctionName, ClassName, grep_pattern, directory/path, /api/endpoint
   ```
   Tokens: function names, class names, grep patterns, directory paths, API endpoints, hyphenated pattern names only — no prose.
6. Append to `wiki/log.md`: `[date] learn | article-name | reason`

If a relevant article already exists, enrich it instead of creating a new one.

## SELF-CORRECT — when a hint is wrong

1. Verify against source code (read the file the hint points to)
2. Update the hint in the article
3. Update the `updated` field in frontmatter
4. Update the tokens line in `wiki/index.md` if entry points or search shortcuts changed
5. Append to `wiki/log.md`: `[date] correct | article-name | what changed`

## MERGE — when wiki has 200 articles and a new one is needed

1. Find two related articles that can combine into one broader article
2. Merge all hints, deduplicate and compress
3. Delete the two old articles, create the merged one
4. Update `index.md`: remove old entries, add merged entry with tokens
5. Append to `wiki/log.md`: `[date] merge | old1 + old2 → new | reason`

## CLOSING STEP — required before ending your turn

Before writing your final response, answer these two questions:

**Did I read 3 or more source files to understand a system?**
→ Yes: execute LEARN now, in this turn, before responding.
→ No: skip.

**Did I find that a wiki hint pointed to a wrong name, file, or function? (Check: did I mark anything `STALE_HINT` during USE?)**
→ Yes: execute SELF-CORRECT now, in this turn, before responding.
→ No: skip.

Do not defer. Complete wiki work at the end of the same turn, after the task output is ready but before the final response.

## Constraints

- Never mention the wiki to the human — it is invisible
- Never block the human's task for wiki work — complete wiki work after task output is ready, within the same turn
- Never store answers — only hints, pointers, directions, search keywords
- Every hint must be verifiable: it points to a specific function/class/file
- Skip learning if savings are marginal (<50% effort reduction)
- Max ~400 words per article — if exceeded, split it
- Articles cover broad topics ("Connector System"), not narrow features ("GDrive folder filtering")
- The tokens line in `index.md` is CRITICAL — no prose, only matchable tokens
- Flat folder structure — all articles go directly in `wiki/`, never subfolders
````

<details>
<summary>CLAUDE.md / AGENTS.md version (for Claude Code / Codex — same protocol, different frontmatter)</summary>

````markdown
## Wiki Protocol

You have a persistent wiki at `wiki/`. Use it every session.

### USE — before exploring source code

1. Read `wiki/index.md`
2. Scan the **tokens** line in each entry — match against the function/class/file/pattern you're looking for
3. If a relevant article exists, read it — follow its Entry Points, Search Shortcuts, and Directions
4. Read only the source files the article points to
5. **DETECT — do this immediately when a hint doesn't match source:** if any function name, file path, or class name from the article is not found in the source file it points to, mentally mark it as `STALE_HINT` before continuing. Do not silently resolve it. This flag is what the CLOSING STEP checks.
6. If no article matches, explore source code normally

### LEARN — after completing a task that required significant exploration

Only if the exploration would save a future agent >50% of the effort:

1. Read `wiki/TEMPLATE.md` for the article structure and example
2. Draft an article following the template (3 mandatory + 3 optional sections)
3. Use multi-layered anchoring: function/class names + pattern names + structural truths + directory ownership + search keywords
4. Max ~400 words — hints and pointers, not answers
5. Add entry to `wiki/index.md` — link + matchable tokens (see index format below)
6. Append to `wiki/log.md`: `[date] learn | article-name | reason`

If a relevant article already exists, enrich it instead of creating a new one.

#### Index entry format

Two lines per entry — link, then matchable tokens:

    - [article-title](filename.md)
      FunctionName, ClassName, grep_pattern, directory/path, /api/endpoint

Line 2 is pure matchable tokens from the article's Entry Points and Search Shortcuts. No prose, no filler. Every token must be a function name, class name, grep pattern, directory path, API endpoint, or short hyphenated pattern name (e.g. `sync-pipeline`, `mcp-acl`, `eager-loading`). Pattern names help match "how does X work?" queries that don't use exact symbol names.

### SELF-CORRECT — when you notice a hint is wrong during normal work

1. Verify against source code (read the file the hint points to)
2. Update the hint to match current code
3. Update the `updated` field in frontmatter
4. Update the tokens line in `wiki/index.md` if entry points or search shortcuts changed
5. Append to `wiki/log.md`: `[date] correct | article-name | what changed`

### MERGE — when wiki has 200 articles and a new one is needed

1. Identify two related articles that can combine into one broader article
2. Merge all hints from both, deduplicate and compress
3. Delete the two old articles, create the merged one
4. Update `index.md` (remove old entries, add merged entry with tokens)
5. Append to `wiki/log.md`: `[date] merge | old1 + old2 → new | reason`

### CLOSING STEP — required before ending your turn

Before writing your final response, answer these two questions:

**Did I read 3 or more source files to understand a system?**
→ Yes: execute LEARN now, in this turn, before responding.
→ No: skip.

**Did I find that a wiki hint pointed to a wrong name, file, or function? (Check: did I mark anything `STALE_HINT` during USE?)**
→ Yes: execute SELF-CORRECT now, in this turn, before responding.
→ No: skip.

Do not defer. Complete wiki work at the end of the same turn, after the task output is ready but before the final response.

### Constraints

- Never mention the wiki to the human — it is invisible
- Never block the human's task for wiki work — complete wiki work after task output is ready, within the same turn
- Never store answers — only hints, pointers, directions, search keywords
- Every hint must be verifiable: it points to a specific function/class/file
- Skip learning if savings are marginal (<50% effort reduction)
- Max ~400 words per article — if it exceeds this, split it
- Articles cover broad topics ("Connector System"), not narrow features ("GDrive folder filtering")
- Use multi-layered anchoring: function/class names + pattern names + structural truths + directory ownership + search keywords
- The tokens line in index.md is CRITICAL — it's what makes articles findable. No prose, only matchable tokens.
- Flat folder structure — all articles go directly in `wiki/`. Never create subfolders.
````
</details>

### Deliverable 2: `wiki/index.md`

Starts empty. Categories are derived from the target codebase — **do not ship a default list**. The bootstrap agent must inspect top-level folders and domain to pick 5–8 categories that actually fit this codebase.

```markdown
# Wiki Index

<!-- Illustrative only — replace with 5-8 categories derived from the target
     codebase's top-level folders and domain. For a typical web app these
     might be: Core Domain, API & Routing, Data Layer, Background Jobs,
     Auth & Permissions, Frontend, Infrastructure. A CLI, library, data
     pipeline, or embedded codebase will need different categories. -->
```

After bootstrap, populated entries look like:

```markdown
## Data Layer
- [connector-system](connector-system.md)
  GoogleDriveConnector, FileConnector, build_connector_tree, folder_ids, file_retrieval, sync-pipeline, connectors/tasks.py, connectors/google_drive/, server/documents/connector.py
```

**Token budget at scale:** 200 articles × ~15 tokens per entry = ~3,000 tokens for the full index. Fits in a single read.

### Deliverable 3: `wiki/log.md`

```markdown
# Wiki Log

> Append-only audit trail. The agent writes to this file but never reads it.
> Purpose: human review of wiki evolution. Not an agent input.
>
> Merge-conflict risk: multiple branches appending here will conflict.
> Mitigation: keep entries on single lines; resolve conflicts by keeping all rows.

| Date | Op | Article | Detail |
|------|----|---------|--------|
| YYYY-MM-DD | init | — | Wiki initialized |
```

### Deliverable 4: `wiki/TEMPLATE.md`

The agent reads this file every time it creates or merges an article. Keep the `Connector System` example **as-is** — it's a validated worked example. Adapt only if the target codebase has domain-specific conventions worth showing.

```markdown
# Article Template

## Frontmatter

    ---
    sources: [path/to/file.py, path/to/other.py]
    related: [[other-article]]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    ---

## Sections

### Mandatory (every article):

## Entry Points [Required]
<!-- functions/classes to start reading from, include API endpoints -->

## Key Patterns [Required]
<!-- architecture, design patterns, conventions; inline DB models as "Models: ..." when relevant; include frontend locations -->

## Search Shortcuts [Required]
<!-- grep patterns, anti-patterns ("don't grep X"), directory ownership -->

### Optional (add when relevant):

## Directions [When flow spans 3+ files]
<!-- trace paths: "A → B → C"; include frontend-to-backend flow when it has a UI -->

## Tests [When tests exist]
<!-- test file paths, just paths — no descriptions needed -->

## Gotchas [When non-obvious things exist]
<!-- past bugs, pitfalls, and architecture "why" notes — anything non-obvious -->

## Rules

> Canonical rules live in the schema file (SKILL.md / CLAUDE.md / AGENTS.md) § Constraints. Do not duplicate them here.

## Example: Connector System

---
sources: [connectors/google_drive/connector.py, connectors/tasks.py, connectors/tree_utils.py]
related: [[search-pipeline]]
created: 2026-04-16
updated: 2026-04-16
---

# Connector System

## Entry Points
- `GoogleDriveConnector` in `connectors/google_drive/connector.py`
- `FileConnector` in `connectors/file/connector.py`
- API: `server/documents/connector.py` — connector tree and folder endpoints
- `connectors/tasks.py` — background sync tasks

## Key Patterns
- Sync pipeline: fetch → convert → index with folder_ids → Vespa
- Folder tree: `build_connector_tree()` in `tasks.py` + `tree_utils.py`
- GDrive: retry + 1-day checkpoint for large accounts
- Admin UI: `web/src/app/admin/connectors/`
- Frontend lib: `web/src/lib/connectors/`
- Models: `Document.folder_ids` (db/models.py), `folder_ids` (danswer_chunk.sd), `SearchFilters.folder_ids` (search/models.py)
- Configs: `vespa_constants.py`, `configs/app_configs.py`

## Search Shortcuts
- Grep: `GoogleDriveConnector`, `build_connector_tree`, `folder_ids`, `file_retrieval`
- Don't grep: `google`, `connector` (too broad)
- Dir: `backend/connectors/google_drive/`

## Directions
- Sync: `connector.py` → `file_retrieval.py` → `doc_conversion.py` → `indexing_pipeline.py`
- Tree: `tasks.py` → `tree_utils.py` → `Document.folder_ids`
- Search: query → `preprocessing.py` → Vespa filter

## Tests
- `tests/unit/test_google_drive_search_tool.py`, `tests/unit/test_folder_passthrough.py`
- `cypress/e2e/project_file_cross_tool.cy.js`

## Gotchas
- GDrive rate limits: retry in `file_retrieval.py`, checkpoint at 1-day intervals (PR #1183)
- Raw content not stored: check `store_raw_file_content` in `doc_conversion.py` (PR #1167)
- `folder_ids` on docs (not just tree) because Vespa can't join at query time
```

---

## 14. Implementation Steps — For the Agent Reading This

When the human says *"implement this"* or *"set up the LLM-wiki"*, do the following in order:

1. **Detect the environment.** Check which agent harness is in use:
   - Cursor → create `.cursor/skills/wiki-memory/SKILL.md` (the Cursor Skill version from Deliverable 1 — recommended). If skills are not available, fall back to `.cursor/rules/wiki.mdc` with `alwaysApply: true`, but note this is weaker.
   - Claude Code → append the Wiki Protocol section to `CLAUDE.md`
   - Codex → append the Wiki Protocol section to `AGENTS.md`
   - Unknown → default to a `CLAUDE.md` section and tell the human
2. **Create the wiki skeleton:**
   - `wiki/index.md` — category headings appropriate to this codebase (inspect top-level folders to pick reasonable categories)
   - `wiki/log.md` — with the initial `init` row
   - `wiki/TEMPLATE.md` — copy from Deliverable 4 verbatim (keep the Connector System example)
3. **Run JS-1 Bootstrap:**
   - Read the last **60 merged PRs** (use `gh pr list --state merged --limit 60 --json number,title,files` if available; otherwise `git log --merges -n 60 --pretty=format:'%h %s'` + `git show --stat <sha>`)
   - Cluster PRs by the files they touch to identify active areas
   - Pick the ~10–20 clusters with the highest PR volume or code complexity
   - For each cluster, read the referenced source files and draft a wiki article following the template
   - Each article ≤ ~400 words, multi-layered anchoring, flat folder structure (no subfolders in `wiki/`)
   - Update `wiki/index.md` with the link + tokens line for each article
   - Append a `learn` entry to `wiki/log.md` per article
4. **Report back to the human:** brief summary — N articles created, which areas they cover, total wiki size.
5. **Stop.** Do not continue writing articles beyond the bootstrap batch. Further articles come from real work (JS-2).

### Constraints while implementing

- **Do not modify source code files** — only create files under `wiki/` and the schema file.
- **Do not write articles for features not represented in the last 60 PRs** — bootstrap reflects actual work, not imagined completeness.
- **Do not exceed ~400 words per article** — if you find yourself writing a manual, stop and compress to hints.
- **Keep the flat folder structure** — all articles go directly in `wiki/`, no subfolders.
- **After bootstrap, stay invisible** — the human's next session should see a faster, more targeted agent, not a wiki announcement.
