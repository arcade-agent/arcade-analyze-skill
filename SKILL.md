---
name: arcade-analyze
description: >-
  Recover and visualize the architecture of a software codebase using
  arcade-agent (the Python ARCADE successor). Use this whenever the user wants
  to understand, recover, map, diagram, or audit the architecture of a project,
  detect architectural smells (dependency cycles, concern overload, scattered
  functionality), compute architecture quality metrics (RCI, TurboMQ,
  connectivity), or get an interactive visual report of how a codebase's
  components fit together. Triggers on requests like "analyze the architecture
  of X", "recover the architecture", "what does this codebase look like
  structurally", "find architectural smells / cycles", "visualize the
  components", "is this code well-modularized", or pointing at a Java / Kotlin /
  Python / C / C++ / TypeScript / JavaScript / Go repo and asking how it's organized.
  Works on a local directory or a git URL.
---

# arcade-analyze

Recover and explore the architecture of a codebase using arcade-agent — a Python
successor to USC's ARCADE workbench (Architecture Recovery, Change, And Decay
Evaluator). It parses source with tree-sitter, recovers a component-level
architecture via clustering, detects architectural smells, computes quality
metrics, and renders an interactive HTML report.

This skill exposes twelve workflows, each a bundled script in `scripts/`. The four
core ones:

| Workflow | Script | Use for |
|----------|--------|---------|
| **Analyze** | `analyze.py` | One codebase → interactive HTML report (components, smells, metrics) |
| **Compare algorithms** | `compare_algorithms.py` | Same codebase under several recovery algorithms, side-by-side |
| **Diff versions** | `diff_versions.py` | Architectural drift between two git refs (added/removed components, metric deltas, new smells) |
| **Query** | `query.py` | Answer questions about the architecture (summarize, explain a component, find relevant code, structured queries) |

## When to use

Use this skill any time the goal is understanding or auditing a codebase at the
**architecture / component level** — not line-level edits. Good fits: "map out
how this project is structured", "recover the architecture", "find dependency
cycles / smells", "how modular is this", "give me a diagram of the components",
"compare PKG vs ACDC recovery", "what changed architecturally since v1.0?",
"which component has the highest fan-in?", "explain the Clustering component".

Not for: editing code, running its test suite, or single-file questions — those
don't need architecture recovery.

## How to run the scripts

The scripts need `import arcade_agent` to work. Two ways to get there — pick
whichever the machine already has, checking in this order:

**A. `$ARCADE_AGENT_HOME` is set (development checkout).** Run with that
checkout's virtualenv interpreter, which has the pipeline deps (tree-sitter,
networkx, scipy, numpy, jinja2):

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/<script>.py" <args...>
```

The variable does double duty: the shell uses it to find the venv interpreter,
and the script itself reads it (after `--arcade-home`) to locate the checkout,
putting `<home>/src` on `sys.path` itself so a stale editable-install `.pth`
can't break it. A configured checkout always wins over a pip install.

**B. No checkout configured — use the PyPI package.** Install once with any
Python >= 3.12 interpreter, then run the scripts with that interpreter, no
environment variable needed:

```bash
python3 -m pip install arcade-agent   # once; needs Python >= 3.12
python3 "<skill-dir>/scripts/<script>.py" <args...>
```

If neither is available the scripts exit with this exact guidance; on a machine
with plain `python3` >= 3.12, `pip install arcade-agent` is the fastest path to
a working run.

`<skill-dir>` is the directory containing this SKILL.md.

Every script except the MCP server (`guard_mcp.py`, which speaks MCP on stdio)
prints a `===ARCADE_SUMMARY_JSON===` … `===END_ARCADE_SUMMARY_JSON===` block to
stdout. Parse that block for the structured result and relay it in chat;
don't try to scrape the human-readable lines.

`<source>` is a local directory **or a git URL** — arcade-agent clones the URL
for you (item 1d). Use this for analyzing repos you don't have locally.

---

## Workflow 1 — Analyze (`analyze.py`)

The default workflow: one codebase → interactive HTML report, auto-opened.

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/analyze.py" \
  /path/to/the/codebase --language java --algorithm pkg
```

### Key options

- `--language / -l` — `java`, `kotlin`, `python`, `c`, `cpp`, `typescript`, `go`,
  or `multi` (parse every detected language and relink cross-language edges).
  Auto-detected if omitted, but pass it when you know it to avoid mis-detection
  on polyglot repos.
- `--algorithm / -a` — recovery algorithm. Default `pkg` (package-based, fast,
  no LLM). Others: `wca`, `acdc`, `arc`, `limbo`. See `references/algorithms.md`.
- `--num-clusters / -n` — target component count for `wca`/`acdc`/`arc`/`limbo`.
- `--use-llm` — semantic concern + smell detection via the `claude` CLI. Richer
  but slower; only use when the user wants deeper "what concern does each
  component own" analysis. Set `ARCADE_MOCK=1` to dry-run without LLM calls.
- `--output / -o` — HTML output path. Defaults to
  `./arcade-report/<name>-<algorithm>.html` in the current directory.
- `--also-mermaid` — also write a `.md` Mermaid component diagram next to the
  HTML (useful for pasting a diagram into chat, a README, or a paper).
- `--no-open` — skip auto-opening (use in headless contexts or when batching).

Run with `--help` to see them all. Default to `pkg` — fast, deterministic, no
LLM. The report opens automatically unless `--no-open`. After it runs, relay the
summary JSON in chat (component breakdown, most severe smells, what the metrics
imply about modularity) and link the report path.

---

## Workflow 2 — Compare algorithms (`compare_algorithms.py`)

Recover the same codebase under several algorithms and produce one side-by-side
HTML report. Use when the architect asks "which recovery algorithm fits this
project?" or wants to confirm a "well-modularized" claim across lenses — a
codebase that looks clean under `pkg` (package-based) but tangled under `wca`
(dependency-based) is telling you the package layout hides the real coupling.

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/compare_algorithms.py" \
  /path/to/codebase --language java --algorithms pkg,wca,acdc
```

- `--algorithms` — comma-separated (default `pkg,wca,acdc`, no LLM needed). Add
  `arc`/`limbo` only with `--use-llm`.
- `--num-clusters / -n` — **strongly recommended** when including `wca`: without a
  target count WCA over-fragments (one cluster per entity). Set it to roughly the
  `pkg` component count for a fair comparison.
- The summary JSON lists per-algorithm component count, smell count, RCI,
  BasicMQ, and TurboMQ. **Contrast on RCI and BasicMQ, not TurboMQ** — each
  algorithm recovers a different number of components, and TurboMQ is an
  unbounded sum that rises with component count, so ranking algorithms by it
  just rewards whichever one made the most clusters. Link the comparison report.

---

## Workflow 3 — Diff versions (`diff_versions.py`)

Quantify architectural drift between two git refs. The script clones the repo to
a temp dir and checks out both refs there, so the user's working tree is never
touched. Use for "what changed architecturally since v1.0?", tracking tech-debt
accrual, or preparing an architecture-review before/after.

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/diff_versions.py" \
  /path/to/LOCAL/git/repo --from v1.0.0 --to v1.2.0 --language java
```

- `<repo>` must be a **local git repository** (it needs the history to check out
  refs). Refs can be tags, branches, or commit SHAs. `--to` defaults to `HEAD`.
- Prints a markdown drift report (A2A similarity, metric deltas, added/removed
  components, entity movements, possible splits/merges, new vs. resolved smells)
  and a summary JSON. Use `-o report.md` to also save the markdown.
- Reading the result: **A2A similarity** near 1.0 means little structural change;
  a low value (e.g. 0.38) signals a major refactor. A large "entity movements"
  count means classes were reorganized across components even if the component
  set looks similar.

---

## Workflow 4 — Query / Q&A (`query.py`)

Answer questions about a recovered architecture without regenerating a report.
This is the back-end for natural-language Q&A: map the architect's question to a
sub-command, run it, and relay the JSON.

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/query.py" <subcommand> /path/to/codebase [args]
```

| Sub-command | Args | Answers |
|-------------|------|---------|
| `summarize` | `[--focus PKG]` | "Give me an overview" / "what's in package X?" — packages, hotspots, entry points |
| `explain` | `<component>` | "Explain the Clustering component" — API surface, dependencies, cohesion |
| `find` | `"<text>"` | "Where is authentication handled?" — ranked relevant entities (architecture-aware) |
| `ask` | `<question>` | structured queries (below) |

`ask` questions: `component_of` (`--entity FQN`), `dependencies` /
`dependents` / `entities` (`--component NAME`), `most_coupled`, `summary`,
`largest`.

**Mapping natural language to a sub-command** (decide, then run one):
- "what does this codebase look like / overview / structure" → `summarize`
- "what's in / drill into package X" → `summarize --focus X`
- "explain / tell me about the X component", "what's X's API" → `explain X`
- "where is X handled", "find code about X", "what's relevant to X" → `find "X"`
- "which component is class Y in" → `ask component_of --entity Y`
- "what does component X depend on / what depends on X" → `ask dependencies` / `ask dependents --component X`
- "biggest components", "most coupled components", "highest fan-in" → `ask largest` / `ask most_coupled`

Parse results are cached by arcade-agent, so repeated questions about the same
codebase don't re-parse — it's cheap to run several sub-commands in a row.

---

## Architect workflows (5–12) — see `references/architect-workflows.md`

Eight more scripts producing architect-grade deliverables. Read that file before
using any of them — it carries the options, the CI/rules setup, and the
`visualizer.py` live-feedback loop. Route by what the user asks for:

| Ask sounds like | Script |
|---|---|
| "executive summary / health score / present to leadership" | `summary_report.py` |
| "dependency matrix / too many components to diagram / show the tangle" | `dsm.py` |
| "C4 / PlantUML / Structurizr / put it in our docs" | `export_c4.py` |
| "what should I fix / prioritize the debt / refactoring plan" | `refactor_plan.py` |
| "enforce rules / no cycles / fail CI if / conformance / is it layered" | `validate.py` (exits 1 on violations) |
| "whole system / these microservices / multiple repos" | `analyze_system.py` |
| "explore / interactive / clickable / drill into components" | `interactive_report.py` |
| "visualizer / dashboard / app-like / simulate a flow / demo I can present" | `visualizer.py` (`--serve` for the live agent loop) |

**Language support:** Java, **Kotlin**, Python, C/C++, **TypeScript/JavaScript**
(`.ts/.tsx/.js/.jsx`), and **Go** are supported (`--language kotlin` /
`typescript` / `go`). Kotlin, TypeScript, Go, and C/C++ need their tree-sitter
grammars in the arcade-agent venv (declared in arcade-agent's optional
`languages` extra). Very large TS trees (~2k+ files) parse slowly; scope with
`--source-root` or point at a sub-package. If a language errors, check
`<arcade-home>/src/arcade_agent/parsers/`.

**Polyglot repos:** pass `--language multi` to parse every detected language in
one pass. Cross-language edges are only linked *within a language family* —
currently just the JVM family (`java` + `kotlin`), the one validated pair. Other
languages are unioned without inventing edges between them, so a Python + Java
repo yields both graphs side by side rather than a fabricated bridge. Use this
for mixed Java/Kotlin services, where a single-language parse would miss half
the dependencies.

---

## Architecture guardrail (arcade-guard) — see `references/guard.md`

For keeping an AI agent (or a human) aligned to an **intended** architecture
*while building*, rather than analyzing after the fact. The intended architecture
is an author-written `architecture.spec.json` (components by path glob, layers,
allowed/forbidden dependencies, decay budgets); conformance is **deterministic**.
Two surfaces over the same engine (`scripts/_spec.py`): the `scripts/guard.py`
CLI (`init`, `check`, `propose`, `preview`, `explain`, `remediate`) and the
`scripts/guard_mcp.py` MCP server.

Reach for it on "enforce architecture", "guardrail for the agent", "keep the AI
from breaking the architecture", "stop architectural drift as we build", or
setting up an architecture gate. The habit to teach an agent: **propose → build →
preview before new deps → check, fix any ERROR**. Enforcement is tiered —
advisory in-loop, blocking at commit/CI (see `assets/guard-*`). Read
`references/guard.md` for the commands; full design + status in `GUARDRAIL_PLAN.md`.

---

## Interpreting the output for the user

- **Components** — the recovered modules and how many entities each holds. Very
  large components relative to the rest are a modularity warning.
- **Smells** — flag `high`-severity ones first. Common types: dependency cycles
  (BDC), concern overload (BCO), scattered functionality (SPF), link overload
  (BUO). Explain the concrete impact, not just the label.
- **Metrics** — `RCI` and `BasicMQ` are normalized to `[0, 1]`; near 1.0 means
  cohesive, well-separated components, while low values or high
  `InterConnectivity` suggest tangled boundaries. Treat these as signals, not
  verdicts.
- **`TurboMQ` is not on a 0–1 scale.** It is the Bunch-style *sum* of per-component
  cluster factors (Mitchell & Mancoridis), so it grows with component count —
  a 5-component recovery scoring `3.37` is normal, not broken. Never read it as
  a percentage or compare it across codebases with different component counts.
  For a bounded number use `BasicMQ` (the normalized mean) or the `normalized`
  field inside TurboMQ's own details. To compare two recoveries by TurboMQ, check
  `num_components` matches first.

The HTML reports are self-contained except that they load Mermaid from a CDN to
render the diagram, so the diagram needs internet to draw; all other report
content works offline.
