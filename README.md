# H2O.ai

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

H2O.ai is an open-source artificial-intelligence and machine-learning company. Its platform spans
H2O-3 (a distributed, in-memory ML engine), H2O Driverless AI (automatic machine learning),
H2O MLOps (model deployment, scoring and monitoring), H2O Wave (a Python/R framework for realtime
AI apps), H2O LLM Studio, and Enterprise h2oGPTe (a private generative-AI, RAG and agent platform).

- Website — https://h2o.ai/
- Documentation — https://docs.h2o.ai/
- GitHub — https://github.com/h2oai
- Status — https://h2oai.statuspage.io/
- Trust Center — https://trust.h2o.ai/

## APIs profiled

| API | Spec | Operations |
|---|---|---|
| Enterprise h2oGPTe REST API | OpenAPI 3.0.1 | 422 across 24 tags |
| H2O MLOps Scoring REST API | OpenAPI 3.0.0 | 7 |
| h2oGPTe MCP Server | first-party MCP (stdio) | one tool per REST operationId |

Both specifications were harvested verbatim from H2O.ai's own hosts —
`https://h2ogpte.genai.h2o.ai/api-spec.yaml` and the YAML linked from the MLOps scoring docs.

## What H2O.ai does not publish

Recorded here because the absence is the finding, not an omission on our side:

- No `Idempotency-Key` on any of the 422 h2oGPTe operations.
- Errors use a vendor `{code, message}` envelope, not RFC 9457 problem details.
- Rate limiting ships in h2oGPTe 1.7.0 but no limit values, header names or `Retry-After` contract.
- No deprecation or version-support policy and no `Sunset`/`Deprecation` headers, across five
  major product versions.
- No webhooks and no AsyncAPI — long-running work is job-polling only.
- No `/.well-known/security.txt`, no API catalog, and no A2A agent card on any host.
- No test-mode key and no sandbox host; the interactive Swagger UI hits the live tenant.

Harvest provenance: surfaced via the API Evangelist harvest backlog (source: secondary-market,
https://forgeglobal.com/h2o-ai_stock/) and profiled by the enrichment pipeline on 2026-08-04.
