---
name: graph-research
description: Run a multi-agent "graph" (goal → split → parallel workers → fresh-context verifier → merge → one report) for research, audits, and design decisions. Use for open-ended, decomposable, high-value work — not routine coding.
---

# Graph Research / Orchestration

**Graph engineering** = designing the graph your agents run in: which specialized **nodes** exist, which
**edges** route work between them, and what **shared state** travels along those edges. Nodes do the
work; edges carry the results. It's the layer above loop engineering: *in a loop you set the goal and
let one agent pick its route; in a graph you declare the valid paths and the checks along them.*

## The blueprint (the canonical graph)

```
1. GOAL      define what you want (one clear objective + an acceptance bar)
2. SPLIT     break the work into distinct pieces (only if they're real specialties)
3. WORKERS   fan out — parallel agents, different angles:
               research · compare · check · find-gaps    (isolated context each)
4. VERIFY    a fresh-context reviewer node with teeth — catches mistakes/hallucinations
5. MERGE     keep only what survived verification
6. SYNTHESIZE  one clear, actionable report
```
Shared state grows along the edges: `{goal} → {goal, findings} → {goal, findings, verdicts} → report`.
Edge types: straight (A→B), conditional (if pass→ship / if reject→loop back), fan-out (one→many),
fan-in (many→one).

## Decide FIRST: loop or graph?

Default to a **loop** (one well-scoped agent + a good verifier). Reach for a **graph** only when:

| | Loop | Graph |
|---|---|---|
| Task shape | one job, clear finish | splits into distinct specialties |
| Parallelism | sequential | need fan-out then join |
| Tools/models | same throughout | different per node |
| Control flow | agent free-roams safely | need explicit, auditable routing |
| Failure | bad step just retries | one node must fail without poisoning the rest |
| Verification | agent self-checks | a dedicated read-only reviewer node |

**Anti-pattern:** a 5-node graph to "summarize this PDF." Over-engineering — that's a loop.
**Win condition:** every node does work a loop couldn't, and you can explain the whole thing in one breath.

## How to run it (in Claude Code)

- **Deterministic graph → the `Workflow` tool.** `agent()` calls are nodes; `parallel()`/`pipeline()`
  are fan-out edges; the JS control flow is the edges; values passed between stages are the shared
  state. Canonical shape: fan out finders → adversarially verify each finding in parallel → synthesize.
  (Only launch a Workflow when the user has opted into multi-agent orchestration.)
- **Ad-hoc fan-out → the `Agent` tool.** Spawn `Explore`/`researcher`/`Plan` subagents in ONE message
  (parallel), each with an isolated focus; then you synthesize their returns.
- **The VERIFY node → the `code-reviewer` / `security-reviewer` subagents** (this plugin) — fresh
  context, read-only, briefed to flag only real defects (not nitpicks).

## Rules of thumb
- **Keep it a loop if you can.** Name a node only if it's a genuine specialty or a reviewer.
- **Draw the edges before you code** — write the node list + routing first.
- **Design the shared-state object explicitly** — decide what travels and who may write it.
- **Give the reviewer node teeth** — separate, read-only, adversarial ("try to refute this; default to
  rejected if unsure").
- **Isolate failure** — one node can fail/retry without corrupting shared state.
- **Set a spend cap + a hard bound.** A graph is many loops; multi-agent uses ~15× the tokens of one
  turn. Reserve it for high-value work — a weak verifier now burns tokens in parallel.

## Good uses
- **Research** — competitor/compliance/library/market research; codebase audits.
- **Design decisions** — Generate→Reflect→Rank→Evolve: produce N candidate approaches, verify each,
  rank, evolve the winner — instead of taking the first plausible answer.
- **Comprehensive audits** — sweep many files/services in parallel for an issue class, then a verifier
  confirms each finding before it's reported.
- **Not** routine coding — cross-file dependencies don't parallelize cleanly; use a loop + verifier.

## Output
A single synthesized report: the goal, what each worker found, what the verifier rejected, and the
merged, actionable conclusion — with sources / `file:line` citations.
