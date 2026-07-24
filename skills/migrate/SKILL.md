---
name: migrate
description: |
  Migrate a SWIRL deployment: Community to Enterprise, or an older version
  to a newer one - inventory, config carry-over, verification.
  /swirl:migrate [from -> to]
user_invocable: true
---

# Migrate SWIRL

Two migration shapes. Establish which one first:

1. **Community → Enterprise** - new stack, carry the configuration over.
2. **Version upgrade** - same edition, newer release.

A structural advantage to lean on: SWIRL is federated - there is no search
index to rebuild and no document corpus to move. The sources keep their
data; a SWIRL migration is configuration (providers, auth, AI setup) plus
the Django database. Set that expectation up front - it's why SWIRL
migrations are small compared to migrating an indexing search engine.

## Community → Enterprise

1. **Inventory the current deployment.** List SearchProviders:
   ```bash
   curl -s http://localhost:8000/swirl/searchproviders/ \
     -H "Authorization: Token <api-key>"
   ```
   Note which are customized (non-seed) - those are what you migrate. Also
   note users and any RAG setup (Community: the `OPENAI_API_KEY` env var).
2. **Stand up Enterprise** alongside, on a different port - do not overwrite
   the Community stack; you want rollback for free. Use `/swirl:install`
   with the SWIRL-provided compose bundle and `<license-json>`.
3. **Carry providers over.** For each customized provider: GET it from
   Community, strip `id`, `owner`, and date fields, POST it to the
   Enterprise endpoint. Credentials are write-only - the export will not
   contain usable secrets; the user must re-enter each `<api-key>` /
   connection string. Never ask them to paste secrets into chat - have them
   edit the JSON file or the admin UI directly.
4. **Recreate users** (admin UI or API), then re-do AI setup as Enterprise
   AI Providers (`/swirl:rag`) - the env-var key does not carry over.
5. **Verify** with the same query on both stacks side by side: same sources
   responding, results ranked, RAG answering with citations. Only then
   retire the Community stack.

## Version upgrade (either edition)

1. **Back up the database first** - Docker: `docker compose exec` a
   `pg_dump` (PostgreSQL) or copy the SQLite file out. No backup, no
   upgrade. Confirm the backup is non-empty before proceeding.
2. Pull the new image (or fetch the new release) and restart the stack.
3. **Apply migrations** - the init sequence usually runs them; verify with
   `python manage.py showmigrations | grep '\[ \]'` (empty = fully
   applied). Unapplied migrations are the classic post-upgrade failure:
   symptoms range from 500s to lists silently coming back empty.
4. Hard-refresh the UI (Cmd-Shift-R) - the Galaxy bundle caches
   aggressively; a stale bundle mimics a broken upgrade.
5. **Verify**: liveness (`/swirl/sapi/branding/`), a federated search
   returning results, a RAG answer, and a scan of `logs/django.log` and the
   celery worker logs for new stack traces.

## Rules

- Never migrate or upgrade a stack someone is actively using or demoing on
  without an explicit OK.
- Roll forward only after verification; keep the old stack (or the DB
  backup) until the new one has passed a real-usage check.
- Major-version upgrades can ship breaking changes - check the release
  notes at https://docs.swirlaiconnect.com before promising a clean jump
  across multiple major versions.
