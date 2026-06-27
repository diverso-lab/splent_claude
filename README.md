# splent — Claude Code plugin for the SPLENT product line

Work at **SPLENT speed**. This plugin teaches Claude Code how SPLENT is built —
features, products, the SPL/UVL model, per-feature migrations, the feature
contract — and gives it skills that drive the `splent` CLI for you.

The `splent` CLI already automates the *mechanics* (scaffolding, env merge,
symlinks, migrations, derive, release). This plugin adds the *judgment*: writing
idiomatic feature code and tests, knowing which command to run in which order,
and recovering when something breaks.

## Install

```text
/plugin marketplace add diverso-lab/splent_claude
/plugin install splent@diverso-lab
```

That's it. No per-project config. Works in any SPLENT workspace (a folder
containing `splent_cli`, `splent_framework`, `splent_catalog` and your products).

To update later: `/plugin marketplace update diverso-lab` then reinstall.

## What you get

| Skill | Invoke | What it does |
|-------|--------|--------------|
| **feature** | `/splent:feature <description>` | Author a feature end-to-end: scaffold → write models/services/routes/templates/hooks + tests → add to the SPL → install into the product → migrate → verify. |
| **product** | `/splent:product <what you want>` | Create, configure, derive, seed and run a product — or onboard an existing one. Picks the right path from the current state. |
| **doctor** | `/splent:doctor [symptom]` | Diagnose and fix: failing `derive`, UVL unsatisfiable, broken migrations, missing features, symlink/cache/env/Docker issues. |
| **conventions** | (automatic) | Always-on SPLENT knowledge. Claude consults it whenever you work in a SPLENT codebase, so generated code follows the real patterns. |

You don't have to use the slash commands — just describe what you want
("add a notes feature with title and body owned by the user") and Claude will
pick up the right skill automatically.

## How it runs the CLI

`splent` lives inside the `splent_cli_container` Docker container, with the
workspace bind-mounted at `/workspace`. The skills detect this and call
`docker exec splent_cli_container splent …` from the host (or `splent …`
directly if you're already inside the container). Files are edited on the host;
the same files are mounted in the container.

## Design principles

- **The CLI is the source of truth.** Skills drive `splent` commands and never
  hand-edit pyproject feature lists, the UVL model, or migration version tables.
- **Confirm before trusting.** Skills check exact flags with `--help` at runtime,
  so they survive CLI version changes.
- **Simplicity first.** Sensible defaults, minimal questions, "it just works."

## Layout

```text
splent_claude/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace listing (diverso-lab)
└── skills/
    ├── conventions/         # always-on framework knowledge + deep reference/
    ├── feature/             # author a feature end-to-end
    ├── product/             # set up & run a product
    └── doctor/              # diagnose & fix
```

## License

Creative Commons CC BY 4.0 — SPLENT — Diverso Lab.
