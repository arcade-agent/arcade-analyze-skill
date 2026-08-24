# Architect workflows (5–12)

Deliverable-producing scripts beyond the four core workflows in `SKILL.md`. Run
them the same way as every other script — see "How to run the scripts" there.

These produce architect-grade deliverables. All take `<source>` + `--language`
(and most `--algorithm`, `--source-root`); all print the summary JSON block.

| # | Script | Produces | Reach for it when |
|---|--------|----------|-------------------|
| 5 | `summary_report.py` | Executive markdown: health score (0–100 + grade), top findings in plain English, recommended actions | The architect needs something to present — a review, a status doc, a non-technical audience |
| 6 | `dsm.py` | Design Structure Matrix HTML (cyclic cells in red) | The system has many components (>~10) and a Mermaid diagram is unreadable |
| 7 | `export_c4.py` | C4-PlantUML `.puml` + Structurizr `.dsl` | They want the architecture in their documentation toolchain |
| 8 | `refactor_plan.py` | Ranked refactoring roadmap (severity × blast radius), quick wins vs big bets | "What should I actually do about these smells?" |
| 9 | `validate.py` | Rule conformance + layered-architecture check; **exits 1 on violations** | They have architectural rules to enforce, or want a CI gate |
| 10 | `analyze_system.py` | Multi-module/microservices view: per-module health + system dependency graph | The target is several modules/services, not one codebase |
| 11 | `interactive_report.py` | **Explorable** HTML report: click a component (diagram node or chip) → side panel drills into its entities, dependencies, cohesion, API surface, and smells; dependency chips are clickable to walk the graph | They want to *explore* the architecture, not read a static report — the richer alternative to `analyze.py` |
| 12 | `visualizer.py` | **App-style** SPA (offline, no CDN; dark/light toggle): sidebar views — pan/zoom component diagram with L1/L2 detail toggle and drill-down panel, weighted dependency list + DSM, smells presented as *failure-point cards* (severity, impact, mitigation, effort), a ranked **Architect Recommendations** roadmap (quick wins / planned / big bets), an **animated dependency-flow simulator** with per-hop waterfall (auto-derived traces + user-recorded ones), a Knowledge view with **balanced scores, principle signals, strengths/risks, and per-component quality** (cluster factor, intra-connectivity), and a feedback bar. With `--serve` it becomes a **live agent loop**: browser feedback is written to a JSON file on disk and the page auto-reloads whenever the model JSON changes | They want the full "architecture recovery workbench" experience — demos, walkthroughs, simulating how a change flows through components, or a live feedback loop with Claude |

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/summary_report.py"  <source> -l java -o summary.md
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/interactive_report.py" <source> -l java
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/visualizer.py"      <source> -l java
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/dsm.py"             <source> -l java
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/export_c4.py"       <source> -l java -o out/
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/refactor_plan.py"   <source> -l java -o plan.md
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/validate.py"        <source> -l java --rules .arcade-rules.json
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/analyze_system.py"  <moduleA> <moduleB> <moduleC> -l java
```

**Rules & CI:** `validate.py` reads `.arcade-rules.json` (sample in
`assets/arcade-rules.sample.json`; YAML works if PyYAML is installed). Rule types:
`forbidden-dependency`, `no-cycles`, `metric-gate`, `max-fan-in`,
`max-component-size`. For a PR gate, copy `assets/arch-gate.yml` to the target
repo, or use `diff_versions.py --min-similarity / --max-new-smells` (both exit 1
when breached).

**Mapping requests to these workflows:**
- "executive summary / health / how healthy / present to leadership" → `summary_report.py`
- "dependency matrix / too many components to diagram / show the tangle" → `dsm.py`
- "C4 / PlantUML / Structurizr / put it in our docs" → `export_c4.py`
- "what should I fix / refactoring plan / prioritize the debt" → `refactor_plan.py`
- "enforce rules / no cycles allowed / fail CI if / conformance / is it layered" → `validate.py`
- "whole system / these microservices / multiple repos / cross-module deps" → `analyze_system.py`
- "explore / interactive / clickable / let me drill into components / explorable report" → `interactive_report.py`
- "visualizer / dashboard / app-like / simulate a flow / animate the architecture / failure points / demo I can present" → `visualizer.py`

`visualizer.py` extras: `--dump-model model.json` saves the computed model, and
`--from-model model.json` re-renders it without re-analyzing (works without
arcade-agent — that is how `examples/arcade-visualizer-demo.html` is built, from
`examples/visualizer-demo-model.json`). Custom simulation traces and feedback
notes are stored in the viewer's browser (localStorage); "Copy for Claude" turns
the collected feedback into a prompt to paste back into a Claude session.

**Live mode (`--serve`)** — the two-way loop with an agent. Instead of writing a
static file, serve the app on localhost:

```bash
"$ARCADE_AGENT_HOME/.venv/bin/python" "<skill-dir>/scripts/visualizer.py" \
  --from-model model.json --serve --port 8123          # or: <source> -l java --serve
```

Run it in the background. The startup output (and the emitted summary JSON)
names two files:

- **model JSON** — edit it (or regenerate it with `--dump-model`) and every open
  browser tab reloads within ~2s. This is how you apply user feedback: change
  the model, the view refreshes itself.
- **feedback JSON** (`<model>-feedback.json`) — every note the user types in the
  page's feedback bar is written here immediately. Poll or read this file to
  pick up their requests; each entry has `text`, `when`, and the `view` it was
  written from.

`--feedback-out` overrides the feedback path; `--no-open` skips the browser.
So the loop is: user annotates in the browser → you read the feedback JSON →
you edit the model JSON → their page auto-reloads.
