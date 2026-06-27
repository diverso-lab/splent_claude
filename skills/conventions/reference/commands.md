# splent command reference & golden paths

The `splent` CLI is the source of truth. This is a map of the useful commands
plus the **golden-path sequences** for common work. Confirm exact flags with
`$SPLENT <command> --help` — the CLI evolves.

```bash
# Resolve how to call splent (host vs inside container):
SPLENT="$(command -v splent >/dev/null 2>&1 && echo splent || echo 'docker exec splent_cli_container splent')"
```

## Command catalog (by area)

**🌿 Feature** — lifecycle & authoring
`feature:create` (scaffold, `--type full|light|config|service`) ·
`feature:add` (register editable into product) · `feature:attach` (register
versioned) · `feature:install` (interactive: clone→register→env→start) ·
`feature:remove` / `feature:detach` · `feature:clone` · `feature:unlock`
(released→editable) · `feature:pin` · `feature:upgrade` / `feature:outdated` ·
`feature:list` / `feature:status` / `feature:order` / `feature:impact` ·
`feature:contract` (infer contract) · `feature:inject-config` (generate
config.py) · `feature:test` · `feature:compile` (webpack) ·
`feature:hook:add` / `feature:hook:remove` / `feature:hooks` · `feature:xray`
(visualize extensions/hooks; `--validate`) · `feature:release`.

**🏗️ Product** — assemble, run, deploy
`product:create` · `product:select` / `product:deselect` · `product:configure`
(pick features from the SPL) · `product:resolve` (sync versioned features into
cache + symlinks) · `product:derive` (full pipeline: sync→env→build→start;
`--dev`) · `product:up` / `product:down` / `product:restart` · `product:logs` ·
`product:console` (Flask shell) · `product:shell` · `product:routes` ·
`product:config` · `product:validate` (UVL + contracts + imports) ·
`product:missing` / `product:auto-require` · `product:env` (`--generate`,
`--merge`) · `product:clean` (DESTRUCTIVE) · `product:release`.

**🧬 SPL / UVL** — variability model (run detached)
`spl:list` · `spl:info` · `spl:features` · `spl:deps <feature>` (`--reverse`) ·
`spl:add-feature <spl> <feature>` (auto-detects constraints) ·
`spl:add-constraints` · `spl:configurations` · `spl:create` · `spl:fetch`.

**🧱 Database**
`db:migrate <feature>` (generate; `--empty` for hand-written) · `db:upgrade`
(apply) · `db:status` · `db:rollback <feature>` · `db:reset` · `db:seed`
(`-y`) · `db:console` · `db:dump` / `db:restore`.

**💾 Cache**
`cache:status` · `cache:size` · `cache:usage` · `cache:versions` ·
`cache:orphans` · `cache:prune` · `cache:clear`.

**🔍 Checks**
`check:env` · `check:docker` · `check:infra` · `check:product` ·
`check:features` · `check:deps` · `check:github` · `check:pypi` ·
`check:pyproject`. And `doctor` runs them all (`--fast`).

**🧰 Utilities**
`lint` (Ruff, `--fix`) · `coverage <feature>` · `selenium` · `locust` /
`locust:stop` · `command:create` · `export:puml` · `version` ·
`clear:build` / `clear:log` / `clear:uploads` · `env:list` / `env:show` ·
`release:cli` / `release:framework`.

## Golden paths

### Create a feature, end to end
```bash
$SPLENT product:deselect                                   # work detached
$SPLENT feature:create splent-io/splent_feature_notes --type full
# … edit models/repositories/services/routes/templates/hooks + tests …
$SPLENT feature:contract splent_feature_notes             # infer the contract
$SPLENT feature:inject-config splent_feature_notes        # if it reads env vars
$SPLENT spl:add-feature sample_splent_spl splent_feature_notes   # into the SPL
$SPLENT product:select my_first_app
$SPLENT feature:add splent-io/splent_feature_notes        # install (editable)
$SPLENT db:migrate splent_feature_notes                   # generate migration
$SPLENT db:upgrade                                         # apply it
$SPLENT feature:test splent_feature_notes                 # verify
$SPLENT feature:xray --validate                           # check wiring
$SPLENT product:validate                                  # check the product
```

### Stand up a product from scratch
```bash
$SPLENT spl:list
$SPLENT product:create my_first_app
$SPLENT product:select my_first_app
$SPLENT product:configure                                 # choose features
$SPLENT product:resolve                                   # symlinks + cache
$SPLENT product:derive --dev                              # build & start (slow first time)
$SPLENT db:seed -y
$SPLENT product:port                                      # get the URL
```

### Onboard an existing product
```bash
$SPLENT product:select my_first_app
$SPLENT product:resolve
$SPLENT product:derive --dev
$SPLENT db:seed -y
```

### Day-to-day
```bash
$SPLENT product:up --dev      $SPLENT product:down --dev
$SPLENT product:restart       $SPLENT product:logs
$SPLENT product:routes        $SPLENT product:console
```

### When something is wrong
```bash
$SPLENT doctor                 # full health check (--fast to skip slow ones)
$SPLENT check:product          $SPLENT check:features      $SPLENT check:infra
$SPLENT product:validate       $SPLENT product:missing
$SPLENT db:status
```

## Notes & gotchas

- Running `splent` **detached** prints harmless import/SESSION_TYPE errors
  because no product is loaded — that's normal for workspace-level commands.
- `product:clean` and `db:reset` are **destructive** (volumes/DB). Confirm with
  the user first.
- The `.gitconfig` must be a *file*, not a directory — the #1 cause of
  `feature:clone`/`feature:install` failures. `make preflight` checks it.
- First `product:derive` / `make setup` builds images and takes several minutes;
  later runs reuse layers.
