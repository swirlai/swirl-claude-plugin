---
name: connector
description: |
  Develop a custom SWIRL connector when no built-in connector fits the
  source - subclassing patterns, federation lifecycle, and testing.
  /swirl:connector [source description]
user_invocable: true
---

# Custom Connector Development

Before writing code, challenge the premise: most sources are covered by
`RequestsGet`/`RequestsPost` with mappings (see `/swirl:provider`). Write a
connector only when the source needs custom request construction, a
non-HTTP protocol, multi-step auth, or response handling that mappings
can't express.

## Architecture

Connectors live in `swirl/connectors/`. Every connector subclasses the base
`Connector` class and participates in the federation lifecycle, executed per
provider, per search, inside a Celery task:

1. `process_query()` - transform the user's query for this source.
2. `construct_query()` - build the request (URL, body, SQL, SDK call).
3. `execute_search()` - perform it; store the raw response.
4. `normalize_response()` - convert to SWIRL's result list (dicts with
   `title`, `body`, `url`, `date_published`, `author`, `payload`).

The base class handles retries, timing, status, and result persistence - 
override only what differs. HTTP sources should subclass the `Requests`
family instead of the root class to inherit auth handling, paging, and
`query_template` interpolation. Database sources should mirror an existing
DB connector (`postgresql.py` is the reference pattern: connect, execute
with the query interpolated safely, map rows, always close in `finally`).

## Rules

- **Parameterize, never interpolate, SQL values.** The query string goes
  into a parameterized query; string-building SQL from user input is an
  injection bug.
- Respect `results_per_query`; never fetch unbounded result sets.
- Errors: log the source's actual response body at warning level (scrubbed
  of credentials) and return an empty result set - one dead source must
  never fail the federated search.
- Credentials come from the provider's `credentials` field - never from
  code, never logged.
- Register the connector: add it to the module registry in
  `swirl/connectors/__init__.py` and to the SearchProvider
  `CONNECTOR_CHOICES` so it appears in the admin UI.

## Testing

1. Unit-test `normalize_response()` with a captured real response fixture - 
   the empty-results case, the error case, and a full page.
2. Create a SearchProvider using the new connector and run a live query:
   ```bash
   curl -s "http://localhost:8000/swirl/search/?q=<test-term>&providers=<name>" \
     -H "Authorization: Token <api-key>"
   ```
3. Watch the celery worker log during the search - that's where connector
   exceptions surface.
4. Mixed-search test: run the new provider alongside 2–3 others and confirm
   its results rank sensibly (normalized scores, not raw engine scores).

A connector is done when a federated search including it returns correctly
mapped, sensibly ranked results and a source outage degrades gracefully.
