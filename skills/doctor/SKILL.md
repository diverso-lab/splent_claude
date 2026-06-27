---
name: doctor
description: Diagnose and fix problems in a SPLENT workspace. Use whenever something is broken or behaving oddly — product:derive or product:up fails, the app won't start, a UVL model is unsatisfiable, migrations fail or the DB is out of sync, features are missing or won't import, symlinks/cache are broken, env vars or Docker/infra are misconfigured, or doctor/product:validate report errors. Triggers on error output, "it's broken", "won't start", "derive failed", "migration error", "feature not found", "UVL unsatisfiable", or any SPLENT troubleshooting request.
argument-hint: [symptom or pasted error]
---

# Diagnose & fix SPLENT issues

Work like a doctor: reproduce, run the *right* diagnostic, read it carefully,
apply the smallest fix, then re-verify. Don't retry the failing command blindly.

Reference: `../conventions/reference/commands.md`. Set the runner:
```bash
SPLENT="$(command -v splent >/dev/null 2>&1 && echo splent || echo 'docker exec splent_cli_container splent')"
```

## 1 · Triage

Start broad, then narrow:
```bash
$SPLENT doctor                 # full health check (--fast skips slow checks)
```
`doctor` runs every `check:*`. Read which check failed and jump to that area
below. If the user pasted an error, match it to a symptom first.

## 2 · Symptom → diagnostic → fix

### Docker / infra won't come up
```bash
$SPLENT check:docker      # daemon + compose present & running
$SPLENT check:infra       # ports, services, networks
```
Fix: start Docker Desktop; free the conflicting port; recreate containers.

### `feature:clone` / `feature:install` fails on git
Almost always the **`.gitconfig` is a directory, not a file** (the #1 cause).
```bash
$SPLENT check:github      # GITHUB_USER / GITHUB_TOKEN present
```
Fix (host): `rm -rf ~/.gitconfig && touch ~/.gitconfig` then set `user.name`
/`user.email`. `make preflight` checks this.

### Features missing / won't import
```bash
$SPLENT product:missing      # required by UVL but absent from pyproject
$SPLENT product:auto-require  # add them from UVL constraints
$SPLENT check:features        # cache, symlinks, install state
$SPLENT check:deps            # imports respect UVL constraints
```
Fix: `product:auto-require` then `product:resolve`. If symlinks are stale,
re-run `product:resolve`; prune junk with `cache:prune`.

### UVL unsatisfiable / validation errors
```bash
$SPLENT product:validate          # UVL + contracts + imports
$SPLENT spl:deps <feature>        # what a feature implies (--reverse for dependents)
$SPLENT spl:configurations        # is any valid selection possible
```
Fix: add the missing required feature, or correct the constraint with
`spl:add-constraints` / `spl:add-feature`. Don't hand-edit the `.uvl`.

### Migrations fail / DB out of sync
```bash
$SPLENT db:status                 # per-feature migration state
```
Fix:
- pending changes not generated → `db:migrate <feature>` then `db:upgrade`.
- autogenerate misses mixin/refinement columns → `db:migrate <feature> --empty`
  and write it by hand.
- a bad migration → `db:rollback <feature>`, fix, regenerate.
- last resort (DESTRUCTIVE, confirm first) → `db:reset` then `db:seed -y`.

### Env / config wrong
```bash
$SPLENT env:list                  # workspace .env (masked)
$SPLENT product:config            # resolved Flask config with origins
$SPLENT product:env --generate --merge
```
Never edit the workspace `.env` directly — it holds tokens/credentials. Use the
commands.

### App starts but a route/feature misbehaves
```bash
$SPLENT product:routes            # is the route registered & attributed
$SPLENT feature:xray --validate   # hook/extension collisions
$SPLENT product:signals           # signal listeners
$SPLENT feature:test <feature>    # reproduce in a test
$SPLENT product:logs              # runtime errors
```

## 3 · Fix, then prove it

Apply the smallest change, then re-run the diagnostic that failed **and** a
broad check:
```bash
$SPLENT product:validate && $SPLENT doctor --fast
```
Report: what was wrong, the root cause, the fix you applied, and the green
verification output. If the fix touched feature code, run `feature:test`.

## Guardrails

- Confirm before anything destructive: `db:reset`, `product:clean`,
  `cache:clear`.
- Never modify the workspace `.env` (tokens/credentials live there).
- Prefer the CLI's repair commands over manual edits to pyproject/UVL/symlinks.
- If a command's flags differ from what's written here, check
  `$SPLENT <command> --help` — the CLI is the source of truth.
