---
name: railway
description: |
  Deploy, configure, manage, and troubleshoot SWIRL Enterprise on Railway -
  the one-click hosted template: deploy flow, variables, AI providers,
  Microsoft 365 connection, the Railway API, version updates, and
  operations recipes. /swirl:railway [what they want to do]
user_invocable: true
---

# SWIRL on Railway

Railway is the hosted alternative to running SWIRL's Docker Compose stack
yourself: a one-click template deploys the full SWIRL 5 Enterprise stack
into the customer's own Railway workspace, on their bill.

- Template: https://railway.com/deploy/swirl-enterprise
- Services deployed: `swirl` (app + celery workers), `postgres`, `redis`,
  `qdrant`, `seaweedfs` (cache body storage, S3 API), `tika` (text
  extraction), `ollama` (local model, CPU only).
- The SWIRL app image is private; the template carries hidden registry
  credentials that attach automatically. Deployers need the Railway **Pro
  plan** for private-image templates.
- Resources are usage-billed per second; services scale vertically. There
  are no instance types to pick.
- License: the customer requests one at https://swirlaiconnect.com/deploy/
  (delivered by email). Docs: https://docs.swirlaiconnect.com/railway-guide

Conduct rules for this skill, on top of the usual ones: never echo license
values, API keys, passwords, or Railway variable values - the Railway
variables API returns DECRYPTED values, so extract the single value you
need programmatically and never print query results raw. Destructive
operations (cache reset) require explicit user consent with the
consequence named.

## Deploy walkthrough

1. Railway account on the Pro plan, logged in.
2. Open the template URL, click Deploy.
3. Fill the prompted variables (below). Only `SWIRL_LICENSE` requires real
   thought; everything else has working defaults or generators.
4. Wait for all services to go green. The `swirl` service is last -
   migrations, data load, and collectstatic run at boot.
5. Open the swirl service's public domain (Railway generates
   `<name>.up.railway.app`). Log in as `admin`; the password is the
   generated `ADMIN_PASSWORD`, readable in the swirl service's Variables
   tab. Have the user read it there - do not fetch and display it.
6. First search works out of the box (Google, News, ArXiv, SWIRL docs).
   RAG answers need an AI provider (next section).

## Template variables

Prompted / worth knowing (names only - never echo values):

- `SWIRL_LICENSE` - multiline license JSON. Mandatory for Enterprise
  features; Semantic Cache additionally requires the cache entitlement in
  the license.
- `ANTHROPIC_API_KEY` - optional, strongly recommended. When set, boot
  activates a Claude provider (claude-haiku-4-5) as the default rag + chat
  provider; RAG answers take roughly 15 seconds. Without it, RAG falls
  back to CPU Ollama: expect 30 to 90+ seconds per answer.
- `ADMIN_EMAIL`, `ADMIN_PASSWORD` - admin account; the password is
  generated if not supplied.
- `M365_AUTHENTICATOR_B64` - optional; enables Microsoft 365 sources (see
  below).
- `SWIRL_RAG_PAGEFETCH_TEXT_EXTRACT_TIMEOUT` - seconds for Tika extraction
  during cache promotion. 300 is the tested Railway value; the code
  default of 90 is too low for multi-MB PDFs there.

Wired automatically by the template - do not touch: `SQL_*`, `CELERY_*`,
Redis URLs, `SWIRL_QDRANT_URL`, `SWIRL_STORAGE_*`, `TIKA_SERVER_ENDPOINT`,
`SWIRL_OLLAMA_URL`, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`,
`IN_PRODUCTION`, `SWIRL_FQDN`, `PORT`. Registry credentials on the swirl
service are template-provided and immutable - do not attempt to change
them.

## AI providers

- With `ANTHROPIC_API_KEY` set at deploy: Claude is active and default for
  rag + chat. Nothing else to do.
- The key can be added AFTER deploy: set the variable on the swirl
  service, then redeploy the service - the boot bootstrap is idempotent
  and picks it up.
- Ollama runs CPU-only and stays enabled for auxiliary functions
  (embedding support, date extraction). Never position local-model chat as
  the primary path; it is a fallback, and it is slow on shared CPU.

## Microsoft 365 connection

Two parts: an Azure app registration in the customer's tenant, and a SWIRL
authenticator delivered through `M365_AUTHENTICATOR_B64`.

Azure app requirements:
- Delegated Graph scopes: `User.Read Mail.Read Files.Read.All
  Calendars.Read Sites.Read.All Chat.Read offline_access`.
- Redirect URI (type Web):
  `https://<their-swirl-domain>/swirl/callback/microsoft-callback` - this
  must be added for the Railway-generated domain exactly, or OAuth fails.

`M365_AUTHENTICATOR_B64` is base64 of a JSON object:

```json
{
  "name": "Microsoft",
  "idp": "Microsoft",
  "client_id": "<azure-app-client-id>",
  "client_secret": "<client-secret>",
  "tenant_id": "<tenant-id>",
  "callback_path": "/swirl/callback/microsoft-callback",
  "scopes": "User.Read Mail.Read Files.Read.All Calendars.Read Sites.Read.All Chat.Read offline_access",
  "broker_mode": "harvest",
  "active": true
}
```

Help the user build this in a LOCAL file where they fill in the secret
values themselves, then base64 it and set the variable - never have them
paste the client secret into chat. Boot applies it, rewrites `app_uri` to
the deploy's own domain, and activates the five M365 SearchProviders
(Teams, Outlook Messages, Outlook Calendar, OneDrive, SharePoint Sites).
After deploy, each user connects Microsoft from the SWIRL UI (profile /
authenticator button) via a consent + redirect round trip.

Known behavior: Microsoft tokens expire after roughly a day. When expired,
all five M365 sources show `status: ERROR, retrieved: -1` with no visible
error banner. The fix is reconnecting Microsoft in the UI - check this
FIRST whenever "M365 sources return nothing".

## Managing a deployment via the Railway API

Endpoint `https://backboard.railway.com/graphql/v2`, header
`Authorization: Bearer <railway-token>` - the CUSTOMER's token, created in
their Railway account settings.

Gotchas:
- Railway's edge returns 403 to the default python-urllib user agent. Use
  curl, or set a curl/browser-like User-Agent.
- The `variables` query returns decrypted values: extract what you need
  programmatically; never print the response.

Proven operations (GraphQL):

```text
# Latest deployment status
query { deployments(first: 1, input: { projectId: "...", environmentId: "...",
  serviceId: "..." }) { edges { node { id status } } } }

# Boot/runtime logs
query { deploymentLogs(deploymentId: "...", limit: 200) { message severity timestamp } }

# Read variables (extract single values; never print raw)
query { variables(projectId: "...", environmentId: "...", serviceId: "...") }

# Set/replace variables
mutation { variableCollectionUpsert(input: { projectId: "...", environmentId: "...",
  serviceId: "...", variables: { NAME: "value" } }) }

# Delete a variable
mutation { variableDelete(input: { projectId: "...", environmentId: "...",
  serviceId: "...", name: "NAME" }) }

# Change start command / healthcheck
mutation { serviceInstanceUpdate(serviceId: "...", environmentId: "...",
  input: { startCommand: "..." }) }

# Redeploy a service (required after variable/start-command changes)
mutation { serviceInstanceRedeploy(serviceId: "...", environmentId: "...") }

# Find service ids
query { project(id: "...") { services { edges { node { id name } } } } }

# Public domain
query { domains(projectId: "...", environmentId: "...", serviceId: "...") {
  serviceDomains { domain } } }
```

## Updating SWIRL versions

- The template pins a specific image tag. New SWIRL releases ship as
  template updates; existing deployments do NOT auto-update.
- Redeploying the swirl service re-pulls the pinned tag. Moving to a NEW
  version is a SWIRL-published template change - direct customers to SWIRL
  support for version upgrades. Do not document or attempt editing the
  image source: the hidden registry credentials make user-side source
  edits fail.
- Database migrations run automatically at boot, so a same-stack redeploy
  after a template update is the whole upgrade procedure.

## Operations recipes

- **Health**: `https://<domain>/swirl/health/celery/` (also Railway's
  probe path).
- **Logs without a Railway token**: log into the SWIRL UI, then
  `https://<domain>/swirl/logs.html?file=<name>.log` - useful names:
  `django.log`, `celery-worker.log`, `celery-search-worker.log`,
  `celery-corpus-worker.log` (cache promotion),
  `celery-pagefetch-worker.log`.
- **Semantic Cache clean-slate reset** (destructive - explicit consent
  first; intended for test/eval resets):
  1. Check the swirl service's current start command. If it already
     carries the `WIPE_CACHE` gate
     (`if [ "$WIPE_CACHE" = "1" ]; then python manage.py
     reset_semantic_cache --apply --yes; fi && ...`), skip to step 2;
     otherwise add the gate via `serviceInstanceUpdate`.
  2. Set `WIPE_CACHE=1` and redeploy the swirl service.
  3. Confirm in deploy logs: "Semantic cache reset summary" with removed
     signal counts, and "swirl_corpus re-created via init_qdrant".
  4. DELETE the `WIPE_CACHE` variable so later redeploys do not wipe
     again. This step is not optional.
- **Scripted SWIRL management on this image generation**:
  token-authenticated writes to `/api/swirl/...` return 403. For
  mutations, use the `/swirl/...` prefix with a session cookie plus
  `X-CSRFToken` header, or make the change in the UI. Reads with
  `Authorization: Token <api-key>` work on both prefixes.

## Symptom catalog

- Search runs but the UI never shows results; browser console shows
  mixed-content errors -> `IN_PRODUCTION` is not true -> set it, redeploy.
- Healthcheck failing on deploy -> `healthcheck.railway.app` missing from
  `ALLOWED_HOSTS` -> restore the template default.
- All M365 sources ERROR / retrieved -1 -> expired Microsoft token ->
  reconnect Microsoft in the UI. If it never worked, check the Azure app
  redirect URI matches the deploy domain exactly.
- Caching a document spins for minutes per doc -> seaweedfs lacks volume
  capacity (S3 writes retry about 2 minutes then give up; documents index
  without stored bodies) -> the seaweed start command must include
  `-master.volumeSizeLimitMB=1024` (present in the current template; older
  deploys may lack it). Healthy promotion is 1 to 2 seconds per document;
  `storage_write_failed` lines in `celery-corpus-worker.log` mean the flag
  is missing.
- Large scanned PDFs fail to cache with "Apache Tika unreachable" -> that
  message usually means the extraction TIMED OUT, not that Tika is down ->
  raise `SWIRL_RAG_PAGEFETCH_TEXT_EXTRACT_TIMEOUT` (300 recommended). Very
  large scanned documents may exceed what shared-CPU OCR can do; set that
  expectation honestly.
- RAG returns an LLM provider error mentioning a bare model name -> the
  local model path requires the `ollama/` model prefix; the current
  template's boot handles this - a redeploy on the current template fixes
  it.
- Chat/RAG very slow with no Anthropic key -> expected on CPU Ollama; add
  `ANTHROPIC_API_KEY`.
- Branding/logo 500s after deploy -> stale deploy from an older template;
  redeploy on the current template.

## What you may claim (verified on a live deploy)

Federated search across web + M365 sources, neural re-ranking, RAG with
Claude at roughly 15-second answers, Semantic Cache promotion at 1 to 2
seconds per document, authenticated M365 content via Graph, version
clustering of near-duplicate documents, canonical version election with
scored reasoning, persistence across restarts, per-user M365 OAuth.

Do not claim: GPUs, deploy-time guarantees, auto-upgrades, or local-model
chat performance.
