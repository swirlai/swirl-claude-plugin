---
name: troubleshoot
description: |
  Diagnose a misbehaving SWIRL deployment from its logs and live endpoints —
  no results, failing providers, RAG errors, auth failures, UI issues.
  /swirl:troubleshoot [symptom]
user_invocable: true
---

# Troubleshoot SWIRL

The user's report is data — investigate it, don't question it. "Are you
sure?" and "try again" are anti-patterns. Every federation problem leaves a
trace in a log; find the trace before proposing a fix.

## Where the evidence lives

| Evidence | Location |
|---|---|
| Django/API errors, auth failures | `logs/django.log` (local) / `docker compose logs swirl` |
| Connector execution, per-provider errors | celery worker logs (`logs/celery-worker*.log` or the worker container) |
| What the browser actually loaded | DevTools network tab; the served bundle, not source |
| What's actually running | `docker compose ps`, image tags, migration state (`python manage.py showmigrations | grep '\[ \]'`) |
| Liveness | `curl -s <base-url>/swirl/sapi/branding/` |

## Diagnosis by symptom

**All searches return nothing / error immediately**
- HTTP 402 in django.log → license missing or invalid (Enterprise).
- HTTP 401/403 → token wrong or expired; session vs `Authorization: Token
  <api-key>` mismatch.
- Unapplied migrations after an upgrade break in surprising ways (e.g. the
  authenticator list coming back empty). `python manage.py migrate` first,
  ask questions later.

**One provider returns nothing**
- Celery worker log shows the actual request and the source's actual
  response — read it. Typical: expired `<api-key>`, source-side paging cap
  exceeded, response schema drift breaking `result_mappings`.
- Provider inactive, or not `default` and the search didn't select it.
- OAuth2 sources (M365/Google/Box): the *user's* token expired — re-auth in
  the UI; the provider config is usually fine.

**Results come back but ranking looks wrong**
- A provider passing raw engine scores (Elasticsearch `_score` etc.) into
  mixed results outscales everything — normalize instead.
- Embedding re-ranker down or misconfigured → check for embedding errors in
  the worker log; a full stack restart clears transient model-load
  poisoning.

**RAG answer missing or wrong**
- `ai_summary` is asynchronous — poll longer before declaring failure
  (local models: 60–90s cold).
- AI provider inactive / wrong role / token limit exceeded → django.log.
- Bad answers with good sources → retrieval scope problem; fix the search
  before touching prompts.

**UI looks stale after an upgrade**
- Hard refresh (Cmd-Shift-R) — the Galaxy bundle is aggressively cached.
- Verify the served bundle actually changed before debugging "the bug".

## Fix protocol

1. Reproduce with the narrowest command (usually one curl).
2. Read the specific log at the failure timestamp.
3. Change one thing.
4. Restart correctly if needed: the whole stack (`python swirl.py restart`
   or `docker compose restart`), never an individual worker — Celery and
   Daphne must restart together. Confirm with the user before restarting
   anything shared or in use.
5. Re-run the reproduction. Done = the original symptom gone, verified live,
   not "the config looks right now".
