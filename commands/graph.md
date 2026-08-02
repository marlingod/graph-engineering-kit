---
description: Run the graph-engineering blueprint (goal → split → parallel workers → verify → merge → report) on a task.
argument-hint: <research question / audit / design decision>
---

Run the **graph-research** blueprint on: **$ARGUMENTS**

Follow the `graph-research` skill:

1. **GOAL** — restate the objective in one line and the acceptance bar (what "done/good" means).
2. **Loop-or-graph check** — if a single well-scoped agent + a verifier can do this, say so and do
   that instead (don't over-engineer). Only build a graph if the work splits into real specialties
   with fan-out.
3. **SPLIT** — list the worker nodes (each a distinct angle: research / compare / check / find-gaps).
4. **FAN OUT** — spawn the workers as parallel subagents in a single message (`Explore`/`researcher`/
   `Plan`), each with an isolated, specific brief. For a deterministic/large graph, use the `Workflow`
   tool instead (nodes = `agent()`, fan-out = `parallel()`/`pipeline()`).
5. **VERIFY** — pass the merged findings through a fresh-context reviewer (the `code-reviewer` /
   `security-reviewer` subagents, or a skeptical verifier agent) that flags mistakes/hallucinations.
6. **MERGE + SYNTHESIZE** — keep only what survived verification; produce ONE clear, actionable report
   with sources / `file:line` citations.

Respect the spend cap: a graph is many loops (~15× tokens) — reserve it for high-value work.
