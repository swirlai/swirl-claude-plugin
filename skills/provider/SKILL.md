---
name: provider
description: |
  SearchProvider wizard: connect a data source to SWIRL by choosing the right
  connector, building the provider JSON, creating it via the API, and testing
  it with a live query. /swirl:provider [source description]
user_invocable: true
---

# SearchProvider Wizard

A SearchProvider is a JSON-configured adapter that tells SWIRL how to query
one source and map its responses into SWIRL's unified result schema. Your job:
get from "I want to search X" to a provider that returns ranked results.

## Step 1 — pick the connector

Built-in connectors (the `connector` field):

| Source type | Connector |
|---|---|
| Any REST API returning JSON (GET) | `RequestsGet` |
| Any REST API returning JSON (POST body) | `RequestsPost` |
| Elasticsearch / OpenSearch | `Elastic` / `OpenSearch` |
| PostgreSQL / Oracle / Snowflake / Sqlite3 / BigQuery / MongoDB | `PostgreSQL`, `Oracle`, `Snowflake`, `Sqlite3`, `BigQuery`, `MongoDB` |
| Vector DBs | `QdrantDB`, `PineconeDB` |
| Microsoft 365 | `M365OutlookMessages`, `M365OneDrive`, `M365OutlookCalendar`, `M365SharePointSites`, `MicrosoftTeams` |
| Google Workspace | `GoogleDriveFiles`, `GoogleGmailMessages`, `GoogleCalendarEvents`, `GooglePeopleContacts` |
| Box | `Box` |
| Generative AI as a source | `GenAI` |
| SWIRL Semantic Cache (Enterprise) | `SwirlCorpusConnector` |

Most REST sources need no code — `RequestsGet`/`RequestsPost` plus mappings
cover them. M365, Google, and Box connectors require an OAuth2 Authenticator
configured first (they delegate the user's own credentials — SWIRL never
sees content the user can't already see).

## Step 2 — build the provider JSON

Core fields:

```json
{
  "name": "My Source",
  "description": "What this source contains and when to use it",
  "connector": "RequestsGet",
  "url": "https://api.example.com/search",
  "query_template": "{url}?q={query_string}",
  "query_mappings": "",
  "result_mappings": "title=name,body=description,date_published=created,url=link,NO_PAYLOAD",
  "results_per_query": 10,
  "credentials": "<api-key-or-auth-spec>",
  "tags": ["example", "docs"],
  "active": true,
  "default": true
}
```

- `query_template` interpolates `{url}` and `{query_string}`; SQL connectors
  interpolate into the query text instead.
- `result_mappings` maps source fields → SWIRL schema
  (`title`, `body`, `url`, `date_published`, `author`); `NO_PAYLOAD` drops
  unmapped fields. Use `FIELD='json.path'` syntax for nested responses —
  fetch one raw response from the source first and map from reality, not
  from API docs.
- `credentials` formats vary by connector (e.g. `bearer=<api-key>`,
  `X-Api-Key=<api-key>` header form, DB connection strings). Never print
  real values back to the user.
- `tags` let users scope searches (`tag:example`) and let agents select
  providers.

## Step 3 — create it

Via the admin UI (`/admin/` → SearchProviders) or the API:

```bash
curl -s -X POST http://localhost:8000/swirl/searchproviders/ \
  -H "Authorization: Token <api-key>" \
  -H "Content-Type: application/json" \
  -d @provider.json
```

## Step 4 — test it (never skip)

```bash
curl -s "http://localhost:8000/swirl/search/?q=<test-term>&providers=<provider-name-or-tag>" \
  -H "Authorization: Token <api-key>"
```

Confirm: results come back, titles/bodies/urls are populated (not raw JSON
blobs), dates parse, and relevancy scores look sane. If zero results, check
the celery worker log for the connector's actual request and the source's
actual response — the error is almost always visible there.

## Gotchas learned the hard way

- A provider that copies raw engine scores (e.g. Elasticsearch `_score`) can
  flood mixed searches — prefer SWIRL's normalized re-ranking unless the
  source is searched alone.
- SQL providers: the query string placement and date fields serialize
  differently per database — always test with a date-bearing row.
- Page size limits are source-side; if the source caps at N, set
  `results_per_query` ≤ N or requests fail.
