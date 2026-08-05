---
name: migrate
description: |
  Migrate or upgrade a SWIRL deployment: Community to Enterprise, version
  upgrades (including 4.x to 5.0), moving the database to external
  PostgreSQL, and air-gapped hosts - with backup doctrine, preflight
  gates, and verification. Guides the user, or executes with granted
  access. /swirl:migrate [from -> to]
user_invocable: true
---

# Migrate SWIRL

Establish the shape first:

1. **Community → Enterprise** - new stack, carry the configuration over.
2. **Version upgrade** - same edition, newer release (4.x → 5.0 detailed below).
3. **Database move** - local Postgres container → external/managed PostgreSQL.

A structural advantage to lean on: SWIRL is federated - there is no search
index to rebuild and no document corpus to move. The sources keep their
data; a migration is configuration plus the Django database.

## Two modes, one contract

You may be **guiding** a user through these steps, or **executing** them
yourself with access the user granted. Either way, these rules are hard:

1. **Never print or echo** license values, passwords, tokens, or `.env`
   contents. Placeholders (`<license-json>`, `<database-password>`) in all
   output; `chmod 600` on every credential-bearing file you write.
2. **Backups are gate zero.** Refuse to proceed past assessment until the
   backups in the doctrine below exist and the dump passed its integrity
   check. Say so explicitly rather than silently skipping.
3. **Explicit informed consent** before: stopping the service (name the
   consequence: "this stops SWIRL for all users until the upgrade
   completes"), any schema migration, any file replacement, any rollback,
   and anything destructive (`down -v`, volume removal, image prune).
4. **One failed step = stop and report.** Show the exact error, the current
   state, and the safe options (fix the specific cause, or roll back).
   Never loop retries of state-changing steps; never run an upgrade twice
   without a fresh state assessment. Rollback is defined before execution
   and executed at most once, deliberately - alternating upgrade/rollback
   attempts corrupts your rollback baseline.
5. **Verify, don't assume**: after every phase, run the verification
   checks and show the evidence (counts, version string, log-gate result)
   before continuing.
6. **Record a timeline**: timestamps for backup completion, service stop,
   and migration start - these are the restore targets.
7. **Scope discipline**: upgrade what you were asked to upgrade. Enabling
   OIDC, cache features, MCP, or TLS changes are separate proposals,
   offered after a green verification, never bundled silently.
8. **Environment honesty**: if the target is not the shape described here
   (no systemd, Kubernetes, marketplace VM images, hand-modified scripts
   beyond recognition), say so and stop rather than adapting on the fly.

## The deployment model (Enterprise Docker Compose)

- The deployment is a directory (canonically `/app`) from the public
  bundle `github.com/swirlai/docker-compose`, usually run by a systemd
  unit `swirl`. `systemctl stop swirl` stops every container including
  the database. When scripting compose directly, pass explicit profiles:
  `COMPOSE_PROFILES=all docker compose stop` - bare `docker compose stop`
  can miss services.
- **`.env` is sourced as shell.** Values with spaces or JSON - notably
  `SWIRL_LICENSE` - must stay single-quoted or the service dies at
  startup with a cryptic `command not found`. After any `.env` edit,
  verify with `bash -c 'source /app/.env'`.
- Compose profiles are computed at start from `.env` flags
  (`USE_LOCAL_POSTGRES` → db, `USE_NGINX` → nginx, `MCP_ENABLED` → mcp);
  the `COMPOSE_PROFILES` line in `.env` only matters for bare
  `docker compose up`.
- One-time seeding runs only while
  `/app/.swirl-application-setup-job-complete.flag` is absent. **An
  upgraded deployment keeps the flag, so new-version seed objects never
  load by themselves** - seeding after upgrade is always an explicit step.
- Run app commands via `docker exec swirl_app ...` - both `swirl.py`
  (orchestration, `load_data`, `reload_ai_prompts`) and `manage.py`
  (Django) live in the app container.
- Enterprise compose creates `admin` from `ADMIN_PASSWORD` with **no
  default** (empty = broken setup job). Never assume or document a
  default password for Enterprise; admin/password is Community-only.

## The invariant phase sequence

Every migration runs these phases in order. Never reorder, never skip:
**Assess → Backup (and verify the backup) → Preflight gates → Execute
(one change domain at a time) → Seed → Verify → (Rollback only if defined
beforehand and needed).**

If both a database move and an app upgrade are planned, do the database
move first, as its own maintenance action - then the app upgrade touches
only images and config.

### Assess

Current version (UI `?` menu or `SWIRL_VERSION` in `.env`), deployment
dir, DB location (local container vs external), TLS shape, network egress
constraints, and custom modifications - diff the deployed scripts and
templates against the pristine bundle of their version; field installs
get hand-edited.

### Backup doctrine

Three backups before any change, in this order:

1. **VM snapshot / restore point** (cloud console) - the primary rollback.
2. **Database safety dump**, taken while running (pg_dump is
   transactionally consistent):
   ```bash
   docker exec swirl_postgres pg_dump -U <sql-user> -d swirl -Fc -f /tmp/pre.dump
   docker cp swirl_postgres:/tmp/pre.dump /app/backups/swirl-preflight-<date>.dump
   docker exec swirl_postgres pg_restore --list /tmp/pre.dump | head   # integrity gate
   ```
   This safety copy is never consumed by the migration (take a separate
   dump at stop time); retain it a week past the migration.
3. **Config backup**: `cp /app/.env /app/.env.pre-upgrade.bak && chmod 600
   /app/.env.pre-upgrade.bak`, plus `docker compose ps` and
   `docker images` state capture.

**A backup you have not restored is a hope, not a backup.** Validate the
restore path on a replica before the maintenance window: dump → fresh
`postgres:16` container → `pg_restore --no-owner --no-privileges` →
row-count comparison (`auth_user`, `swirl_searchprovider`).

App-dir file backup at stop time, before file replacement:
```bash
tar czf /app/backups/app-pre-<ver>-<date>.tar.gz --exclude=/app/backups --exclude=/app/logs /app
```

### Preflight gates

- **Registry reachability** (if pulling):
  `curl -sI --max-time 10 https://registry-1.docker.io/v2/` must return
  **401** - that is the healthy response. A timeout means the firewall is
  closed: stop. Allowlist endpoints: `registry-1.docker.io`,
  `auth.docker.io`, `production.cloudflare.docker.com`, all :443.
- **Disk**: at least 50 GB free for a 5.x image set.
- **License**: the target-version license value in hand (see below).
- **Air-gapped**: choose the image path up front (see air-gap notes).
- Plant a **marker object** (a distinctively named inactive
  SearchProvider) - a cheap tracer that proves data survived
  dump/restore/migrations.

## 4.4.x → 5.0.0.0 specifics

- **Release artifact**:
  `https://github.com/swirlai/docker-compose/archive/refs/tags/v5_0_0_0.tar.gz`.
  GitHub strips the leading `v`: it extracts to `docker-compose-5_0_0_0/`.
  Get this right or every `cd` in your instructions fails.
- **New services** in 5.0: qdrant, seaweedfs, ollama, and a built-in MCP
  server (same app image; off by default). Some 4.x-era service
  containers are retired and linger as stopped orphans after upgrade -
  remove them only after verification passes.
- **Rebuild `.env` from the new `env.example`** - do not patch the old
  file; 5.0 adds many new keys with correct defaults. Carry over: license
  (the NEW value, single-quoted), `ADMIN_PASSWORD`/`ADMIN_USER_EMAIL`,
  all `SQL_*`, `SWIRL_FQDN`/`ALLOWED_HOSTS`/`CSRF_TRUSTED_ORIGINS`/
  `PROTOCOL`, `USE_*` flags, Microsoft auth values, and any custom
  timeout/concurrency overrides. Recommend a fresh `SECRET_KEY`.
  `chmod 600 .env`.
- **File replacement** (preserve `.env`, certs, `uploads/`, `logs/`):
  copy from the new bundle `docker-compose.yml`, `env.example`,
  `Makefile`, `scripts/*`, the nginx entrypoint/reloader/templates, the
  certbot entrypoint, and `doc/`.
- **TLS with a customer-owned cert**: the 5.0 layout is
  `nginx/certificates/ssl/<SWIRL_FQDN>/ssl_certificate.crt` +
  `ssl_certificate_key.key` with `USE_NGINX=true, USE_TLS=true,
  USE_CERT=true`. Older field installs often have hand-modified nginx
  templates and a different cert directory: move the cert files into the
  new layout and use the stock templates; do not carry patched ones.
- **Licensing**: an existing 4.x license **works unchanged on 5.0**
  through its expiration - the signature covers only the fields present,
  so older payloads validate as-is. **No license re-issue is required to
  upgrade.** The separately licensed Semantic Cache is enabled only when
  the license carries the cache entitlement; a customer cannot self-enable
  it (editing the payload invalidates the signature). Without it, behavior
  is clean and inert: the banner shows the cache as unlicensed, the cache
  provider returns no results without errors, cache-dependent features do
  not appear in the UI, and search runs normally. A new license is needed
  only to add the cache entitlement (a commercial decision) or at renewal
  - set that expectation instead of telling every customer to request one.
- **Migrations are NOT purely additive** in the 4.4 → 5.0 chain - some
  steps drop schema. Two consequences: the old version must never run
  against a migrated database, and app rollback requires a database
  restore too (skippable only if the init logs prove migrations never
  ran). Migrations apply automatically at first start via the init
  container.
- **Seed explicitly after the first healthy start** (the setup flag
  suppresses the seed job), then restart the stack so workers pick up
  prompts:
  ```bash
  docker exec swirl_app python swirl.py load_data          # add-only, skips existing
  docker exec swirl_app python swirl.py reload_ai_prompts
  docker exec swirl_app python swirl.py load_branding
  docker exec swirl_app python manage.py reconcile_ollama_url
  ```

## Database move: local container → external PostgreSQL

Do this before the app upgrade. Match the major version (the bundles pin
`postgres:16`), so it is dump/restore, not a version upgrade. Run all
client commands via the `postgres:16` image already on the host.

1. Create the target server. Azure Flexible Server specifics: delegated
   subnet (`/27` or larger, delegated to
   `Microsoft.DBforPostgreSQL/flexibleServers`, empty, dedicated) in the
   same or peered VNet; private access; PostgreSQL 16; automated backups
   on.
2. Create role and database. Azure gotcha - the Flex admin is not
   superuser, so grant first:
   ```sql
   CREATE ROLE <sql-user> LOGIN PASSWORD '<database-password>';
   GRANT <sql-user> TO <admin-user>;    -- required on Azure
   CREATE DATABASE swirl OWNER <sql-user>;
   ```
   Re-use the deployment's existing `SQL_USER`/`SQL_PASSWORD` to minimize
   `.env` churn.
3. Stop the stack; start only the DB
   (`docker compose --profile db up -d postgres`); take the migration
   dump (`pg_dump -Fc`).
4. Restore with `pg_restore --no-owner --no-privileges -h <server-fqdn>
   -U <sql-user> -d swirl <dump>`. Ownership warnings are normal; data
   errors are not. Verify row counts against pre-stop numbers.
5. `.env`: `USE_LOCAL_POSTGRES=false`, `SQL_HOST=<server-fqdn>`,
   `SQL_PORT=5432`, `SQL_SSLMODE="require"` (managed Postgres enforces
   TLS; a lab stand-in container without TLS needs `prefer` instead).
   Restart - the db profile drops out automatically. The local volume
   remains untouched as instant rollback: flip the keys back.
6. Record the timestamp before any later schema-changing step - it is the
   point-in-time-restore target.

## Air-gapped hosts

- Side-load images: on a connected machine,
  `docker pull --platform linux/amd64 <image>` (the platform flag is
  mandatory when pulling on Apple silicon - the target VM is almost
  always amd64), then `docker save | gzip`, transfer, `docker load`.
  A 5.x set is roughly 12 GB compressed; plan transfer time. Grouping
  that works: app image alone, ollama alone, small sidecars together.
- After load: `docker images | grep <version-tag>` is the gate before
  proceeding.
- If a temporary egress window is used instead: run the 401 registry test
  before starting, close the window after, and run the bundle's image
  pull script only after the new `.env` is in place (it reads versions
  and profiles from it).

## Verification (minimum bar)

- All expected containers up and healthy; retired ones absent or
  stopped-orphan only.
- Version string: UI `?` menu, or in-container
  `python -c "from swirl.banner import SWIRL_VERSION; print(SWIRL_VERSION)"`.
- Pre-existing data intact: user count, provider count, and the marker
  object if planted.
- Live federated search through the API with a real token; expect ranked
  results from multiple sources.
- Log gate: zero `ERROR`/`Traceback` in `logs/django.log`; scan
  `docker logs swirl_app` and the init container.
- License banner matches expectation (licensee, entitlement lines).

## Symptom catalog

- Service exits instantly, log shows `<word>: command not found` → an
  unquoted value in `.env` (usually the license). Single-quote it.
- `password authentication failed` in the init container on a "fresh"
  install → a stale named Docker volume reused old DB init. Remove the
  stale volumes or expect the old data. (Labs and re-installs only - in
  a customer upgrade, the volume IS the data.)
- Compose orphan warnings after upgrade → retired services; remove the
  stopped containers after verification, not before.
- Loaded images won't run on the customer VM → arm64 images side-loaded
  from Apple silicon without `--platform linux/amd64`.

## Community → Enterprise

1. **Inventory the current deployment.** List SearchProviders:
   ```bash
   curl -s http://localhost:8000/swirl/searchproviders/ \
     -H "Authorization: Token <api-key>"
   ```
   Note which are customized (non-seed) - those are what you migrate.
   Also note users and any RAG setup (Community: the `OPENAI_API_KEY`
   env var).
2. **Stand up Enterprise** alongside, on a different port - do not
   overwrite the Community stack; you want rollback for free. Use
   `/swirl:install` with the public compose bundle
   (https://github.com/swirlai/docker-compose) and `<license-json>`.
3. **Carry providers over.** For each customized provider: GET it from
   Community, strip `id`, `owner`, and date fields, POST it to the
   Enterprise endpoint. Credentials are write-only - the export will not
   contain usable secrets; the user must re-enter each `<api-key>` /
   connection string. Never ask them to paste secrets into chat - have
   them edit the JSON file or the admin UI directly.
4. **Recreate users** (admin UI or API), then re-do AI setup as
   Enterprise AI Providers (`/swirl:rag`) - the env-var key does not
   carry over.
5. **Verify** with the same query on both stacks side by side: same
   sources responding, results ranked, RAG answering with citations.
   Only then retire the Community stack.
