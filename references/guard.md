# Architecture guardrail (arcade-guard)

Full command reference for `scripts/guard.py` and `scripts/guard_mcp.py`. See
`SKILL.md` for when to reach for the guardrail at all.

For keeping an AI agent (or a human) aligned to an **intended** architecture
*while building* — not just analyzing after the fact. The intended architecture
is an author-written `architecture.spec.json` (components by path glob, layers,
allowed/forbidden dependencies, decay budgets). Conformance is **deterministic**.

Two surfaces, same engine (`scripts/_spec.py`):
- **CLI** — `scripts/guard.py <cmd> <source>`:
  - `init --template hexagonal|layered|clean|mvc` — scaffold a spec.
  - `check` — verdict PASS/WARN/FAIL + violations + fixes; `--fail-on error`
    exits ≠0 (pre-commit / CI gate).
  - `propose --intent "..."` — **proactive**: where should a new thing live + what
    may it depend on. Call *before* writing code.
  - `preview --from A --to B` — would that dependency be allowed?
  - `explain --violation <id>` / `remediate` — why + ranked fixes.
- **MCP server** — `scripts/guard_mcp.py` exposes `check_architecture`,
  `propose_placement`, `preview_impact`, `explain_violation`, `remediate` to any
  MCP agent. Run with the venv interpreter + `ARCADE_AGENT_HOME` set.

Use it when the user wants to "enforce architecture", "guardrail for the agent",
"keep the AI from breaking the architecture", "stop architectural drift as we
build", or set up an architecture gate. The habit to teach an agent: **propose →
build → preview before new deps → check, fix any ERROR**. Enforcement is tiered —
advisory in-loop, blocking at commit/CI (see `assets/guard-*`). Full design +
status: `GUARDRAIL_PLAN.md`.
