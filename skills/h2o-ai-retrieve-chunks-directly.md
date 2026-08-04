---
name: Retrieve h2oGPTe chunks directly for your own RAG loop
description: Use lexical and semantic chunk search against an h2oGPTe Collection to fetch grounding passages, when you want to run generation yourself instead of using the chat completion endpoint.
api: openapi/h2o-ai-h2ogpte-openapi-original.yml
base_url: https://h2ogpte.genai.h2o.ai/api/v1
operations:
  - list_collections
  - search_collection_chunks
  - match_collection_chunks
  - get_collection_chunks
  - list_models
  - answer_question
  - summarize_content
generated: '2026-08-04'
method: generated
source: openapi/h2o-ai-h2ogpte-openapi-original.yml + conventions/h2o-ai-conventions.yml
---

# Retrieve h2oGPTe chunks directly for your own RAG loop

Enterprise h2oGPTe will do the whole RAG loop for you through `get_completion`. But it also
exposes the retrieval layer on its own, which is what you want when the generation step lives
in your own agent, in another model, or in a framework you already run.

## Steps

1. **Resolve the Collection — `list_collections`** (`GET /collections`) and match on name,
   or use a Collection id you already hold.

2. **Retrieve grounding passages.** Two retrieval modes are exposed as separate operations,
   and they are not interchangeable:

   - **`match_collection_chunks`** (`POST /collections/{collection_id}/chunks/match`) —
     *semantic* search. Finds chunks related to a message by embedding similarity. This is
     the default choice for natural-language questions.
   - **`search_collection_chunks`** (`POST /collections/{collection_id}/chunks/search`) —
     *lexical* search. Finds chunks by term match. Use it for identifiers, error codes,
     part numbers and exact phrases, where embeddings under-perform.

   Run both and merge when recall matters more than latency.

3. **Fetch specific chunks by id — `get_collection_chunks`**
   (`GET /collections/{collection_id}/chunks/{chunk_ids}`) when you already know which chunks
   you want — for example to expand context around a hit from step 2, or to re-fetch the exact
   passages cited in an earlier answer.

4. **Generate however you like.** If you want to stay inside h2oGPTe without opening a chat
   session, two stateless model operations take content directly:

   - **`answer_question`** (`POST /models/{model_name}/answer_question`) — send a message and
     get a response from a named LLM.
   - **`summarize_content`** (`POST /models/{model_name}/summarize_content`) — summarize one
     or more contexts.

   Resolve `model_name` from `list_models` (`GET /models`) rather than hard-coding it. The
   available model set changes with almost every release — 1.6.54 added GPT-5.2 and
   Gemini 3 Pro Preview, 1.6.57 added Gemini 3.1 Pro Preview and Claude 4.6 — so a hard-coded
   name is a time bomb.

## Rules

- **These are read operations and safe to retry.** Unlike the create/ingest flows, chunk
  search has no duplication hazard, so the missing idempotency contract does not bite here.
- **Errors are `{"code": <int>, "message": "<string>"}`.** 401 is declared on every
  operation; 400 and 404 are the common failures (bad Collection id, malformed body).
- **Pagination is `offset` + `limit`** where the operation accepts it; responses are bare
  arrays.
- **A Collection-specific API key is enough for this skill.** It cannot create or delete
  Collections, which is exactly right for a retrieval-only agent. Prefer it over a global key.
- **Back off on failure.** Rate limits are enforced (h2oGPTe 1.7.0) but no thresholds or
  headers are published, so treat repeated 4xx/5xx as a signal to slow down.
