# LLM-Wiki for Code Repositories

> A **memory layer for coding repos** — nothing else.
>
> Your AI coding agent forgets your codebase between sessions. Every task starts from zero: same files re-read, same flows re-traced, same patterns re-derived. This gives it persistent memory — a compact wiki (≤200 articles) that the agent builds, uses, and self-corrects **autonomously**. No human maintenance. No commands to learn.

Based on Karpathy's [LLM-Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), specialized for a **single code repository** (not notes, not research, not team wikis — just your repo).

---

## How it works

```
+---------------------+         +--------------------+         +---------------------+
| Your source code    |  read   | wiki/ (hints only) |  read   | Your coding agent   |
| (.py, .ts, .go ...) | <-----  | index.md           | <-----  | reads index first,  |
| Your merged PRs     |  write  | feature-a.md       |  uses   | then targeted files |
|                     | <-----  | feature-b.md       |  fixes  | instead of blind    |
| (NEVER modified)    |  fix    | log.md             | <-----  | scanning            |
+---------------------+         +--------------------+         +---------------------+
                                        ^
                                        |
                              +---------------------+
                              | Schema rule         |
                              | (wiki.mdc /         |
                              |  CLAUDE.md section /|
                              |  AGENTS.md section) |
                              | ALWAYS-APPLIED      |
                              +---------------------+
```

- **Source code + merged PRs** are the source of truth. The wiki **never** replaces them.
- **The wiki** stores *hints* (file pointers, function names, grep patterns, gotchas) — not answers.
- **The schema rule** fires every session and tells the agent: check the wiki first, learn from real work, self-correct stale hints, stay invisible.
- **No human reads the wiki.** It exists solely for the agent.

## Why

Before:

> *"How does feature X work?"* → agent reads 3 files, ~2,400 lines, scans everything, derives the answer from scratch.

After:

> *"How does feature X work?"* → agent reads `index.md` (20 lines) → finds `feature-x.md` (30 lines of hints) → reads only the 50 lines that matter. **~24x less.**

The wiki doesn't answer the question. It tells the agent *where the answer lives*.

---

## Usage — 3 steps

### 1. Copy the file into your repo

```bash
curl -o LLM_WIKI_FOR_CODE.md \
  https://raw.githubusercontent.com/YOUR_USERNAME/llm-wiki-for-code/main/LLM_WIKI_FOR_CODE.md
```

Or just download and drop [`LLM_WIKI_FOR_CODE.md`](./LLM_WIKI_FOR_CODE.md) at the root of your project.

### 2. Ask your coding agent

Open Cursor / Claude Code / Codex / OpenCode in your repo and paste:

> *"Read `LLM_WIKI_FOR_CODE.md` and implement it. Detect my agent environment, create the schema rule, scaffold the wiki, and run JS-1 Bootstrap."*

### 3. Let it bootstrap (JS-1)

The agent will:

1. **Detect your environment** → create the right schema file:
   - Cursor → `.cursor/rules/wiki.mdc` (`alwaysApply: true`)
   - Claude Code → section in `CLAUDE.md`
   - Codex → section in `AGENTS.md`
2. **Scaffold the wiki** → `wiki/index.md`, `wiki/log.md`, `wiki/TEMPLATE.md`
3. **Read your codebase** to understand top-level structure and pick reasonable category headings.
4. **Read the last ~60 merged PRs** (`gh pr list --state merged --limit 60 ...` or `git log --merges -n 60 ...`) to identify what your team actually works on.
5. **Seed ~10–20 wiki articles** — one per active area — each ≤ ~400 words, anchored to real function/class/file names.
6. **Stop.** Further articles grow organically from your next coding sessions.

That's it. Every subsequent session, your agent reads the wiki first and reads ~100 targeted lines instead of thousands.

---

## What you get

After bootstrap, your repo will have:

```
your-repo/
├── .cursor/rules/wiki.mdc    # (Cursor) always-applied schema rule
│   OR
├── CLAUDE.md                  # (Claude Code) schema section appended
│   OR
├── AGENTS.md                  # (Codex) schema section appended
│
└── wiki/
    ├── index.md               # categorized catalog: link + matchable tokens
    ├── log.md                 # append-only audit trail (agent writes, never reads)
    ├── TEMPLATE.md            # article template + worked example
    ├── connector-system.md    # bootstrapped articles
    ├── auth-flow.md
    ├── search-pipeline.md
    └── ...                    # ~10-20 articles from real PR activity
```

No subfolders. Flat. ≤200 articles total, ever. If the budget fills, the agent **merges** related articles — never deletes knowledge.

## What it is NOT

- Not a general knowledge base (use Karpathy's original for that)
- Not a documentation system for humans
- Not RAG (no embeddings, no vector DB)
- Not a replacement for your source code or PRs — those remain the source of truth
- Not something you edit by hand — the agent owns `wiki/` entirely
- Not Obsidian / Notion / Confluence — it's plain markdown files the agent reads before exploring

## Four operations (all autonomous)

| Op | When | What |
|----|------|------|
| **USE** | Every task, before exploring | Read `index.md` → match tokens → read the relevant article → read only the source files it points to |
| **LEARN** | After a task that required significant exploration, *if* future savings > 50% | Extract hints into a new or existing article (≤ ~400 words) |
| **SELF-CORRECT** | Inline, when a hint is found stale | Read the code, fix the hint, update `index.md` tokens if needed, log it |
| **MERGE** | When the wiki hits 200 articles | Combine two related articles into one broader one; free a slot |

## Design principles

1. **Hints, not answers.** Every article is a developer's notebook page, not a manual.
2. **Verifiable.** Every hint points to a function/class/file the agent can open and check.
3. **Robust.** Anchor to function names, class names, patterns, and structural truths — never line numbers.
4. **Invisible.** The agent never announces what it's doing. You just notice it's faster.
5. **Budgeted.** Max 200 articles, ~400 words each. If it's longer, it's storing answers instead of hints.
6. **PR-driven, not speculation-driven.** Bootstrap and growth reflect what your team actually works on.

## When to NOT use this

- Solo scripts < 50 files → the agent already has enough context.
- Short-lived spikes / prototypes → maintenance overhead not worth it.
- Codebases where merged PRs don't reflect current reality (e.g., squash-and-rewrite cultures where the tree is frequently rewritten).
- When you want team-readable docs — this is written for the agent, not for humans.

## Compatibility

Works with any coding agent that can:
- Read files on disk
- Follow an always-applied instruction rule (`.cursor/rules/*.mdc`, `CLAUDE.md`, `AGENTS.md`, or similar)
- Run shell commands (to read PRs via `git` or `gh`)

Tested with: Cursor, Claude Code. Should work with: Codex, OpenCode, Aider, Cline, Windsurf, Continue.

## Credits

- Pattern: [Andrej Karpathy's LLM-Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- This specialization: distilled from a real production codebase where the pattern cut agent exploration time dramatically across many sessions.

## License

MIT — see [LICENSE](./LICENSE).
