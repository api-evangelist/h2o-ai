---
name: Ingest documents into h2oGPTe asynchronously with jobs
description: Start a long-running h2oGPTe ingestion as a job, poll it to completion, and reconcile the resulting documents — the correct pattern for bulk or slow sources.
api: openapi/h2o-ai-h2ogpte-openapi-original.yml
base_url: https://h2ogpte.genai.h2o.ai/api/v1
operations:
  - upload_file
  - create_ingest_upload_job
  - create_ingest_from_s3_job
  - create_ingest_from_website_job
  - get_job
  - list_jobs
  - list_documents_for_collection
  - get_document
  - delete_document_from_collection
generated: '2026-08-04'
method: generated
source: openapi/h2o-ai-h2ogpte-openapi-original.yml + conventions/h2o-ai-conventions.yml
---

# Ingest documents into h2oGPTe asynchronously with jobs

Enterprise h2oGPTe exposes every ingestion source twice: a synchronous operation that blocks
until indexing finishes, and a `create_*_job` sibling that returns immediately with a job
handle. For anything beyond a single small file, use the job form — the synchronous call will
time out at the ingress long before a large corpus finishes indexing.

There is **no webhook and no callback**. Completion is discovered by polling. That is the
whole reason this skill exists.

## The job-form operations

| Source | Synchronous | Job form |
|---|---|---|
| Uploaded file | `ingest_upload` | `create_ingest_upload_job` |
| Public website crawl | `ingest_from_website` | `create_ingest_from_website_job` |
| Plain text | `ingest_from_plain_text` | `create_ingest_from_plain_text_job` |
| AWS S3 | `ingest_from_s3` | `create_ingest_from_s3_job` |
| Google Cloud Storage | `ingest_from_gcs` | `create_ingest_from_gcs_job` |
| Azure Blob Storage | `ingest_from_azure_blob_storage` | `create_ingest_from_azure_blob_storage_job` |
| Confluence | `ingest_from_confluence` | `create_ingest_from_confluence_job` |
| SharePoint Online | `ingest_from_sharepoint_online` | `create_ingest_from_sharepoint_online_job` |
| Local file system | `ingest_from_file_system` | `create_ingest_from_file_system_job` |
| Agent-only → standard | `ingest_agent_only_to_standard` | `create_ingest_agent_only_to_standard_job` |

## Steps

1. **Upload first when the source is a file — `upload_file`** (`PUT /uploads`), multipart.
   Collect the upload ids. Uploading is separate from ingesting; an upload that is never
   ingested is indexed nowhere.

2. **Start the job — `create_ingest_upload_job`**
   (`POST /uploads/{upload_ids}/ingest/job`), passing the target Collection id. Multiple
   upload ids can be ingested in one job. For a cloud source use the matching
   `create_ingest_from_*_job` operation instead.
   Record the returned job id.

3. **Poll — `get_job`** (`GET /jobs/{job_id}`). Poll with backoff; start around two seconds
   and grow. Do not poll in a tight loop: rate limiting is enforced but its thresholds are
   not published, so a hot loop can get the whole key throttled or deactivated.
   `list_jobs` (`GET /jobs`) enumerates every job for the user if you lost the id.

4. **Reconcile — `list_documents_for_collection`**
   (`GET /collections/{collection_id}/documents`). A job reporting done is not proof that
   every source document landed; compare the returned documents against what you submitted.
   `get_document` (`GET /documents/{document_id}`) reads one back.

5. **Repair, do not re-run.** If a document is missing, ingest that document alone rather
   than re-running the whole job. There is no idempotency key, so re-running a completed
   ingestion job duplicates every document that *did* succeed.
   Remove a duplicate with `delete_document_from_collection`
   (`DELETE /collections/{collection_id}/documents/{document_id}`).

## Rules

- **Poll, never wait on a webhook.** h2oGPTe publishes no AsyncAPI, no webhook catalog and no
  event surface of any kind. Polling `get_job` is the only completion signal.
- **The default MCP tool set hides these operations.** The first-party
  `h2ogpte-mcp-server` defaults to `H2OGPTE_ENDPOINT_SET=all_without_async_ingest`, which
  removes exactly the ten `create_ingest_*_job` operations. An agent that needs this skill
  must be configured with `H2OGPTE_ENDPOINT_SET=all`. See `mcp/h2o-ai-tool-crosswalk.yml`.
- **Retries duplicate.** No `Idempotency-Key` exists on this API. Every retry decision in
  this flow has to be made against observed state, not against a replay key.
- **Errors are `{"code": <int>, "message": "<string>"}`**, not RFC 9457 problem details.
