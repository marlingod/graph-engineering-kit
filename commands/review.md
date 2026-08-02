---
description: Gate the current diff/PR through fresh-context code + security reviewer subagents before merge.
argument-hint: [optional PR # or branch/commit range; defaults to origin/<main>...HEAD]
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(gh pr view:*), Task
---

Run the **verifier node** on the current change: **$ARGUMENTS**

1. Determine the diff range — use `$ARGUMENTS` if given, else `git diff origin/<default-branch>...HEAD`
   (find the default branch with `git symbolic-ref refs/remotes/origin/HEAD`).
2. Launch BOTH reviewer subagents **in parallel** (single message) on that range:
   - the `code-reviewer` subagent (correctness / regression / requirement gaps),
   - the `security-reviewer` subagent (auth / injection / secrets / money / PII).
3. Each runs in fresh context, read-only, and flags **real defects only** (not style/nitpicks).
4. **Merge** their verdicts into one report: Blockers → Concerns → overall verdict
   (approve / approve-with-followups / request-changes). If both come back clean, say so plainly.

This is the fresh-context check that the implementing session can't do for itself.
