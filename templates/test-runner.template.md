---
name: <project>-test-runner
description: Run the right test suite for what changed in <project> and report results. Use when asked to run tests or after changes that need verification.
---

# Test Runner — <PROJECT>

<One line: how many test runners exist and the golden rule — run the narrowest scope first.>

## Runners

| Stack | Runner | Where | Command |
|---|---|---|---|
| <stack> | <runner> | `<path>` | `<exact command incl. any non-default settings/env>` |
| ... | ... | ... | ... |

> Note any COUNTER-INTUITIVE requirement here (e.g. "the default test settings are broken — use
> `--settings=…`"; "the local venv is stale — use a container"; "must pass `DUMMY_SECRET=x`").

## Procedure
1. Scope from the diff: `git diff --name-only <main>...HEAD`.
2. Run the narrowest matching suite first; widen only if needed.
3. Confirm the change has a machine-checkable pass/fail — never report "done" on a "looks right".

## Common failure patterns
| Symptom | Cause | Fix |
|---|---|---|
| <error> | <cause> | <fix> |

## Output
```
Tests: <stack> — N passed, M failed
Failures: - <test>: <one-line cause>
Verdict: pass / fail / partial
```
