---
name: <project>-controlled-deploy
description: Deploy <project> to <env> safely (backup → build → release → migrate → verify), with the known gotchas. Use when deploying or when the site is down.
---

# Controlled Deploy — <PROJECT>

<One line: what/where prod is, and why care — e.g. "single VM, patient/money-facing; verify every step.">

## Connection / access
- Host: `...`  ·  Key/auth: `...`  ·  App path: `...`
- <local-shell caveats, e.g. macOS zsh has no `timeout`; long builds exceed a 2-min tool timeout →
  run in background>
- DB user/name for backups: `...`

## Procedure (per service)
1. **Back up** the DB/state before any migration: `...`
2. **Fetch + pin** the target commit: `...`
3. **Build** the changed service: `...`
4. **Release / recreate**: `...` ; wait for health: `...`
5. **Migrate** (if schema shipped): `...`  (see the migration-review skill first)
6. **Post-release step** the platform requires: `...`  (e.g. restart the proxy so it re-resolves upstreams)
7. **Verify**: health checks return expected codes AND a real end-to-end check (not just health).

## ⚠️ Gotchas (fill from real incidents — this is the most valuable section)
- <each outage you've hit, its cause, and the fix — e.g. "proxy caches upstream IP at boot → restart
  after recreating the app container"; "proxy refuses to boot if any upstream is down">

## Rollback
- <exact steps to get back to the last-good state; when to restore the DB backup>

## Verdict format
```
Deployed <svc> @ <sha> to <env>.
Migrations: <applied / none>
Health: <checks> = OK
E2E: <what was verified>
```
