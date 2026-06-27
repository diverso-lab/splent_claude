# Testing conventions

SPLENT tests use **pytest** with fixtures provided by the framework. Tests live
inside each feature under `tests/`, split by level.

## Fixtures

Every `conftest.py` re-exports the framework fixtures:

```python
from splent_framework.fixtures.fixtures import *  # noqa: F401,F403
```

| Fixture              | Scope    | Use it for |
|----------------------|----------|------------|
| `test_app`           | session  | The Flask app; schema created once per session. |
| `test_client`        | function | **Default.** Test client with a clean DB per test. |
| `test_client_module` | module   | Shared client/DB for a sequence of tests in one module. |
| `clean_database`     | function | Manually reset the DB mid-test. |

Feature-specific fixtures go in the feature's `tests/conftest.py`. Build rows in
an app context, commit, then `db.session.expunge(obj)` before returning so the
object is detached and safe to use across the test.

```python
from splent_framework.fixtures.fixtures import *  # noqa: F401,F403
import pytest
from splent_framework.db import db
from splent_io.splent_feature_notes.models import Notes

@pytest.fixture(scope="function")
def sample_note(test_app):
    with test_app.app_context():
        note = Notes(title="t", body="b", user_id=1)
        db.session.add(note)
        db.session.commit()
        db.session.expunge(note)
        return note
```

## The four levels

### unit/ — isolated, mocked
Verify one class/function; mock the DB, HTTP, and other services. Fast, zero
side effects.

```python
from unittest.mock import MagicMock
from splent_io.splent_feature_notes.services import NotesService

def test_list_for_user_calls_repo():
    service = NotesService()
    service.repository = MagicMock()
    service.list_for_user(7)
    service.repository.get_by_user.assert_called_once_with(7)
```

### integration/ — real DB, no HTTP
Exercise services/repositories against the test database.

```python
def test_create_and_read(test_client):
    with test_client.application.app_context():
        from splent_io.splent_feature_notes.services import NotesService
        svc = NotesService()
        note = svc.create(title="hi", body="x", user_id=1)
        assert svc.get_by_id(note.id).title == "hi"
```

### functional/ — full HTTP via the test client
Hit routes; assert status, redirects, rendered HTML. A login-required route
returns 302 when anonymous.

```python
def test_index_reachable(test_client):
    resp = test_client.get("/notes")
    assert resp.status_code in (200, 302)
```

### e2e/ — Selenium (slow)
Real browser against a running server. Run via `splent selenium`.

### load/ — Locust
`tests/load/test_locustfile.py`, launched with `splent locust`.

## Running tests

- `splent feature:test <feature>` — pytest for one feature.
- `splent feature:test` — all features of the active product.
- `splent product:test` — all feature tests for the product.
- `splent coverage <feature>` — pytest with coverage.

Write at least: a unit test for new service logic, an integration test for new
repository queries, and a functional test for each new route. Replace the
generated `test_placeholder` stubs with real assertions.
