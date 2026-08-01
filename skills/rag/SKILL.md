---
name: rag
description: |
  Set up AI providers and RAG in SWIRL: connect an LLM (OpenAI, Anthropic,
  Azure, Ollama, ...), assign roles, tune grounded question-answering over
  federated results, and configure the ranking models (embedding reader,
  cross-encoder). /swirl:rag
user_invocable: true
---

# AI Providers and RAG

SWIRL's RAG pipeline: federate the search, re-rank with embeddings, select
and deduplicate the best passages under a token budget, then make a single
LLM call that returns a grounded answer with citations. One call - not a
chain - so token cost stays flat as sources grow.

**Ask which edition first - configuration differs:**

- **Community**: bring your own OpenAI key via the environment - 
  `OPENAI_API_KEY=<api-key>` in the shell or the compose file's
  `environment:` block - then restart the stack. That's the whole setup;
  the "AI provider model" section below is Enterprise-only. Verification
  and the retrieval-side tuning advice apply to both editions.
- **Enterprise**: multiple configurable AI providers (OpenAI, Anthropic,
  Azure, Ollama, and more) with per-role assignment, managed as data - 
  continue below.

## AI provider model (Enterprise)

AI providers are configured objects (admin UI → AI Providers, or
`/swirl/aiproviders/` API) with:

- `api_key` - never print it; placeholder `<api-key>`.
- `model` - e.g. `gpt-4o`, `claude-sonnet-5`, a local Ollama model tag.
- `config` - endpoint URL, API version, token limits.
- `tags` / `defaults` - which provider serves which role by default.

Roles (a provider can hold several):

| Role | Used for |
|---|---|
| `rag` | The answer-generating LLM call |
| `reader` | Embedding re-ranking of results |
| `query` | Query understanding/rewriting |
| `connector` | GenAI-as-a-source providers |

Assume one active default per role. Activating/deactivating a provider is a
runtime behavior change - flag it as such.

## Setup steps

1. Pick the LLM. Cloud (OpenAI/Anthropic/Azure) needs an `<api-key>`;
   local Ollama needs the model pulled and the endpoint reachable from the
   SWIRL containers (on Docker for Mac that's `host.docker.internal`, not
   `localhost`). Note: some client libraries require the `api_key` field to
   be non-empty even for local Ollama - use any placeholder string.
2. Create/activate the provider with role `rag` and confirm it's the
   default for that role.
3. Verify model + token limit compatibility: the configured context limit
   must exceed SWIRL's RAG passage budget plus prompt overhead, or answers
   silently truncate.
4. Test end-to-end with a question whose answer you can verify in the
   sources:
   ```bash
   curl -s "http://localhost:8000/swirl/sapi/search/?q=<question>&rag=true" \
     -H "Authorization: Token <api-key>"
   ```
   The `ai_summary` arrives asynchronously - poll the result; local models
   can take 60–90s on a cold load.

## Ranking models: cross-encoder and embeddings (Enterprise)

When the user asks about the cross-encoder, embedding models, the
re-ranker, or "replacing spaCy": **do not go hunting through settings
files, environment variables, or Python source** - ranking models are
configured as AI Providers, the same surface as everything else in this
skill. Enterprise images ship compiled code; the admin-facing
configuration surfaces are the AI Providers (in the database) and the
`.env` file, and ranking lives in AI Providers.

What to know before touching anything:

- Relevancy in SWIRL 5 Enterprise is three passes: keyword/BM25, embedding
  re-ranking (the `reader` role), then a **cross-encoder** that reads
  query and document together. The cross-encoder ships **on by default**
  as the final pass - there is nothing to enable. Confirm it rather than
  configure it: a result's explain output shows cross-encoder scores per
  result.
- The **embedding model is whatever AI Provider holds the `reader` role.**
  The default reader is spaCy: small, CPU-cheap, fine for getting started.
  To use a larger model, activate (or create) a reader-role AI Provider
  pointing at it and make it the reader default, deactivating the spaCy
  reader. The Enterprise compose stack's Ollama sidecar can serve
  `mxbai-embed-large` (1024 dimensions) for exactly this - no new
  infrastructure needed.
- **AI Provider changes take effect immediately at query time.** No
  restart, no redeploy.

Verify the swap with a live search, not by re-reading config: the
per-result explain output shows the ranking passes that actually ran, and
the Ollama container's logs show embedding calls landing during the query.
Larger embeddings cost latency per result - if search gets noticeably
slower, that's the trade the user chose; say so rather than debugging it.

## Tuning knobs (change one at a time, re-test)

- **Source scope**: RAG quality tracks retrieval quality. Scope the search
  to providers that actually contain answers before touching the LLM side.
- **Token budget / passage count**: more passages ≠ better answers; the
  default (~3000 tokens of curated passages) beats stuffing the context.
- **Prompts**: system/RAG prompts are configurable per deployment (and
  overridable per model family). Keep the grounding instruction - "answer
  only from the provided sources, cite them" - intact in any custom prompt.
- **Timeouts**: slow local models need the RAG poll timeout raised, not the
  prompt shortened.

## Verification checklist

- Answer cites real result URLs (spot-check one citation against the doc).
- A question with no answer in the sources yields "not found in sources",
  not a hallucination.
- Latency acceptable (cloud: seconds; local 70B-class: a minute+ - set
  expectations).
