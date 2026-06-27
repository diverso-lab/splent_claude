---
name: feature
description: Author a SPLENT feature end-to-end. Use whenever the user wants to build, add, scaffold, or extend a feature in a SPLENT workspace — from a plain description like "add a notes feature with title and body owned by the user" all the way to working code. It will scaffold with feature:create, write models/repositories/services/routes/templates/hooks and tests following SPLENT conventions, add it to the SPL, install it into the product, run migrations, and verify. Triggers on "new feature", "add a feature", splent_feature_*, feature:create, or any request to implement domain functionality in a SPLENT product.
argument-hint: <description of the feature to build>
---

# Author a SPLENT feature, end to end

Turn a plain-language request into a working, tested, wired-up feature. The
`splent` CLI handles the mechanics; you write the logic and drive the commands.

First read the conventions reference so the code is idiomatic SPLENT:
- `../conventions/reference/feature-anatomy.md` (file-by-file patterns)
- `../conventions/reference/testing.md` (fixtures & test levels)
- `../conventions/reference/commands.md` (commands & golden paths)

Set up the CLI runner once and reuse it:
```bash
SPLENT="$(command -v splent >/dev/null 2>&1 && echo splent || echo 'docker exec splent_cli_container splent')"
```

## 1 · Understand the request, then confirm a short plan

Infer as much as possible; ask only what you genuinely can't:

- **Name** → `splent-io/splent_feature_<name>` (snake_case). Derive from the
  description unless given (org defaults to `splent-io`).
- **Archetype** → choose the lightest that fits (see the matrix in
  feature-anatomy.md):
  - has DB models → `full`
  - UI/pages only, no tables → `light`
  - backend logic/commands, no UI → `service`
  - infrastructure/env only → `config`
- **Domain** → entities, fields, relationships. A `ForeignKey` to another
  feature's table (e.g. `user.id`) means this feature **requires** that feature.
- **Target** → which SPL (`$SPLENT spl:list`) and which product
  (`$SPLENT product:list`). Default to the workspace's only SPL/product if there
  is just one.

State the plan in 2-3 lines (name, archetype, models, routes, requires) and
proceed. Don't over-interview — simplicity first.

## 2 · Scaffold

```bash
$SPLENT product:deselect                  # feature creation is workspace-level
$SPLENT feature:create splent-io/splent_feature_<name> --type <archetype>
```

Then **read what was generated** (list the package dir) so you edit the real
stubs instead of recreating them. Look at a sibling feature for live patterns if
one exists.

## 3 · Implement, layer by layer

Only the files your archetype includes. Follow feature-anatomy.md exactly
(imports, blueprint naming `<short>_bp`, PascalCase classes, namespaced
templates).

1. **`models.py`** — define entities; FKs for relations.
2. **`repositories.py`** — `super().__init__(<Model>)`; add custom queries.
3. **`services.py`** — business logic over the repository.
4. **`__init__.py`** — register the service in `init_feature(app)`.
5. **`routes.py`** — resolve via `service_proxy("<Name>Service")`; namespaced
   templates; `@login_required` where it touches a user.
6. **`forms.py`** — WTForms for any write routes.
7. **`templates/<short>/…`** — the UI; extend the product's base template.
8. **`hooks.py`** — surface the feature in shared layout slots
   (`layout.authenticated_sidebar`, etc.) so it's reachable in the UI.
9. **`seeders.py`** — a little realistic sample data.

Keep it real: implement what the user asked for, not placeholder TODOs.

## 4 · Contract & config

`feature:contract` and `feature:inject-config` **require an active product** —
if none is selected yet, run them right after step 6 (`product:select`).

```bash
$SPLENT feature:contract splent_feature_<name>        # infer provides/requires
$SPLENT feature:inject-config splent_feature_<name>   # only if it reads env vars
```
Review the inferred `[tool.splent.contract]`. Inference scans Python imports, so
a **foreign key referenced as a string** (e.g. `db.ForeignKey("user.id")`) is
NOT detected as a dependency — set `requires.features` by hand to list every
feature you FK into (e.g. `features = ["auth"]`).

## 5 · Tests

Replace the generated `test_placeholder` stubs (see testing.md):
- a **unit** test for new service logic (mock the repo),
- an **integration** test for new repository queries (real DB),
- a **functional** test per new route.

The test DB is **MariaDB/InnoDB with foreign keys enforced**. If your model FKs
another feature's table (e.g. `user_id → user.id`), you must **create those
parent rows first** (e.g. `from splent_io.splent_feature_auth.models import User`,
add + commit a User, use its real id) — otherwise inserts raise IntegrityError
and the test errors out before asserting. Don't assume `user_id=1` exists.

## 6 · Wire into the SPL and install into the product

```bash
$SPLENT spl:add-feature <spl> splent_feature_<name>   # detached; auto-detects deps
$SPLENT product:select <product>
$SPLENT feature:add splent-io/splent_feature_<name>   # editable install
```
Notes from real runs:
- `spl:add-feature` is **interactive** (`Add '<name>'? [Y/n]`) and needs the
  feature **editable at the workspace root** (a freshly-created one is). Pipe the
  answer if you can't type: `printf 'Y\n' | … spl:add-feature …`. It scans the
  source for deps and adds the right `=>` constraints (a string FK like
  `user.id` IS detected here, even though `feature:contract` misses it).
- To add an existing **versioned** feature instead, use
  `feature:attach <org>/<feature> v<version>` (the version needs the `v` prefix);
  `feature:unlock <org>/<feature>` converts a versioned feature to editable
  (requires an active product).

Never hand-edit the product's `pyproject.toml` feature lists or the `.uvl` file —
these commands do it correctly.

## 7 · Migrate (full archetype)

```bash
$SPLENT db:migrate splent_feature_<name>     # generate from model changes
$SPLENT db:upgrade                           # apply
```
If autogenerate can't see the changes (e.g. mixin/refinement columns), use
`db:migrate splent_feature_<name> --empty` and write the migration by hand.

## 8 · Verify and report

```bash
$SPLENT feature:test splent_feature_<name>
$SPLENT feature:xray --validate
$SPLENT product:validate
$SPLENT product:restart            # then check the route in the browser
```

Report concisely: feature name & archetype, files written, what it requires, the
route/URL to view it, and test results. If a command fails, diagnose with the
**doctor** skill's approach (read the right `check:*`/`product:validate` output,
fix, re-run) rather than guessing.

## Guardrails

- The CLI owns mechanics (pyproject lists, UVL, env, symlinks, migrations); you
  own logic. Don't bypass it.
- Read before you write — don't clobber generated stubs blindly.
- Confirm before destructive commands (`product:clean`, `db:reset`).
- Match the surrounding code's style; reuse base classes instead of reinventing.
