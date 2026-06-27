---
name: product
description: Set up, configure, run, and onboard a SPLENT product. Use whenever the user wants to create a new product, choose its features, derive/build/start it, seed the database, get it running locally, or onboard an existing product after cloning the workspace. Handles the full pipeline (create → select → configure → resolve → derive → seed → run) and day-to-day ops (up/down/restart/logs/console/routes), picking the right path from the current state. Triggers on "set up the product", "run the app", "get it running", "configure features", product:create, product:derive, product:up, or a fresh-clone onboarding.
argument-hint: <what you want, e.g. "create and run my_first_app" or "get the app running">
---

# Set up & run a SPLENT product

Drive a product from nothing to running in the browser, or onboard one that
already exists. The `splent` CLI does the heavy lifting; your job is to pick the
right path from the current state and recover from snags.

Reference: `../conventions/reference/commands.md` (golden paths) and
`../conventions/SKILL.md` (workspace model). Set the runner:
```bash
SPLENT="$(command -v splent >/dev/null 2>&1 && echo splent || echo 'docker exec splent_cli_container splent')"
```

## 1 · Read the current state first

Don't assume — detect:
```bash
$SPLENT product:list          # what products exist
$SPLENT check:docker          # is Docker up
```
Is a product already selected? Does the requested product exist? Is it already
derived/running (`$SPLENT product:containers`)? Choose the path accordingly.

## 2 · Pick the path

### A. Brand-new product
```bash
$SPLENT spl:list                              # which SPL to build from
$SPLENT product:create <name>
$SPLENT product:select <name>
$SPLENT product:configure                     # interactive: choose features
$SPLENT product:resolve                       # symlinks + cache
$SPLENT product:derive --dev                  # build & start (slow first time)
$SPLENT db:seed -y
$SPLENT product:port                          # show URL
```
`product:configure` is an interactive wizard (it walks the UVL tree; mandatory
features are pre-selected). If you can't drive interactive prompts, tell the user
to run it and continue once features are chosen — then verify with
`$SPLENT feature:status`.

### B. Onboard an existing product (fresh clone)
```bash
$SPLENT product:select <name>
$SPLENT product:resolve
$SPLENT product:derive --dev
$SPLENT db:seed -y
```

### C. Just run / restart one that's already set up
```bash
$SPLENT product:up --dev      # or product:restart
$SPLENT product:port
```

## 3 · Verify it's healthy
```bash
$SPLENT product:validate      # UVL satisfiable, contracts compatible, imports OK
$SPLENT product:routes        # routes are registered
$SPLENT product:containers    # containers are up
```
Then give the user the URL (`product:port`) and the seeded login if relevant.

## 4 · Day-to-day operations
```bash
$SPLENT product:up --dev       $SPLENT product:down --dev
$SPLENT product:restart        $SPLENT product:logs
$SPLENT product:console        # Flask shell with the app loaded
$SPLENT product:config         # resolved config with origin tracing
```

## When something fails

`product:derive` chains many steps; if it stops, read the error and route to the
right check rather than retrying blindly:
- missing features → `product:missing` then `product:auto-require`
- UVL unsatisfiable → `product:validate`, `spl:deps <feature>`
- symlinks/cache → `check:features`, re-run `product:resolve`
- env vars → `product:env --generate --merge`, `env:list`
- Docker/infra → `check:docker`, `check:infra`

For anything deeper, use the **doctor** skill.

## Guardrails

- `product:clean` is **destructive** (stops containers, wipes volumes, resets the
  DB, clears uploads). Never run it without explicit confirmation.
- First derive/build takes several minutes — say so, don't assume it hung.
- Don't hand-edit env files, symlinks, or feature lists; use the commands.
