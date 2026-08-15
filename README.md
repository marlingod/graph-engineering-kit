# Graph Engineering Kit

A portable Claude Code **plugin** that drops the *graph-engineering* blueprint into any project:
**goal → split → fan-out parallel workers → fresh-context verifier → merge → one report.**
Nodes do the work; edges carry the results.

It's the **generic, project-agnostic** layer. Project-specific tooling (your deploy runbook, your
gotchas) stays in each repo's own `.claude/` — this kit ships the reusable patterns + templates to
create those quickly.

## What's inside

| Component | What it is |
|---|---|
| `skills/graph-research/` | The graph blueprint as a skill — the loop-vs-graph decision, the fan-out/verify/merge recipe, spend caps, and how to run it (Workflow tool / subagents). |
| `skills/diagnose-reachability/` | Debug "works from region A, times out from region B" — layered elimination (firewall/SG/DNS/cert/NACL/path-MTU) → MSS-clamp bridge fix → CDN-in-front durable fix, with the cutover footguns. |
| `agents/code-reviewer.md` | Fresh-context **verifier node**: reviews a diff for correctness/regression/requirement gaps (read-only, real-defects-only). |
| `agents/security-reviewer.md` | Fresh-context verifier for auth/injection/secrets/money/PII. |
| `commands/graph.md` | `/graph <task>` — run the blueprint on a research/audit/design task. |
| `commands/review.md` | `/review [range]` — gate the current diff through both reviewer subagents in parallel. |
| `templates/` | Fill-in-the-blank starting points for the **project-specific** skills: `CLAUDE.md.template`, `deploy-runbook.template.md`, `test-runner.template.md`. |
| `hooks/hooks.json.example` | Opt-in deterministic gates (format-on-edit, block-migration-writes, verification Stop-gate) to copy into a project's `.claude/settings.json`. Not active by default. |

## Install / dogfood

**Fastest (no marketplace) — load it for one session:**
```
claude --plugin-dir ~/Documents/projects/graph-engineering-kit
```

**Persistent — install from the bundled local marketplace:**
```
/plugin marketplace add ~/Documents/projects/graph-engineering-kit
/plugin install graph-engineering-kit@graph-engineering-marketplace
/reload-plugins
```
This repo is **both** the plugin and the marketplace (the plugin's `source` is `./`).

**After editing the kit:** `/reload-plugins` (or `/plugin marketplace update graph-engineering-marketplace`).
**Validate before sharing:** `claude plugin validate .`

Plugin commands/skills are namespaced: `/graph-engineering-kit:graph`, `@graph-engineering-kit:code-reviewer`.

## Dropping this into a new project

1. **Install the plugin** (above) — you now have `/graph`, `/review`, and the reviewer subagents everywhere.
2. **Create the project's own `.claude/`** for the specifics:
   - Copy `templates/CLAUDE.md.template` → `<repo>/CLAUDE.md` (or per-directory) and fill in the
     non-obvious gotchas + commands + a verification rule.
   - Copy `templates/test-runner.template.md` → `<repo>/.claude/skills/<project>-test-runner/SKILL.md`
     and fill in the real test commands.
   - Copy `templates/deploy-runbook.template.md` → `<repo>/.claude/skills/<project>-controlled-deploy/SKILL.md`
     and fill in the deploy steps + **the outage gotchas as you hit them** (this section is the payoff).
3. **Optionally** copy a hook from `hooks/hooks.json.example` into `<repo>/.claude/settings.json`.
4. Project `.claude/` overrides the plugin where names collide, and both sets of hooks merge and run.

## The one discipline
Keep it a **loop** unless the work genuinely splits into specialties + fan-out + a reviewer node.
A graph is many loops (~15× tokens) — reserve it for high-value research/audits/design. Give the
reviewer node teeth (read-only, adversarial). Never mark work done without a machine-checkable pass/fail.

## Security posture
Installing this plugin lets it run **agents and (opt-in) hooks inside your Claude Code session, across
every project it's installed in**. Treat it like any dependency with execution rights:
- **Keep write access to this repo tight** — it is its own marketplace (`source: "./"`), so anyone who
  can push here can change what runs in your sessions. Protect the default branch; review PRs.
- **Prefer pinned/tagged installs** over "always latest" for anything you didn't just author, and
  **review the diff before `/reload-plugins`** after pulling changes.
- **The reviewer subagents are scoped to read-only git/gh verbs** (not full `Bash`) by design — they
  ingest untrusted diffs/PRs, so keep it that way. Don't widen their `tools:` grant.
- **Hooks ship inert** — `plugin.json` does not reference `hooks/`. The examples are opt-in and must be
  adapted per project; verify any guard actually fires (a guard that fails open is worse than none).

## License
MIT
