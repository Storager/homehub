# AGENTS.md

HomeHub is a self-hosted Flask + SQLAlchemy (SQLite) + Jinja + Tailwind 3 family dashboard, shipped as a Docker image and upgraded in place. CI and the Docker image use Python 3.14.

## Commands

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt          # runtime deps + pytest
python -m pytest                             # full suite; self-contained, no services
python -m pytest tests/test_expenses.py -q   # one file; add ::test_name for one test

cp config-example.yml config.yml              # optional; without it the app uses the example config
npm install && npm run build:css              # builds gitignored static/output.css
FLASK_DEBUG=1 python run.py                   # http://localhost:5000 with reloader/debugger

flask translations update                     # after adding/changing user-facing text
flask translations compile                    # local run; Docker does this at build time
```

CI runs only `python -m pytest`; there is no lint, typecheck, or formatter config.

## Architecture

- The app factory is `create_app` in `app/__init__.py`. Routes live in `app/blueprints/*` and all attach to one `main_bp`; `create_app` imports each module for side effects. A new route module must be imported there or its routes will never register. `app/routes.py` is a legacy import shim with no routes.
- `config.yml` is optional (falls back to `config-example.yml`) and is reloaded per request, cached by file mtime. New keys need a default in `app/config.py` (`_parse_config`) and an entry in `config-example.yml`; existing configs without the key must keep working.
- SQLite is `data/app.db` in WAL mode; `data/`, `uploads/`, `media/`, and `pdfs/` are runtime dirs. There is no Alembic: startup `app/migrations.py` upgrades existing databases additively. A new column needs the model field **and** an entry in `COLUMNS` (existing table) or `LATE_COLUMNS` (table may have just been created). New tables go in `TABLES` or must be safe for `db.create_all()`. Never rename, drop, or change a column type in startup migrations.
- Production runs one gunicorn worker with 4 threads (`Dockerfile`). Recurring generation relies on a process-wide lock (`app/recurrence.py`), and media downloads use daemon threads. Do not raise the worker count without redesigning those.
- Tailwind scans `templates/` and `static/js/`; rebuild CSS after changing classes. `static/output.css` is build output and is gitignored.

## Conventions

- There are no per-member accounts. The acting user is unauthenticated, so request-supplied names (`user`, `creator`, `payer`, ...) must be sanitized and authorized with `can_modify` / `is_admin` from `app/admin.py`. Never compare names directly to `admin_name`, `Administrator`, or `admin`.
- There are no CSRF tokens, by design. Cross-site writes are rejected in `app/blueprints/auth.py` using `Sec-Fetch-Site` or `Origin`; keep state changes on POST/PUT/PATCH/DELETE, never GET.
- Store user text after `sanitize_text`; rich text after `sanitize_html`. Fetch user-supplied URLs server-side only after `is_url_safe_for_fetch`. Jinja autoescapes; use `|safe` only on HTML cleaned with `sanitize_html`.
- Money: use `_split_shares` in `app/blueprints/expenses.py` so rounded parts sum exactly (the payer absorbs the difference). Do not round each share independently.
- Recurring dates: use `app/recurrence.py`; count from the rule's start date, not the previous occurrence. Do not rewrite past occurrences when editing a rule unless that is the explicit operation.
- Time: store naive UTC with `app/clock.utcnow()` and display via the `localtime` filter or `format_local_datetime`. Do not use `datetime.utcnow()`.
- i18n: wrap Python/Jinja text in `_()` / `ngettext()`; page scripts use `t()`, `th()` for HTML strings, `tn()` for plurals, and `tc()` for context. After changing strings run `flask translations update`; do not hand-edit other languages' `.po` files.

## Tests

- Route/DB tests typically build their own app with `create_app({'TESTING': True, 'SQLALCHEMY_DATABASE_URI': 'sqlite://', 'SECRET_KEY': 'test'})`, then `db.create_all()`. Follow that pattern; never use the real database.
- `tests/conftest.py` autouse fixtures point `CONFIG_PATH` at a copy of `config-example.yml` and redirect uploads/PDFs to `tmp_path`, so tests need no local `config.yml` and will not modify one. If you monkeypatch config, reset `config._cache`.
- Startup migrations are skipped under `TESTING`; test them by calling `migrations.run()` on a temporary SQLite file.
- The suite has no external services, so `python -m pytest` is the authoritative check.
- i18n tests compile `.mo` files from `.po` into `translations/`; these files are gitignored.

## Repo hygiene

- `compose.yml` is the published GHCR config; do not switch it to a local build.
- Do not commit `config.yml`, `.env`, `data/`, `uploads/`, `media/`, `pdfs/`, `static/output.css`, or `node_modules/`.
- `README.md` covers user-facing setup, config, theming, and languages. `.github/skills/code-review/SKILL.md` has the PR review checklist; its notes about tests needing `config.yml` or failing on main are stale.
