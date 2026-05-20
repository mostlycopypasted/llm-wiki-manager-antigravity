# llm-wiki-manager

> A Claude Code skill for building and maintaining a personal LLM-managed wiki — a persistent, compounding knowledge base where the LLM does all the writing, cross-referencing, and bookkeeping while you curate sources and ask questions.

[![Release](https://img.shields.io/github/v/release/sametbrr/llm-wiki-manager?display_name=tag&sort=semver)](https://github.com/sametbrr/llm-wiki-manager/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/agentskills.io-compatible-blue)](https://agentskills.io)
[![Works with Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-blueviolet)](https://claude.ai/code)

```bash
# Install
git clone https://github.com/sametbrr/llm-wiki-manager ~/.claude/skills/llm-wiki-manager
```

---

## What is this?

Instead of RAG — where the LLM rediscovers answers from raw documents on every query — this pattern has the LLM **compile** raw sources into a persistent, interlinked markdown wiki. Each new source enriches existing pages. Cross-references are built eagerly. Contradictions are flagged. Knowledge compounds over time.

Implements [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) as a full Claude Code skill with 8 operating modes (including multi-wiki routing), 4 idempotent Python scripts, 8 page templates, and 9 reference documents.

```
Without this pattern          With this pattern
────────────────────          ──────────────────
Query 1 → re-read 50 docs     Query 1 → read compiled wiki (already synthesized)
Query 2 → re-read 50 docs     Query 2 → read updated wiki (cross-refs already there)
Query 3 → re-read 50 docs     Query 3 → read updated wiki (contradictions already flagged)
```

---

## Quick start

```bash
# 1. Go to your research folder
mkdir ~/research/my-topic && cd ~/research/my-topic && claude

# 2. In Claude Code — bootstrap the wiki
> "Set up an LLM wiki here. Topic: history of nutrition science."

# 3. Drop a source
cp ~/Downloads/pollan-2008.pdf raw/

# 4. Ingest it
> "Ingest Pollan's In Defense of Food"

# 5. Ask questions
> "What does the wiki say about nutritionism?"

# 6. Health check (auto-saves dated report to wiki/reports/)
> "Lint the wiki"
```

---

## The three-layer model

```
your-wiki/
├── CLAUDE.md          # Schema — conventions for this wiki (co-evolved over time)
├── raw/               # YOUR layer — immutable sources you curate. LLM reads, never writes.
└── wiki/              # LLM layer — all pages written and maintained by the LLM
    ├── index.md       # Content catalog (updated on every ingest)
    ├── log.md         # Append-only operation log (greppable)
    ├── hot.md         # Hot cache — most recently ingested sources and active references
    ├── sources/       # One summary page per ingested source
    ├── entities/      # People, organizations, places, products
    ├── concepts/      # Ideas, theories, frameworks, terms
    ├── notes/         # Filed-back query answers and loose pages
    └── reports/       # Auto-generated dated lint reports (wiki/reports/lint-YYYY-MM-DD.md)
```

**Division of labor:**

| You do | The LLM does |
|---|---|
| Curate sources (decide what to read) | Read sources end-to-end |
| Ask questions, steer direction | Write summaries, entity and concept pages |
| Review the wiki, follow links | Update cross-references in-place |
| Decide what matters | Maintain index.md and log.md |
| Own `raw/` | Flag contradictions, surface gaps |

You almost never write wiki pages by hand. The LLM does the bookkeeping — that's what makes the wiki compound instead of collapse.

---

## Modes

The skill auto-detects which mode applies from natural language. No slash commands needed.

| Mode | Trigger examples | What happens |
|---|---|---|
| **Bootstrap** | "Set up a wiki", "start a knowledge base here" | Scaffolds `raw/`, `wiki/`, `CLAUDE.md` from templates |
| **Ingest** | "Add this PDF to the wiki", "I just read X, file it" | Reads source → writes summary → updates entity/concept pages → indexes → logs |
| **Query** | "What does the wiki say about X?", "Compare X and Y" | Reads index → candidate pages → synthesizes answer with citations → offers to file back |
| **Update** | "Smith 2024 supersedes Keys 1980, update the wiki" | Semantic sweep across all pages → diff-before-write per page → single log entry |
| **Lint** | "Health check the wiki", "anything broken?" | Runs `lint_wiki.py` → auto-saves `wiki/reports/lint-YYYY-MM-DD.md` → auto-tracks in index and log |
| **Schema-evolve** | "We should always do X going forward" | Updates `CLAUDE.md` so future sessions inherit the convention |
| **Multi-wiki** | "Add this to my global wiki", "what does the global wiki say about X", "promote this page to global" | Routes between project wiki and global wiki using the `External Wiki:` declaration in project `CLAUDE.md` |
| **Teach** | "How does this pattern work?", "explain the LLM wiki idea" | Explains the pattern, compares with RAG, walks through a concrete example |

---

## Core disciplines (enforced in every mode)

1. **LLM owns `wiki/`. You own `raw/`.** No exceptions.
2. **Every operation logs to `log.md`** via `append_log.py`. Greppable: `grep "^## \[" log.md | tail -20`
3. **Every new or updated page touches `index.md`** via `update_index.py`. Stale index = wiki that feels lost.
4. **Cross-reference aggressively.** When a source mentions an entity that already has a page, update that page. Don't leave connections implicit.
5. **Cite back to `raw/`.** Every claim is traceable to a specific source file.
6. **Flag contradictions, don't overwrite.** New source disagrees with old claim? Both stay, marked with their source, with a `> [!warning] Sources disagree` callout.
7. **Schema lives in `CLAUDE.md`.** When a convention works, write it down. The next session starts informed.

---

## What's inside

### Scripts (Python stdlib, no dependencies, all idempotent)

| Script | Purpose |
|---|---|
| `scripts/init_wiki.py` | Scaffold a new wiki — creates `raw/`, `wiki/`, `CLAUDE.md`, `index.md`, `log.md`, and `hot.md`. Idempotent. |
| `scripts/append_log.py` | Append a `## [YYYY-MM-DD] action \| title` entry to `log.md`. Supports flexible log path detection. |
| `scripts/update_index.py` | Add or update an entry under a category in `index.md`. Upserts by (category, title). Flexible index path detection. |
| `scripts/lint_wiki.py` | Health check. Detects orphan pages and index drift in both standard markdown and Obsidian wiki-link (`[[...]]`) format. Default: writes `wiki/reports/lint-<today>.md` and auto-tracks. Run `--stdout` for terminal output. |

### Templates

| Template | Used for |
|---|---|
| `wiki-CLAUDE.md.tmpl` | The schema file dropped into a fresh wiki |
| `source-summary.md.tmpl` | One ingested source — claims, methodology, cross-links, open questions |
| `entity-page.md.tmpl` | People, organizations, places, products |
| `concept-page.md.tmpl` | Ideas, frameworks, theories, terms |
| `comparison-page.md.tmpl` | "X vs Y" pages, often filed-back query answers |
| `index.md.tmpl` | Initial content catalog |
| `log.md.tmpl` | Initial log with bootstrap entry |
| `hot.md.tmpl` | Initial hot cache — rewritten after each ingest with latest sources and active references |

### Reference docs

Nine detailed workflow documents in `references/`:
`philosophy.md` · `architecture.md` · `bootstrap-workflow.md` · `ingest-workflow.md` · `query-workflow.md` · `update-workflow.md` · `lint-workflow.md` · `schema-design-guide.md` · `multi-wiki-routing.md` · `teaching-mode.md`

The skill reads these selectively — you don't need to. They're there to give the LLM depth on each mode.

---

## Update mode — what makes this different

Standard ingest already handles contradictions on a single page (Disputes section). **Update mode** is for when a new source supersedes a claim that's paraphrased across multiple pages — the same idea written four different ways in four different files.

```
Scenario: Smith 2024 reanalysis shows Keys 1980's seven-countries study cherry-picked data.
The Keys r=0.87 claim appears as:
  concepts/saturated-fat.md     → "Keys found r=0.87 across seven countries"
  entities/ancel-keys.md        → "famous for showing strong correlation"
  concepts/heart-disease.md     → "saturated fat is a primary driver per Keys"
  concepts/dietary-policy.md    → "the saturated fat hypothesis drove decades of policy"

Update mode:
  1. Semantic sweep — finds all four (grep won't, LLM will)
  2. Shows scope: "4 pages affected. Proceed?"
  3. Diff-before-write per page — y/n/skip/edit per change
  4. Per-page strategy: revise / disputes / annotate (not one-size-fits-all)
  5. One log entry tying all four edits to Smith 2024
```

This is change **propagation**, not change tracking. The log and frontmatter track what was edited. Update mode ensures the edit reaches everywhere it should.

---

## Auto-dated lint reports

`lint_wiki.py` with no flags:
- Writes `wiki/reports/lint-YYYY-MM-DD.md` (overwrites same-day run — idempotent daily)
- Adds a `Reports` index entry automatically
- Appends a `lint | Health check` log entry automatically
- Exits with code 1 if any block-severity issue found (useful for CI)

Reports accumulate as a longitudinal health record. `git log wiki/reports/` shows how wiki quality evolved over time.

Override flags: `--stdout` (terminal, no tracking), `--no-track` (write file, skip index/log), `--report PATH` (custom path).

---

## Multi-wiki — project wiki + global wiki

Most users start with one wiki. Once you have **two** — say, a per-project wiki at the working directory plus a long-lived global "second brain" (often an existing Obsidian vault) — the skill routes writes between them based on a single declaration in the project's `CLAUDE.md`.

```
~/projects/x-project/          ← active project (current working directory)
├── CLAUDE.md                  ← project schema — declares the global wiki path
├── raw/                       ← project sources
└── wiki/                      ← project wiki

~/Documents/obsidian/          ← global wiki (long-lived, exists across projects)
├── CLAUDE.md                  ← global schema
├── raw/
└── wiki/
```

### One-time setup

Add this to the project's `CLAUDE.md` (or ask the agent to do it):

```markdown
## External Wiki

Global knowledge base: ~/Documents/obsidian/

### Routing rules
- Project-specific code decisions, architecture, bugs, configuration → this project's `wiki/`
- Concepts, frameworks, patterns, ideas applicable beyond this project → global wiki
- When in doubt, ask before writing
- Scripts always need `--path` flag pointing to the right wiki root

### Cross-wiki links
- Use absolute paths (`~/...`) when linking from one wiki to the other.
- Never use relative paths that cross wiki boundaries.
```

The agent reads this on every session. No further configuration needed. The same `## External Wiki` block is present (commented out) in `assets/templates/wiki-CLAUDE.md.tmpl` — uncomment it on bootstrap if the project should reference a global wiki.

### Four canonical scenarios

| # | Scenario | Trigger | What the agent does |
|---|---|---|---|
| **A** | **Write to global** while in a project | "Add JWT refresh rotation to my global wiki" | Reads project `CLAUDE.md` → resolves global path → writes to global, project wiki untouched |
| **B** | **Pull from global** into a project | "What does the global wiki say about rate limiting? Apply it to /api/search" | Reads global pages → synthesizes recommendation → writes a project-specific page that **links** to the global one (never copies) |
| **C** | **Promote** a page from project to global | "concepts/event-sourcing.md has matured, promote it" | Moves content to global → leaves a one-line redirect stub at the project path → updates both indexes and logs |
| **D** | **Lint both** wikis at once | "Lint both wikis" | Runs `lint_wiki.py --path` against each → reads both reports → returns one summary |

The agent always uses `--path <wiki-root>` when targeting scripts, never relies on `cwd`. Cross-wiki links are absolute (`~/...`) so they survive moves and git history rewrites. Project pages link **out** to global pages — they never copy global content, because copies rot when the global page evolves.

Full walkthroughs of all four scenarios — exact prompts, plans, scripts, resulting file structures — are in [`references/multi-wiki-routing.md`](references/multi-wiki-routing.md).

### When *not* to use multi-wiki

Don't split prematurely. A single project wiki with everything inside is not worse than a multi-wiki setup. Add a global wiki only when (1) you have 2+ active projects sharing concepts, (2) you keep re-explaining patterns across project wikis, or (3) you have a long-running personal knowledge base that predates the project.

---

## Install

**Option 1 — git clone (recommended)**
```bash
git clone https://github.com/sametbrr/llm-wiki-manager ~/.claude/skills/llm-wiki-manager
```

**Option 2 — GitHub CLI** (requires gh CLI v2.90+)
```bash
gh skill install sametbrr/llm-wiki-manager
```

**Option 3 — .skill file**
```bash
curl -L -o llm-wiki-manager.skill \
  https://github.com/sametbrr/llm-wiki-manager/releases/latest/download/llm-wiki-manager.skill
unzip llm-wiki-manager.skill -d ~/.claude/skills/llm-wiki-manager
```

After installing, start a new Claude Code session. The skill loads automatically when relevant.

**Works with:** Claude Code · OpenAI Codex · Cursor · Gemini CLI · Windsurf · any agent supporting the [agentskills.io](https://agentskills.io) standard.

---

## Compatibility

| Tool | Skills path | Notes |
|---|---|---|
| Claude Code | `~/.claude/skills/` or `.claude/skills/` | Global or project-level |
| OpenAI Codex | `~/.codex/skills/` | Same SKILL.md format |
| Cursor | `.cursor/skills/` | Project-level |
| Gemini CLI | `~/.gemini/skills/` | |

---

## Related

- [Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — the original idea
- [agentskills.io](https://agentskills.io) — open standard spec

---

## License

MIT — see [LICENSE](LICENSE).

Inspired by Andrej Karpathy's LLM Wiki idea. Built with the [skill-creator](https://github.com/anthropics/skills) framework.
