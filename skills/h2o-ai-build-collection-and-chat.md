---
name: Build an h2oGPTe Collection and chat over it
description: Create an Enterprise h2oGPTe Collection, ingest documents into it, open a chat session scoped to that Collection, and ask grounded questions.
api: openapi/h2o-ai-h2ogpte-openapi-original.yml
base_url: https://h2ogpte.genai.h2o.ai/api/v1
operations:
  - create_collection
  - list_collections
  - get_collection
  - upload_file
  - ingest_upload
  - list_documents_for_collection
  - create_chat_session
  - get_completion
  - get_chat_session
  - delete_collection
generated: '2026-08-04'
method: generated
source: openapi/h2o-ai-h2ogpte-openapi-original.yml + conventions/h2o-ai-conventions.yml
---

# Build an h2oGPTe Collection and chat over it

This is the core RAG flow for Enterprise h2oGPTe: a Collection holds documents, a chat
session is bound to a Collection, and completions are answered against that Collection's
indexed content.

## Authentication

Every request carries a bearer API key:

```
Authorization: Bearer sk-...
```

Two key types exist and the choice matters for this flow:

- A **global** key can create and delete Collections. Use it for the setup steps.
- A **Collection-specific** key can only chat with the one Collection it was created
  against. Use it for the chat step when handing credentials to a narrower agent.

Keys are created in the h2oGPTe UI (Account Circle → Using the API → + New API Key). See
<https://docs.h2o.ai/enterprise-h2ogpte/guide/apis>.

## Steps

1. **Check whether the Collection already exists — `list_collections`**
   (`GET /collections`). There is no idempotency key on this API, so calling
   `create_collection` twice creates two Collections. Always list first and match on name.
   `list_collections` takes `offset` and `limit`; `get_collection_count`
   (`GET /collections/count`) returns the total if you need to page.

2. **Create the Collection — `create_collection`** (`POST /collections`).
   Body carries `name`, `description` and `embedding_model`. Pick the embedding model with
   `list_embedding_models` (`GET /embedding_models`) or take the default from
   `get_default_embedding_model` (`GET /embedding_models/default`) rather than hard-coding
   a model name — the available set changes between releases.
   Keep the returned Collection id.

3. **Upload the file — `upload_file`** (`PUT /uploads`).
   This is a `multipart/form-data` request with the file bytes and an optional `mtime`.
   It returns an upload id. Nothing is indexed yet.

4. **Ingest the upload into the Collection — `ingest_upload`**
   (`POST /uploads/{upload_ids}/ingest`). Pass the Collection id. This is the *synchronous*
   form and it blocks until indexing finishes. For anything large, use the asynchronous
   sibling instead — see the ingest-at-scale skill.

   Other sources are available as first-class operations rather than a generic connector:
   `ingest_from_website`, `ingest_from_plain_text`, `ingest_from_s3`, `ingest_from_gcs`,
   `ingest_from_azure_blob_storage`, `ingest_from_confluence`, `ingest_from_sharepoint_online`.

5. **Confirm indexing — `list_documents_for_collection`**
   (`GET /collections/{collection_id}/documents`). Do not proceed to chat until the document
   you ingested appears here; asking before indexing completes returns an ungrounded answer,
   not an error.

6. **Open a chat session — `create_chat_session`** (`POST /chats`), passing the Collection id.
   Keep the returned session id. `get_chat_session` (`GET /chats/{session_id}`) reads it back;
   `list_chat_sessions` (`GET /chats`) enumerates.

7. **Ask the question — `get_completion`** (`POST /chats/{session_id}/completions`).
   Set `stream` to control delivery: when streaming is enabled the server sends a stream of
   delta messages over `text/event-stream`. If the LLM connection drops mid-stream you get the
   `ChatError` envelope — `{"error": "The communication with llm has been interrupted"}` — not
   an HTTP error code. Handle both.

8. **Clean up when the Collection was temporary — `delete_collection`**
   (`DELETE /collections/{collection_id}`). For a large Collection prefer the job form,
   `create_delete_collection_job`, so the call does not block.

## Rules

- **No idempotency.** There is no `Idempotency-Key` on any of the 422 operations. Every
  create step in this skill must be guarded by a list-and-match, or you will accumulate
  duplicates on retry.
- **Errors are a vendor envelope**, not RFC 9457. Non-2xx responses return
  `{"code": <int>, "message": "<string>"}`. Do not look for `type`/`title`/`detail`.
  401 is declared on every operation; 400, 404, 409 and 500 are common.
- **Pagination is `offset` + `limit`** and list responses are bare JSON arrays with no
  envelope. Totals come from the sibling `*_count` operations.
- **Rate limiting exists but is undocumented.** h2oGPTe 1.7.0 added per-user API rate
  limiting and automatic key deactivation, but no limit values, header names or `Retry-After`
  contract are published. Back off on repeated failures rather than trusting a header.
- **Everything is live.** There is no test mode and no sandbox host. Scope the key to one
  Collection when you want a blast radius.
