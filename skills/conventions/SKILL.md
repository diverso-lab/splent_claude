---
name: conventions
description: SPLENT framework architecture and conventions — the canonical reference for working in a SPLENT codebase. Consult this whenever you are reading, writing, or reviewing SPLENT code such as features, products, blueprints, services, repositories, models, routes, template hooks, the service locator, per-feature Alembic migrations, the feature contract in pyproject.toml, the SPL/UVL variability model, or the splent CLI. Use it before writing any feature/product code so it follows the real patterns instead of generic Flask. Triggers on mentions of splent, splent_feature_*, splent_io, feature:create, product:derive, BaseService/BaseRepository, UVL, SPL, or work inside splent_cli/splent_framework/splent_catalog.
---

# SPLENT conventions

SPLENT is a **Software Product Line (SPL)** framework: products are assembled
from reusable **features**, with the valid combinations described by a **UVL**
variability model in a **catalog**. The `splent` CLI does all the mechanics.

Read this map, then open the reference file you need. Keep code idiomatic to
SPLENT — not generic Flask.

## The workspace

A SPLENT workspace is a folder containing these sibling repos/dirs:

- `splent_cli/` — the `splent` command (≈120 commands, runs in Docker).
- `splent_framework/` — Flask app factory, base classes, fixtures, hooks, db.
- `splent_catalog/` — SPL definitions: `<spl>/metadata.toml` + `<spl>/<spl>.uvl`.
- `<product>/` — products (e.g. `my_first_app`) built from an SPL.
- Features live in the local cache or as editable repos, wired into products.

## Running the splent CLI

`splent` runs **inside the `splent_cli_container` Docker container**; the
workspace is bind-mounted at `/workspace`. From Claude Code (host) you normally
cannot call `splent` directly. Use this rule every time you run a command:

- If `command -v splent` succeeds → you're inside the container; run `splent …`.
- Otherwise → run `docker exec splent_cli_container splent …`.

A robust one-liner to reuse:

```bash
SPLENT="$(command -v splent >/dev/null 2>&1 && echo splent || echo 'docker exec splent_cli_container splent')"
$SPLENT --help
```

Edit files with normal tools (host paths); they're the same files mounted in the
container. **When unsure of a command's exact flags, run `$SPLENT <command> --help`** —
the CLI is the source of truth and may evolve.

## Golden rules

1. **The CLI owns the mechanics.** Never hand-edit the `features`/`features_dev`/
   `features_prod` lists in a product's `pyproject.toml`, the `.uvl` model, env
   merging, symlinks, or Alembic version tables. Use the commands
   (`feature:add`, `spl:add-feature`, `product:env`, `db:migrate`…).
2. **You own the logic.** Models, services, routes, templates, hooks, tests —
   write these by hand, following the conventions below.
3. **Validate, don't assume.** After changes run `product:validate`,
   `feature:xray --validate`, `feature:test`, and `doctor`.
4. **Detached vs selected.** Workspace-level work (creating a feature, editing an
   SPL) is done *detached* (`product:deselect`). Product-level work needs a
   selected product (`product:select <name>`).

## Feature anatomy (the essentials)

- Python namespace: `splent_io.splent_feature_<name>`; package dir
  `src/splent_io/splent_feature_<name>/`.
- Blueprint: created with `create_blueprint(__name__)` → named `<short>_bp`.
- Four **archetypes**: `full` (models+routes+services+UI+migrations), `light`
  (routes+hooks+templates, no DB), `service` (services+config+commands+signals,
  no UI), `config` (config.py only).
- Layers: `models.py` → `repositories.py` (extends `BaseRepository`) →
  `services.py` (extends `BaseService`) → `routes.py` (uses `service_proxy`).
- `__init__.py` exposes the blueprint, `init_feature(app)` (registers services),
  and `inject_context_vars(app)`.
- Cross-feature UI via **template hooks** (`register_template_hook("layout.…")`).
- Contract lives in `pyproject.toml` under `[tool.splent.contract]`.

→ Full file-by-file patterns with code: **`reference/feature-anatomy.md`**

## Tests

Four levels under `tests/`: `unit/` (mocked), `integration/` (real DB via
service/repo), `functional/` (HTTP via Flask test client), `e2e/` (Selenium).
Fixtures come from `from splent_framework.fixtures.fixtures import *`.

→ Fixtures, scopes, and templates: **`reference/testing.md`**

## Commands

The command catalog grouped by area, plus the **golden-path sequences** for the
common workflows (create a feature, set up a product, migrate, debug).

→ **`reference/commands.md`**

## When to use the workflow skills

- Building/changing a feature end-to-end → use **`/splent:feature`**.
- Creating, configuring, deriving or running a product → **`/splent:product`**.
- Something is broken (derive fails, UVL unsat, migrations, env/Docker) →
  **`/splent:doctor`**.
