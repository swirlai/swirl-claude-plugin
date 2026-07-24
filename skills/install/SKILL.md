---
name: install
description: |
  Guided installation of SWIRL — Docker Compose (recommended) or local
  development install — with post-install verification. /swirl:install
user_invocable: true
---

# Install SWIRL

Walk the user through getting SWIRL running, then verify it actually works
with a live search. Ask which edition they have before anything else:

- **SWIRL Community** — open source, https://github.com/swirlai/swirl-search
- **SWIRL Enterprise** — licensed; the user receives an image reference,
  a Docker Compose bundle, and a license file from SWIRL.

## Path A — Docker Compose (recommended)

Confirm Docker Desktop (or engine) is installed and running: `docker info`
must succeed.

**Community** (documented quick start):
```bash
curl https://raw.githubusercontent.com/swirlai/swirl-search/main/docker-compose.yaml -o docker-compose.yaml
docker-compose pull && docker-compose up
```
Default login is `admin` / `password` — have the user change it immediately.

**Enterprise**: use the compose bundle provided by SWIRL. The license goes
in the environment file as `SWIRL_LICENSE=<license-json>` — never commit
that file, never echo its contents.

Either way, wait for the stack: the `swirl` container logs show the init
sequence (migrations, seed data) before Django starts —
`docker compose logs -f swirl` until the server accepts connections. UI at
`http://localhost:8000/galaxy/`.

## Path B — local (non-Docker) install

**Community**: follow the Quick Start at https://docs.swirlaiconnect.com —
the Docker path above is the supported fast route; don't improvise local
commands from memory.

**Enterprise** (requires Python 3.11+; PostgreSQL for production-like use):

```bash
git clone -b main <swirl-repo-url> swirl
cd swirl
./install.sh
python swirl.py config_db        # schema, migrations, initial admin user
python swirl.py load_data        # branding, SearchProviders, AI Providers
python manage.py collectstatic
python swirl.py start
```

`swirl.py start` launches the full stack (Django/Daphne + Celery workers).
To restart, always restart the whole stack — `python swirl.py restart` —
never individual workers; Celery and Daphne must restart together.

## Verify (always do this)

1. `curl -s http://localhost:8000/swirl/sapi/branding/` → JSON = server up.
2. Log into `/galaxy/`, run a search against a pre-loaded web provider
   (e.g. an arXiv or web search provider) and confirm ranked results render.
3. Check `logs/django.log` and the celery worker logs for stack traces even
   if the UI looks fine — connector errors are logged, not always surfaced.

## Common install failures

| Symptom | Cause / fix |
|---|---|
| Port 8000 in use | Another stack running; stop it or change the published port. |
| Init container loops | Database not ready or stale volume; `docker compose down -v` for a truly clean start (destroys data — confirm first). |
| UI loads, all searches fail | Migrations unapplied or license invalid (Enterprise → HTTP 402). Run `python manage.py migrate`; check the license. |
| Blank provider list | Seed step skipped; run `python swirl.py load_data` (local) or check init logs (Docker). |

Never run destructive resets (`down -v`, DB drops) without telling the user
data will be lost and getting an explicit yes.
