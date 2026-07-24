---
name: rag
description: |
  Set up AI providers and RAG in SWIRL: connect an LLM (OpenAI, Anthropic,
  Azure, Ollama, ...), assign roles, and tune grounded question-answering
  over federated results. /swirl:rag
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
