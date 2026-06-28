# Feature anatomy — file-by-file conventions

Canonical patterns for a SPLENT feature. These mirror what `splent feature:create`
generates; fill them with real logic. Always read the actual generated files
before editing — don't recreate what scaffolding already produced.

## Contents
- [Naming & layout](#naming--layout)
- [`__init__.py`](#initpy)
- [`config.py`](#configpy)
- [`models.py`](#modelspy)
- [`repositories.py`](#repositoriespy)
- [`services.py`](#servicespy)
- [`routes.py`](#routespy)
- [`forms.py`](#formspy)
- [`hooks.py` & the hook namespace](#hookspy--the-hook-namespace)
- [Contributing to the main nav](#contributing-to-the-main-nav)
- [`seeders.py`](#seederspy)
- [The feature contract](#the-feature-contract-pyprojecttoml)
- [Archetype file matrix](#archetype-file-matrix)
- [Base class APIs](#base-class-apis)
- [Migrations](#migrations)

## Naming & layout

```text
splent_feature_<name>/
├── pyproject.toml                          # metadata + [tool.splent.contract]
└── src/splent_io/splent_feature_<name>/
    ├── __init__.py        models.py        repositories.py   services.py
    ├── routes.py          forms.py         hooks.py          signals.py
    ├── config.py          commands.py      seeders.py        decorators.py
    ├── migrations/env.py
    ├── templates/<short>/index.html
    ├── templates/hooks/…
    ├── assets/{css,js,dist}
    └── tests/{unit,integration,functional,e2e,load}
```

- **Namespace**: `splent_io.splent_feature_<name>` (the `splent_io` org folder).
- **`<short>`**: the feature name without the `splent_feature_` prefix
  (e.g. `splent_feature_notes` → `notes`). The blueprint is `notes_bp`.
- **PascalCase** of `<short>` names the model/service/repo classes
  (`Notes`, `NotesService`, `NotesRepository`).

## `__init__.py`

`create_blueprint(__name__)` infers the short name and sets template/static
folders. `init_feature(app)` registers services; `inject_context_vars(app)`
returns a dict merged into the Jinja context.

**full / service archetype:**
```python
from splent_framework.blueprints.base_blueprint import create_blueprint
from splent_framework.services.service_locator import register_service

from splent_io.splent_feature_notes.services import NotesService

notes_bp = create_blueprint(__name__)

def init_feature(app):
    register_service(app, "NotesService", NotesService)

def inject_context_vars(app):
    return {}
```

**light archetype** (no services):
```python
from splent_framework.blueprints.base_blueprint import create_blueprint

notes_bp = create_blueprint(__name__)

def init_feature(app):
    pass

def inject_context_vars(app):
    return {}
```

**config archetype** (injects config on init):
```python
from splent_framework.blueprints.base_blueprint import create_blueprint

redis_bp = create_blueprint(__name__)

def init_feature(app):
    from splent_io.splent_feature_redis.config import inject_config
    inject_config(app)

def inject_context_vars(app):
    return {}
```

## `config.py`

Expose env vars into `app.config` so the framework can track them. Prefer
`splent feature:inject-config` to auto-generate this by scanning the source.

```python
import os  # noqa: F401

def inject_config(app):
    app.config.update({
        # "NOTES_PAGE_SIZE": int(os.getenv("NOTES_PAGE_SIZE", "20")),
    })
```

## `models.py`

`db` is the shared SQLAlchemy instance from the framework. Models subclass
`db.Model`. Use `db.ForeignKey` to reference another feature's tables (this is
what `spl:add-feature` auto-detects as a dependency constraint).

```python
from splent_framework.db import db

class Notes(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(255), nullable=False)
    body = db.Column(db.Text, default="")
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)

    def __repr__(self):
        return f"Notes<{self.id}>"
```

## `repositories.py`

Thin: pass the model to `BaseRepository`. Add custom queries as methods.

```python
from splent_io.splent_feature_notes.models import Notes
from splent_framework.repositories.BaseRepository import BaseRepository

class NotesRepository(BaseRepository):
    def __init__(self):
        super().__init__(Notes)

    def get_by_user(self, user_id: int) -> list[Notes]:
        # Add a deterministic tiebreaker (id) when ordering by a coarse column:
        # created_at is second-precision, so rows in the same second would
        # otherwise have undefined order.
        return (
            self.model.query.filter_by(user_id=user_id)
            .order_by(Notes.created_at.desc(), Notes.id.desc())
            .all()
        )
```

## `services.py`

Business logic. Inject the repository in `__init__`. Routes never touch
repositories directly — they go through the service.

```python
from splent_io.splent_feature_notes.repositories import NotesRepository
from splent_framework.services.BaseService import BaseService

class NotesService(BaseService):
    def __init__(self):
        super().__init__(NotesRepository())

    def list_for_user(self, user_id: int):
        return self.repository.get_by_user(user_id)
```

## `routes.py`

Resolve services with `service_proxy("<Name>Service")` at module level — it
returns a proxy that fetches the (possibly refined) service on every access, so
refinement features can override it.

```python
from flask import render_template
from flask_login import current_user, login_required
from splent_io.splent_feature_notes import notes_bp
from splent_framework.services.service_locator import service_proxy

notes_service = service_proxy("NotesService")

@notes_bp.route("/notes", methods=["GET"])
@login_required
def index():
    notes = notes_service.list_for_user(current_user.id)
    return render_template("notes/index.html", notes=notes)
```

Templates are namespaced under the short name: `templates/notes/index.html`
renders as `render_template("notes/index.html")`.

## `forms.py`

WTForms via Flask-WTF (CSRF is enabled framework-wide).

```python
from flask_wtf import FlaskForm
from wtforms import StringField, TextAreaField, SubmitField
from wtforms.validators import DataRequired, Length

class NotesForm(FlaskForm):
    title = StringField("Title", validators=[DataRequired(), Length(max=255)])
    body = TextAreaField("Body")
    submit = SubmitField("Save")
```

## `hooks.py` & the hook namespace

Hooks inject HTML fragments into the product's layout slots — this is how a
feature adds UI to shared pages without editing the base template. Register in
`hooks.py`; the slot names come from the product's `base_template.html`
(`[tool.splent.layout].hooks`), e.g. `layout.authenticated_sidebar`,
`layout.anonymous_sidebar`, `layout.head.css`, `layout.scripts`.

```python
from flask import render_template, url_for
from splent_framework.hooks.template_hooks import register_template_hook

def notes_sidebar():
    return render_template("hooks/sidebar_items.html")

register_template_hook("layout.authenticated_sidebar", notes_sidebar)

# Only if the feature ships compiled assets in assets/dist/:
def notes_scripts():
    return ('<script src="'
            + url_for("notes.assets", subfolder="dist", filename="splent_feature_notes.bundle.js")
            + '"></script>')

register_template_hook("layout.scripts", notes_scripts)
```

Hook API (`splent_framework.hooks.template_hooks`):
- `register_template_hook(name, func)` — append a callback (additive).
- `replace_template_hook(name, func)` — replace all callbacks (used by
  refinement features to override a base feature's hook).
- `remove_template_hook(name, func)` / `get_template_hooks(name)`.

Manage hooks with `splent feature:hook:add` / `feature:hook:remove` /
`feature:hooks`, and inspect wiring with `feature:xray`.

## Contributing to the main nav

A content feature that owns a public page declares **one** entry in the main
navigation from inside `init_feature(app)` with `register_nav_item` — that's how
the public menu is *composed from the installed features* instead of hardcoded
per product. Install the feature → its entry appears; remove it and re-derive →
it disappears (derivation-time variability).

```python
from splent_framework.nav.nav_registry import register_nav_item

def init_feature(app):
    register_nav_item(key="post", label="Blog", href="/blog", order=50)
```

`key` is unique per feature (idempotent), `order` sorts the menu (default `100`),
and `icon` is optional. The theme reconciles these base entries with the runtime
admin **Menus** override and falls back to `SITE_NAV` only if no feature
registered anything. See [Navigation registry](/cli/feature/nav) for the full
contract.

## `seeders.py`

Subclass `BaseSeeder`, implement `run()`, insert with `self.seed([...])`. Run
with `splent db:seed`.

```python
from splent_framework.seeders.BaseSeeder import BaseSeeder
from splent_io.splent_feature_notes.models import Notes

class NotesSeeder(BaseSeeder):
    def run(self):
        self.seed([
            Notes(title="Welcome", body="First note", user_id=1),
        ])
```

## The feature contract (`pyproject.toml`)

Declares what the feature provides/requires and what is extensible. Used by
`product:validate` and `product:derive`. Prefer `splent feature:contract` to
infer it from source rather than editing by hand.

```toml
[tool.splent]
cli_version = "1.8.0"

[tool.splent.contract]
description = "notes feature"

[tool.splent.contract.provides]
routes = ["/notes"]
blueprints = ["notes_bp"]
models = ["Notes"]
commands = []

[tool.splent.contract.requires]
features = ["auth"]      # because Notes.user_id → user.id
env_vars = []

[tool.splent.contract.extensible]
services = []
templates = []
models = []
hooks = []
routes = false
```

## Archetype file matrix

| File / dir            | full | light | service | config |
|-----------------------|:----:|:-----:|:-------:|:------:|
| `__init__.py`         |  ✓   |   ✓   |    ✓    |   ✓    |
| `config.py`           |  ✓   |       |    ✓    |   ✓    |
| `models.py`           |  ✓   |       |         |        |
| `repositories.py`     |  ✓   |       |         |        |
| `services.py`         |  ✓   |       |    ✓    |        |
| `routes.py`           |  ✓   |   ✓   |         |        |
| `templates/`          |  ✓   |   ✓   |         |        |
| `hooks.py`            |  ✓   |   ✓   |         |        |
| `signals.py`          |  ✓   |       |    ✓    |        |
| `commands.py`         |  ✓   |       |    ✓    |        |
| `migrations/`         |  ✓   |       |         |        |
| `tests/unit`          |  ✓   |   ✓   |    ✓    |   ✓    |
| `tests/integration`   |  ✓   |       |         |        |
| `tests/functional`    |  ✓   |   ✓   |         |        |

Pick the lightest archetype that fits. Adding a model later means the feature
should be `full`.

## Base class APIs

**`BaseRepository`** (`splent_framework.repositories.BaseRepository`) —
`__init__(model)`, then: `create(commit=True, **kwargs)`, `get_by_id(id)`,
`get_by_column(name, value)`, `get_or_404(id)`, `update(id, **kwargs)`,
`delete(id)`, `delete_by_column(name, value)`, `count()`.

**`BaseService`** (`splent_framework.services.BaseService`) — `__init__(repository)`,
then thin pass-throughs: `create`, `count`, `get_by_id`, `get_or_404`,
`update`, `delete`. Add domain methods on top.

**`BaseSeeder`** (`splent_framework.seeders.BaseSeeder`) — implement `run()`;
helper `seed(list_of_objects)` inserts and returns them with IDs.

**Service locator** (`splent_framework.services.service_locator`):
- `register_service(app, name, cls)` — register (later calls override → refinement).
- `service_proxy(name)` — lazy proxy used in routes.
- `get_service_class(app, name)`, `get_all_services(app)`.

**App factory** (`splent_framework.app_factory`):
`create_splent_app(import_name, config_name="development")`. A product's
`src/<product>/__init__.py` is just:
```python
from splent_framework.app_factory import create_splent_app
def create_app(config_name="development"):
    return create_splent_app(__name__, config_name)
```

## Migrations

Each feature owns its Alembic version table (`alembic_<feature_name>`), so
migrations never collide across features. `migrations/env.py` imports the
feature's models and calls `run_feature_migrations(FEATURE_NAME, FEATURE_TABLES)`.

```python
from splent_io.splent_feature_notes import models  # noqa
from splent_framework.migrations.feature_env import run_feature_migrations

FEATURE_NAME = "splent_feature_notes"
FEATURE_TABLES = set()  # empty → auto-detect from imported models

run_feature_migrations(FEATURE_NAME, FEATURE_TABLES)
```

Workflow:
- `splent db:migrate <feature>` — **generate** a migration from model changes.
- `splent db:upgrade` — **apply** pending migrations (all, or one feature).
- `splent db:status` — see migration state.
- Refinement features that add columns via mixins can't be autogenerated — use
  `db:migrate --empty` and write the migration by hand.
