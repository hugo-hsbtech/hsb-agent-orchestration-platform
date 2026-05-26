# J4 — Knowledge Source + First Ingestion

> **Entry:** `Add source` on KB Management (`68:56`, button `68:284`). **Figma worked example:** add `acme-helpcenter` web crawl (ties to the `kb_lookup` breaker thread) — frames `223:62` → `227:62`. **Grounds:** [AD-18](../../01-PRD.md) (Qdrant KB exposed as a registered tool), [33-data-pipeline-kb-management](../../deep-dive/33-data-pipeline-kb-management.md) (connectors, pipeline stages, chunking/embedding/sync), [v5-chatbot-layer](../../roadmap/v5-chatbot-layer.md) (KB integration + filter validation), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace-scoped collections), [09-security-model](../../deep-dive/09-security-model.md) (credential refs).

The KB is Qdrant ([AD-18](../../01-PRD.md)); a source connects an external corpus into a namespace-scoped collection. The 5-step wizard maps onto the **ingestion pipeline stages** ([33](../../deep-dive/33-data-pipeline-kb-management.md) §Data Pipeline Stages: Ingestion → Processing → Embedding → Indexing → Validation): **Source → Configure → Test/preview → Ingest → Indexed**.

---

## Step 1 · Source — connector branch

Six connector cards; each requires distinct config in Step 2.

| Connector | What it pulls | Distinct config it needs | Sync model |
|---|---|---|---|
| **Web crawl** | site pages via sitemap/crawl | Start URL; respects robots.txt; depth/page limits | scheduled (e.g. daily) |
| **S3** | objects in a bucket | bucket + prefix/path; AWS credentials/role; region | scheduled or storage-event (S3 file uploaded) |
| **Google Drive** | Drive files/folders | OAuth to Drive; folder ID(s); file-type filter | API poll (hourly) |
| **Notion** | Notion pages/databases | Notion integration token; workspace/database IDs | API poll (hourly) |
| **File upload** | direct PDF/Word/Markdown/TXT/HTML | the uploaded file(s) | on upload (one-shot) |
| **REST API** | structured data via HTTP | base URL; auth ref; request/response mapping | scheduled |

- **Necessary actions per connector:**
  - **Web crawl:** the start URL must be reachable and the host's robots.txt must permit crawling.
  - **S3 / Google Drive / Notion / API:** a stored **credential/auth ref** with read permission to the source must exist (J6) — e.g. AWS keys/role, Drive OAuth, Notion integration token, API key. The platform stores the auth as a reference (`auth_ref`), never inline ([33](../../deep-dive/33-data-pipeline-kb-management.md) example config; [09-security-model](../../deep-dive/09-security-model.md) §least privilege — "Chatbots access only their designated knowledge base filters").
  - **Upload:** no external network/creds; the file is the corpus.
  - All connectors need **network access** from the ingestion workers to the source.

## Step 2 · Configure

Per the chosen connector, plus the shared processing config:

| Field | Branch / values | Notes |
|---|---|---|
| Source-specific | Start URL (web) / bucket+prefix (S3) / folder IDs (Drive) / DB IDs (Notion) / file (upload) / endpoint (API) | from Step 1's branch |
| **Target collection** | namespace-scoped Qdrant collection (dropdown, e.g. `acme`) | collections are per-namespace ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md); [33](../../deep-dive/33-data-pipeline-kb-management.md) §Indexing) |
| **Chunking** | strategy + size + overlap, e.g. `recursive · 800 tok · 80 overlap`. Strategies: fixed size / semantic (paragraph) / hierarchical (parent-child) / sliding-window | [33](../../deep-dive/33-data-pipeline-kb-management.md) §Story 1.2 |
| **Embedding model** | dropdown (e.g. `text-embedding-3-large`) | must match the model used by the chatbots/tools querying this collection, else vectors are incompatible |
| **Sync schedule** | branch on connector: `on upload` (upload) / `real-time` (webhook/CDC) / `5-min poll` / `hourly` / `daily HH:MM UTC` / `weekly` / `manual`; plus `full_refresh` cadence | [33](../../deep-dive/33-data-pipeline-kb-management.md) §Sync Schedule Options |
| Filters *(source-dependent)* | spaces/labels (Confluence/Notion), prefixes (S3), include/exclude | scope what's ingested |

- **Necessary actions:** the **embedding model must be available** (provider/local) and **consistent** across the collection — mixing embedding models in one collection breaks retrieval. The target collection is created per namespace; hybrid (dense+sparse) search must be enabled if the querying agents use `retrieval_strategy: hybrid`.
- **Validation:** Qdrant filter definitions referenced later by specialists are validated against the Qdrant schema at chatbot provisioning ([v5](../../roadmap/v5-chatbot-layer.md) Story 3.2) — a source whose metadata doesn't support a specialist's declared filters will silently return nothing.

## Step 3 · Test & preview

Connection test + sample documents preview (e.g. `success` chip "Reached help.acme.com — 1,204 pages discovered" + 3 sample doc rows).

- **Necessary action:** proves credentials/network/permissions all hold before a full ingest.
- **Failure:** auth failure, unreachable source, or empty discovery → `error` chip; user returns to Configure/Source. (`exclude_labels`/filters too aggressive can produce a zero-document preview.)

## Step 4 · Ingesting (running state)

Pipeline progress as **Progress-step rows** with counts, mapping to the pipeline stages:

| Stage | Row example | What happens |
|---|---|---|
| **Fetch** (Ingestion) | done · `1,204 pages` | collect raw documents from the source |
| **Chunk** (Processing) | done · `9,841 chunks` | extract text, clean boilerplate, chunk per strategy, extract metadata |
| **Embed** (Embedding) | active · `6,002 / 9,841` | generate vectors (batched; dedup unchanged content) |
| **Index** (Indexing) | pending | write points into the namespace Qdrant collection |

Status bar shows overall % (e.g. "Embedding in progress · 61%"); the running state offers **Cancel only**.

- **Necessary actions:** embedding generation consumes provider tokens/quota (or local compute); large corpora are batch-processed; incremental sync re-embeds only changed content ([33](../../deep-dive/33-data-pipeline-kb-management.md) §Story 1.3). Failed documents are logged for review (human review queue), not silently dropped.
- **Failure:** a stage failure (extraction error, embedding rate-limit, Qdrant write error) halts/retries with backoff; failed syncs are alertable and retryable. A KB tool failure later trips the `kb_lookup` circuit breaker (Health `65:56` → degradation banner Chat `54:56` → `Error` source on KB `68:56`).

## Step 5 · Indexed ✓

Success chip + stats: pages indexed, chunks created, collection, freshness. Primary: **Done** → the source appears `Healthy` in the KB list.

- **What this enables:** the collection is now queryable via the registered KB tool (`knowledge_base_lookup`) and attachable to chatbots in J1 Step 5c (`knowledge_base.collection` + filters). Freshness/coverage feed KB health monitoring and analytics (KB hit rate, Analytics `59:56`).

---

## Pipeline stages ↔ wizard steps (quick reference)

| Pipeline stage ([33](../../deep-dive/33-data-pipeline-kb-management.md)) | Wizard step |
|---|---|
| Ingestion (collect) | Source + Configure + Fetch |
| Processing (extract/clean/chunk) | Chunk |
| Embedding (vectorize) | Embed |
| Indexing (store in Qdrant) | Index |
| Validation (verify quality) | Indexed ✓ + ongoing health monitoring |

## Failure-state summary

| Step | Failure | Fix |
|---|---|---|
| Source/Configure | missing/insufficient credential, no network access | create + scope a credential (J6); open network to the source |
| Configure | embedding model unavailable / inconsistent with collection | pick an available model; keep one model per collection |
| Test | unreachable source / zero documents / robots.txt blocks crawl | fix URL/auth; relax filters; respect robots policy |
| Ingest | extraction/embedding/index failure | inspect failed-doc log; retry; watch embedding quota/rate limits |
| Retrieval (later) | specialist filters don't match source metadata | align Qdrant filters with the ingested metadata |

## Cross-references

- Decisions: [AD-18](../../01-PRD.md).
- Deep-dives: [33-data-pipeline-kb-management](../../deep-dive/33-data-pipeline-kb-management.md) (connectors, pipeline, chunking/embedding/sync, governance), [v5-chatbot-layer](../../roadmap/v5-chatbot-layer.md) (KB tool + filter validation), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (per-namespace collections), [22-chatbot-degradation](../../deep-dive/22-chatbot-degradation.md) (KB circuit breaker).
- Figma (worked example): `223:62` (Source) · `224:62` (Configure) · `225:62` (Test) · `226:62` (Ingesting) · `227:62` (Indexed). Parent screen: KB Management `68:56`. Story thread: Health `65:56` → Chat `54:56`.
