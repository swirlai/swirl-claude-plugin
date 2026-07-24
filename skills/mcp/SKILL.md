---
name: mcp
description: |
  Connect Claude (Desktop, Code, or any MCP host) to SWIRL via the SWIRL MCP
  server, so agents can run federated searches, read documents, and get
  grounded RAG answers over the organization's sources. /swirl:mcp
user_invocable: true
---

# SWIRL MCP Hookup

SWIRL Enterprise ships an MCP server (`swirl_mcp`) that exposes federated
search to any MCP host. It's a standalone process that calls a running SWIRL
deployment over HTTP — all permissions, licensing, and workspace scoping are
enforced by SWIRL exactly as for any API client.

## Tools the agent gets

`search`, `get_search_results`, `list_providers`, `search_rag` (grounded
answer + citations), `chat` (SWIRL Assistant, needs the `chat_api` license
feature), `read_document` (windowed full text of docs from *your own*
results — arbitrary URLs are refused), `score_document`. Read-only resources
(`swirl://providers`, `swirl://health`, `swirl://license`, ...) are
secret-scrubbed by field whitelists — credentials can never appear.

## Setup — Mode A (static token; laptops and PoCs)

1. On the SWIRL host, mint a DRF token for the user the agent should act as
   (the agent inherits exactly that user's permissions and source access):
   ```bash
   python manage.py drf_create_token <username>
   ```
2. Register with Claude Code (stdio, token via environment — never
   hardcoded, never in args):
   ```bash
   claude mcp add swirl \
     -e SWIRL_MCP_TOKEN=<api-key> \
     -e SWIRL_MCP_BASE_URL=http://localhost:8000 \
     -- python -m swirl_mcp
   ```
   For Claude Desktop, the equivalent `mcpServers` entry goes in its config
   with the same command and `env` block.
3. Verify: in a fresh session, ask the agent to `list_providers`, then run a
   search whose results you can confirm in the Galaxy UI.

Key environment variables (flag > env > default): `SWIRL_MCP_TOKEN`
(required in static mode), `SWIRL_MCP_BASE_URL` (default
`http://localhost:8000`), `SWIRL_MCP_TRANSPORT` (`stdio` default, or
`http`), `SWIRL_MCP_PORT` (default 8675), `SWIRL_MCP_RAG_POLL_TIMEOUT`
(default 90s — raise for slow local models).

## Setup — Mode B (OAuth 2.1 resource server; multi-user production)

For shared deployments, run the server with `SWIRL_MCP_AUTH=oidc` over HTTP.
The MCP host runs PKCE against the customer IdP; the server validates each
bearer JWT (issuer, audience, JWKS signature) and forwards it to SWIRL,
which maps it to the real calling user — per-user permission trimming
applies to each caller. Server-side requirements: an active Authenticator
with `issuer` and `jwks_uri` set, accepted audiences configured
(`SWIRL_OIDC_API_AUDIENCE`), and an IdP audience mapper. Never deploy
static-token mode multi-user: everyone would act as one SWIRL user.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Every tool fails with a license message | SWIRL license missing/invalid (SAPI answers 402). Check `swirl://license`. |
| `chat` refuses | License lacks the `chat_api` feature. |
| Tools time out on RAG | Cold local model; raise `SWIRL_MCP_RAG_POLL_TIMEOUT`. |
| Connection refused | `SWIRL_MCP_BASE_URL` wrong from the server's vantage point (containers: `host.docker.internal`, not `localhost`). |
| Agent sees no providers | The token's user has no shared/owned providers — fix SWIRL-side access, not the MCP config. |
