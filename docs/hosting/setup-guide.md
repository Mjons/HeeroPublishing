# Janeway — Quick Reference

## 1. Run It Locally (Just to See It)

You need Docker installed. Then two commands:

```bash
make install       # first time only — sets up DB, asks you to create admin account
make janeway       # starts the server
```

Open http://localhost:8000 — that's it. Login with the admin account you just created.

There is NO separate frontend. It's one Django app that serves everything.

---

## 2. Host It for Production

### Where to Host

Any VPS provider works. Cheapest options:
- **DigitalOcean** — $6/mo droplet (good docs, beginner friendly)
- **Hetzner** — $4/mo (cheapest, EU-based)
- **Linode** — $5/mo
- **Vultr** — $5/mo

Pick the smallest plan to start (1 CPU, 1-2GB RAM). You can upgrade later.

### What to Do on the VPS

```bash
# 1. SSH into your server
ssh root@your-server-ip

# 2. Install Docker
curl -fsSL https://get.docker.com | sh

# 3. Clone YOUR fork
git clone https://github.com/YOUR_USERNAME/HeeroPublishing.git
cd HeeroPublishing

# 4. Create production settings
cp src/core/example_settings.py src/core/settings.py
```

### 5. Edit settings.py — What to Change

Open `src/core/settings.py` and change these things:

```python
# CHANGE THIS — generate a random string, 50+ characters
SECRET_KEY = "put-a-long-random-string-here-never-share-it"

# CHANGE THIS — your actual domain
DEFAULT_HOST = "https://yourdomain.com"

# EMAIL — so the platform can send emails (use a provider like Brevo, SendGrid, etc)
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
# Then configure SMTP host/port/user/password in the Django email settings

# DATABASE — use PostgreSQL for production
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "janeway",
        "USER": "janeway",
        "PASSWORD": "a-strong-password",
        "HOST": "localhost",
        "PORT": "5432",
    }
}

# URL CONFIG — "path" = one domain, "domain" = each journal gets its own domain
URL_CONFIG = "path"
```

Then:
```bash
# 6. Install and run
make install
make janeway       # for testing — use gunicorn + nginx for real production
```

For proper production you'd put **nginx** in front as a reverse proxy and use **gunicorn** instead of `runserver`, plus **Let's Encrypt** for SSL. But `make janeway` works to verify everything is set up.

---

## 3. Sync Code from Original Repo (Step by Step)

Git only syncs CODE files. Your database (journals, articles, users) is never affected.

### First time ever (do this once):

```bash
git remote add upstream https://github.com/openlibhums/janeway.git
```

### Every time you want their latest updates:

```bash
# Step 1: Make sure you have no uncommitted changes
git status
# If you see changes, commit them first:
#   git add .
#   git commit -m "my changes"

# Step 2: Fetch what's new from the original repo
git fetch upstream

# Step 3: Make sure you're on master
git checkout master

# Step 4: Merge their code into yours
git merge upstream/master

# Step 5: If there are conflicts, git will tell you which files.
# Open those files, fix the conflicts (look for <<<< ==== >>>> markers),
# then:
#   git add .
#   git commit -m "merge upstream"

# Step 6: Push to YOUR GitHub
git push origin master

# Step 7: If the merge included new migration files, apply them
make migrate
```

That's it. Your database content stays untouched. Only the code updates.

### Keeping Your Custom Changes Safe

We've made custom edits to some files (aesthetic/branding changes). When merging from upstream, these are the files to watch:

**Files we customized:**
- `src/themes/OLH/templates/core/base.html` — removed Janeway footer logo
- `docs/hosting/` — new documentation files (won't conflict, upstream doesn't have them)

**What happens during merge:**
- **New files you added** (like docs) — merge cleanly, no risk
- **Files only upstream changed** — merge cleanly, no risk
- **Files BOTH you and upstream changed** — git will flag a **merge conflict**

**If you get a merge conflict:**
1. Git tells you which files have conflicts
2. Open the file — look for `<<<<<<<`, `=======`, `>>>>>>>` markers
3. Keep the version you want (usually your version for branding changes)
4. Remove the conflict markers
5. Save, then `git add .` and `git commit -m "resolve merge conflict"`

**Pro tip:** Before merging, always check what files upstream changed:
```bash
git fetch upstream
git diff master..upstream/master --name-only
```
This shows which files changed. If none of your customized files are in the list, the merge is safe.

---

## 4. What You Need to Know About the Database

- **Locally:** Docker creates PostgreSQL automatically. You don't configure anything.
- **Production:** YOU create a PostgreSQL database and put the credentials in `settings.py`.
- **Your data is yours.** Syncing git never touches it.
- **Migrations** = small code files that update your database STRUCTURE (add columns, tables). They're in git. After pulling new ones: `make migrate`.
- **SQLite** works for quick local testing. Never use it in production.
- **The database is managed through the app.** You create content (journals, users, articles) via the web interface, not by writing SQL.

---

## 5. Commands Cheat Sheet

| Command | What it does |
|---------|-------------|
| `make install` | First-time setup |
| `make janeway` | Start dev server (port 8000) |
| `make shell` | Terminal inside the container |
| `make migrate` | Apply database structure changes |
| `make check` | Run tests |
| `make format` | Format code with ruff |
| `make build_assets` | Rebuild CSS |

---

## 6. Janeway Concepts (Quick Summary)

- **Press** = top level (your organization). Has one or many journals.
- **Journal** = has issues, sections, articles.
- **Submission flow:** Author submits → Editor assigns reviewers → Peer review → Copyediting → Production/Typesetting → Proofing → Publish
- **Galleys** = final formatted files (PDF, HTML, XML) readers download
- **Roles** = author, editor, reviewer, copyeditor, typesetter, proofreader, reader
- **Themes** = OLH, material, clean — control how the public site looks
- **Settings** = most config done through the web admin, stored in database
- **Plugins** = extend functionality (homepage elements, custom features)
- **Identifiers** = DOI registration via Crossref
- **Repository** = preprint server (separate from journals)

**Admin pages:**
- `/manager/` — journal management dashboard
- `/admin/` — Django admin (low-level database access)

---

## 7. Official Docs

https://janeway.readthedocs.io/ — you don't need them to run it, but useful later for:
- How to use the publishing interface (editor/manager guides)
- Configuration options and themes
- Plugin system
