# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Janeway is an open-source academic publishing platform (journals, preprints, conferences) built with **Python 3.9+ / Django 4.2**. Developed by the Open Library of Humanities at Birkbeck, University of London. Licensed under AGPL v3.

## Common Commands

### Docker-based development (primary method)

```bash
make install          # Run install_janeway setup
make janeway          # Start dev server (port 8000), alias: make run
make shell            # Interactive bash shell in container
make check            # Run full test suite (SQLite)
make migrate          # Apply migrations
make makemigrations   # Generate migrations
make rebuild          # Rebuild Docker image
make format           # Run ruff formatter
make command CMD="<django_command>"  # Run arbitrary manage.py command
```

### Running tests directly (CI uses this)

```bash
# Full suite
DB_VENDOR=sqlite JANEWAY_SETTINGS_MODULE=core.janeway_global_settings python src/manage.py test

# Single app
python src/manage.py test core

# Single test class
python src/manage.py test core.tests.test_models.TestAccount

# Single test method
python src/manage.py test core.tests.test_models.TestAccount.test_some_method
```

### Code formatting (CI enforced)

```bash
ruff format .           # Format all code
ruff format --check .   # Check without modifying (CI runs this)
```

### CI checks (all must pass)

1. `ruff format --check .` — code formatting
2. `python src/manage.py makemigrations --check` — no unmigrated model changes
3. `python src/manage.py test` — full test suite

## Architecture

### Django Apps (all under `src/`)

The platform follows the **editorial publishing workflow** as its core domain:

- **`core`** — Central app: custom `Account` user model (`AUTH_USER_MODEL = "core.Account"`), `File`, `Galley`, `Setting`/`SettingValue` (dynamic settings system), `Workflow`/`WorkflowElement`, `EditorialGroup`, role-based access (`Role`, `AccountRole`), `DomainAlias` for multi-domain support
- **`submission`** — Article submission intake and management
- **`review`** — Peer review assignments and decisions
- **`copyediting`** — Copyediting workflow
- **`production`** — Production/typesetting management
- **`proofing`** — Proofing assignments
- **`typesetting`** — Typesetting tasks (recently merged from plugin to core)
- **`journal`** — Journal, Issue, Section models
- **`press`** — Press-level management (multi-journal parent entity)
- **`repository`** — Preprint repository
- **`identifiers`** — DOIs, Crossref deposit/polling
- **`cms`** — Pages, navigation items
- **`comms`** — News items
- **`metrics`** — Article access tracking, altmetrics
- **`api`** — REST API (DRF) and OAI-PMH endpoints
- **`events`** — Hook/event registration system for plugins
- **`cron`** — Scheduled task management

### Settings System

Janeway uses a layered settings approach:
- **Base**: `src/core/janeway_global_settings.py` (always loaded)
- **Custom**: set `JANEWAY_SETTINGS_MODULE` env var to override
- **Dev**: `src/core/dev_settings.py` (debug toolbar, console email)
- `INSTALLED_APPS` and `MIDDLEWARE` from custom modules are **merged**, not replaced

The platform also has a **dynamic settings system** via `Setting`/`SettingValue` models — journal-level configuration stored in the database, accessed via `utils.setting_handler`.

### URL Routing

- Root: `src/core/urls.py` → delegates to `src/core/include_urls.py`
- Two modes: `URL_CONFIG = "path"` (path-based) or `"domain"` (domain-based multi-journal)
- Plugin and homepage element URLs are dynamically registered

### Plugin System

- Plugins live in `src/plugins/` (gitignored)
- `core/plugin_loader.py` handles discovery
- Homepage elements in `src/core/homepage_elements/` (about, carousel, featured, html, issue, journals, news, popular, preprints, search_bar)
- Hooks registered via `events/registration.py`

### Themes

Three built-in themes in `src/themes/`: **OLH**, **material**, **clean**. Template override middleware allows per-journal customization.

### Testing Utilities

`src/utils/testing/helpers.py` provides factory functions: `create_user()`, `create_press()`, `create_journals()`, `create_submission()`, `create_review_assignment()`, etc. Tests use `django.test.TestCase` with SQLite in CI.

## Conventions

- **Formatting**: Ruff (line length 88, double quotes, Python 3.9 target). Excludes: `plugins/`, `static/`, `media/`, `files/`
- **Commit messages**: Conventional Commits format with issue references (e.g., `fix: #123 description`)
- **Branch naming**: `<issue_number>-Feature` or `<issue_number>-Hotfix`
- **Pre-commit hook**: `git config --local core.hooksPath .git-hooks` to enable ruff format check
- **Migrations**: Never modify existing migration files; always create new ones
- **Database**: `DB_VENDOR` env var selects backend (postgres/mysql/mariadb/sqlite)
- **i18n**: Uses `django-modeltranslation` for model field translation; supported languages: en, en-us, fr, de, nl, cy, es
