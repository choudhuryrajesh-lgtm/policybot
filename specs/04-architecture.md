# 04 — Architecture

## Status
Draft v0.3 — 2026-08-21 (v0.2 added the MCP Tool Layer and Social
Service, §2.10/§2.11; v0.3 adds the Placeholder People-Data Service,
§2.12, resolving ADR-006's "what does it integrate with" question)

Traces from: [00-vision-and-scope.md](00-vision-and-scope.md),
[02-functional-requirements.md](02-functional-requirements.md),
[03-non-functional-requirements.md](03-non-functional-requirements.md)

---

## 1. System Context

```
                         ┌───────────────────────┐
   Employee (browser) ───▶  React SPA (CloudFront +  │
                         │  S3, independently deployed)│
                         └───────────┬───────────┘
                                     │ HTTPS (REST + SSE streaming)
                                     │ Cognito-issued JWT
                                     ▼
                         ┌───────────────────────┐
                         │   API Gateway (HTTP API)│
                         │   + Cognito Authorizer  │
                         └───────────┬───────────┘
                                     ▼
                         ┌───────────────────────┐
                         │   Chat Backend Service  │
                         │  (ECS Fargate) — hosts   │
                         │  the LangGraph agent    │
                         └──┬──────┬──────┬───────┘
                            │      │      │
              ┌─────────────┘      │      └─────────────┐
              ▼                    ▼                    ▼
  ┌────────────────────┐ ┌─────────────────┐ ┌────────────────────┐
  │ Postgres + pgvector │ │     DynamoDB     │ │   OpenAI API        │
  │ (Aurora, HNSW/ANN + │ │ sessions, msgs,  │ │ (chat + embeddings) │
  │ BM25 hybrid search) │ │ long-term memory,│ │  behind provider    │
  │                     │ │ doc metadata,    │ │  abstraction layer  │
  │                     │ │ audit log        │ └────────────────────┘
  └─────────▲───────────┘ └────────▲────────┘
            │                      │
            │ writes               │ writes
            │                      │
  ┌─────────┴──────────────────────┴───────────┐
  │            Ingestion Pipeline                │
  │  S3 upload/sync ──▶ EventBridge/S3 Event      │
  │  ──▶ Step Functions ──▶ workers (Lambda/ECS)  │
  │  (extract → OCR/caption/transcribe → chunk    │
  │   → hash/dedup → embed → index)               │
  └────────────────────────────────────────────┘
            ▲
            │
  ┌─────────┴───────────┐      ┌──────────────────────┐
  │  Content owners /     │      │  Legacy doc source    │
  │  Admin UI (upload)    │      │  (SharePoint/Confl./  │
  │                       │      │  Drive) → sync job     │
  └───────────────────────┘      │  → same S3 landing zone│
                                  └──────────────────────┘

  Cross-cutting: New Relic (APM, dashboards, alerts) instruments API,
  backend, ingestion workers. LangSmith receives traces from every
  LangGraph node for tracing + eval. Guardrail checks run inline in the
  backend service (input + output).
```

### 1.1 v0.2/v0.3 Additions — Org/People Data & Social Layer

Three new components, hung off the existing Chat Backend Service rather
than replacing anything in the diagram above:

```
                         ┌───────────────────────┐
   Chat Backend Service ─┤   MCP Tool Layer        │──▶ Placeholder
   (LangGraph agent,     │  (org/people lookups,   │     People-Data Service
    per §2.3)            │   read-only, rate-      │     (§2.12 — our own ECS
                         │   limited + cached)     │      service, mock data;
                         └───────────────────────┘      real HRIS is future work)

                         ┌───────────────────────┐
   Chat Backend Service ─┤   Social Service         │──▶ DynamoDB (Profiles,
   (profile/feed/DM API) │  (profiles, feed, posts,│     Posts, Comments, Feed,
                         │   comments, DMs,        │     Follows, DirectMessages,
                         │   moderation)           │     ModerationReports —
                         └───────────────────────┘     see 05-domain-model §3.1)
```

The MCP Tool Layer is called *from within* the LangGraph graph (a tool
the `generate` node can invoke — see
[08-generation-spec.md](08-generation-spec.md) §8), not a user-facing
API. The Social Service is a user-facing API the frontend calls directly
(profile views, feed, posting, messaging) — it does not run through the
LangGraph graph at all, since it's CRUD on native application data, not a
question-answering flow.

**Resolved v0.3** ([00-vision-and-scope.md](00-vision-and-scope.md) §8
Q7): the MCP Tool Layer's upstream in v1 is not an external vendor HRIS —
it's the Placeholder People-Data Service (§2.12), a service this team
owns and deploys. The MCP Tool Layer's interface to it is written
identically to how a real HRIS integration would look, so the swap later
is a config/endpoint change behind that interface, not a rewrite of the
MCP Tool Layer or anything upstream of it (generation graph, FRs).

## 2. Components

### 2.1 Frontend — React SPA
- Deployed to S3, served via CloudFront (independent pipeline from
  backend, per NFR-062).
- Authenticates via Cognito Hosted UI / Amplify Auth (OIDC), holds JWT,
  attaches as Bearer token to API calls.
- Multi-chat-window UI: session list + active session panes, SSE-based
  streaming render.
- Calls: `POST /sessions`, `GET /sessions`, `POST
  /sessions/{id}/messages` (streaming), `DELETE /sessions/{id}`,
  `GET/DELETE /memory`, admin ingestion endpoints (role-gated).

### 2.2 API Gateway + Auth
- API Gateway (HTTP API) fronting the backend; Cognito JWT authorizer
  validates tokens and forwards claims (user ID, groups) to the backend.
- Rate limiting / WAF at this layer for basic abuse protection (also see
  security spec for prompt-injection-specific controls, which are
  application-layer, not WAF-layer).

### 2.3 Chat Backend Service (LangGraph host)
- Runs on ECS Fargate (chosen over Lambda for this component because
  LangGraph sessions can be long-running/streaming and benefit from a
  persistent process — revisit in ADR if Lambda + response streaming
  proves sufficient and cheaper at low launch scale).
- Hosts the LangGraph graph (see
  [08-generation-spec.md](08-generation-spec.md)) — query
  rewrite → retrieve (hybrid) → grade → generate → guardrail/verify.
- Talks to: Postgres/pgvector (retrieval), DynamoDB (session/message/
  memory read-write), OpenAI (via abstraction layer), LangSmith
  (tracing).
- Stateless per-request beyond what's persisted to DynamoDB/Postgres —
  horizontally scalable (NFR-020).

### 2.4 Ingestion Pipeline
- **Landing zone**: single S3 bucket (prefixed by source: `/uploads/*`
  for admin-UI uploads, `/sync/*` for legacy-repo sync jobs) — one
  ingestion path regardless of origin, per resolved open question 5.
- **Trigger**: S3 event → EventBridge → Step Functions state machine.
- **Stages** (Step Functions, workers as Lambda for text/small files,
  ECS/Batch for heavier video transcription/vision jobs):
  1. Classify file type.
  2. Extract raw content (text extraction / OCR for images / ASR +
     keyframe extraction for video) — see
     [06-ingestion-spec.md](06-ingestion-spec.md).
  3. Compute document + chunk content hashes; check against DynamoDB
     document-metadata table for dedup/diff (FR-041–043).
  4. Context-aware chunk (only changed regions on re-upload).
  5. Embed via OpenAI embeddings API.
  6. Upsert chunks + embeddings into Postgres/pgvector; upsert
     doc/chunk metadata + status into DynamoDB.
  7. Emit ingestion status events (consumed by admin UI for status
     display, FR-091).
- Failure isolation per document (NFR-013) — Step Functions per-document
  execution, failures don't block the batch.

### 2.5 Postgres + pgvector (Aurora PostgreSQL)
- Stores: chunks table (text, metadata, doc/version FK), embeddings
  column (`vector` type), HNSW index for ANN, full-text (`tsvector`)
  column + GIN index for BM25-style sparse search.
- Aurora chosen over vanilla RDS Postgres for multi-AZ durability +
  easier storage autoscaling as corpus grows toward the ~1000x design
  ceiling (NFR-022); revisit instance sizing once real corpus data
  exists.
- Not used for chat sessions, memory, or document *metadata* — those are
  DynamoDB (per resolved open question 4). Postgres is scoped to
  vector+lexical retrieval only.

### 2.6 DynamoDB
- Tables (see [05-domain-model.md](05-domain-model.md) for full key
  design): `ChatSessions`, `Messages`, `LongTermMemory`,
  `DocumentMetadata`, `IngestionStatus`, `AuditLog`.
- Chosen for operational data because: serverless scaling fits
  spiky per-user session/message workload, single-digit-ms access
  pattern for session/message reads, integrates cleanly with
  Lambda-based ingestion status updates, no separate operational DB to
  manage.

### 2.7 OpenAI Integration Layer
- Thin internal abstraction (`LLMProvider`, `EmbeddingProvider`
  interfaces) implemented for OpenAI in v1, so a future Ollama
  implementation is a config/DI swap, not a rewrite (per resolved open
  question 6).
- Handles retries/backoff (NFR-011), token/cost accounting (NFR-052),
  and request/response logging hooks for LangSmith.

### 2.8 Guardrails Layer
- Inline in the backend service: input guardrail node (prompt-injection
  screening) before retrieval, output guardrail node (groundedness/PII/
  policy check) before streaming the response to the client. See
  [11-security-spec.md](11-security-spec.md).

### 2.9 Observability
- New Relic APM agents on backend service and ingestion workers; custom
  metrics for retrieval latency, token cost, guardrail-block rate,
  hallucination-flag rate.
- LangSmith for LLM/graph-level tracing and offline eval runs.

### 2.10 MCP Tool Layer (new, v0.2)
- An adapter service (or in-process module of the chat backend — **still
  the open half of ADR-006**; the *what it talks to* half of that ADR is
  now resolved, see below) that exposes org/people-data lookups (role,
  manager, department, hierarchy, training, certifications,
  performance/360 reviews, salary/compensation, and mocked
  family/dependent fields) to the LangGraph agent as MCP tools (FR
  Group L).
- **Read-only by construction**: the tools it registers can only query
  the upstream People-data source, never write to it — there is no write
  code path here to accidentally expose (per
  [00-vision-and-scope.md](00-vision-and-scope.md) §3 Non-Goals).
- **Upstream resolved v0.3** (Q7): calls the Placeholder People-Data
  Service (§2.12), not a real vendor system — see §1.1.
- Owns: auth to the upstream People-data service (internal service-to-
  service auth within the VPC — simpler than an external vendor
  integration would require, since both services are ours), short-TTL
  response caching and rate limiting (NFR-092), and the review-data and
  salary/compensation-data visibility checks (FR-114/BR-09,
  FR-118/BR-13 — both self + **immediate manager only** + HR) — enforced
  *in this layer*, not left to the LLM to self-police.
- Registers a family/dependent-data tool **only against the mocked
  fields** the placeholder service seeds (FR-116, resolved v0.3 for
  build purposes) — gated by the same entitlement check. This registration
  MUST be revisited before any real HRIS (Q7's eventual real resolution)
  is connected — see NFR-095.
- Every call logged to `AuditLog` (NFR-091).

### 2.11 Social Service (new, v0.2)
- A user-facing API (called directly by the frontend, not via LangGraph
  — see §1.1) for profiles, feed, posts, comments, and direct messages
  (FR Group M).
- **Feed fan-out**: a new post is written once to `Posts`; a DynamoDB
  Streams trigger asynchronously fans it out into each follower's `Feed`
  item collection (per
  [05-domain-model.md](05-domain-model.md) §3.1) — this is the standard
  DynamoDB pattern for feed reads (`Query` on one partition) instead of a
  request-time fan-in read across everyone the viewer follows, which
  would not scale as follow-graphs grow. Trade-off: feed delivery is
  eventually consistent (a post may take a moment to appear in
  followers' feeds), which is an accepted cost of this pattern, not an
  oversight.
- **Moderation**: report/removal actions (FR-123–125) go through this
  service; content passes the shared guardrail/content-safety screening
  (NFR-093) before being written to `Posts`/`Comments`/`DirectMessages`.
- Feed visibility default (org-wide, per FR-126) is enforced as a query
  predicate in this service, not a client-side filter — same posture as
  document ACLs (FR-064) and review-data gating (FR-114): access control
  lives in the query, not downstream of it.

### 2.12 Placeholder People-Data Service (new, v0.3)
- Stands in for a real HRIS/People system for v1 (resolves
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q7). Owned and
  built by this team, deployed as its own service in the same ECS
  cluster as the Chat Backend Service — a standalone service rather than
  a module of the backend, so the eventual swap to a real vendor system
  is "point the MCP Tool Layer at a different endpoint," not "extract
  logic out of the chat backend."
- Exposes a simple internal read API (role, manager, department, org
  hierarchy, training, certifications, performance/360 reviews,
  salary/compensation, mocked family/dependent fields) backed by a
  small seeded dataset — a lightweight store (e.g. a single Postgres
  table or DynamoDB table of synthetic employee records) is sufficient;
  this is explicitly not a system designed for production HR data
  volumes or durability guarantees, since it exists to be replaced.
- **All data behind this service is synthetic.** No real employee data —
  compensation, reviews, or otherwise — is ever seeded here. This is
  what makes FR-116's mocked family-data path safe to build now without
  it implying anything about the still-unresolved legal question (Q9).
- Called only by the MCP Tool Layer (§2.10) — not exposed to the
  frontend or any other component directly, so there is exactly one
  integration point to update when a real HRIS eventually replaces it.
- Explicitly out of scope for this service: write endpoints (nothing
  should ever be able to modify "employee" records through PolicyBot,
  mock or otherwise — keeps the read-only posture in §3 Non-Goals true
  even against the placeholder), and production-grade auth/rate-
  limiting beyond basic VPC-internal access control (not warranted for
  a service holding no real data).

## 3. Deployment Topology (AWS)

- **Frontend**: S3 (static assets) + CloudFront (CDN/TLS) + Route53.
  Independent CI/CD pipeline (see
  [14-infra-and-cicd-spec.md](14-infra-and-cicd-spec.md)).
- **Backend**: ECS Fargate service behind an internal ALB, fronted by
  API Gateway (HTTP API + VPC Link) or ALB directly with Cognito
  authorizer — final choice in infra spec.
- **Placeholder People-Data Service (new, v0.3)**: a second ECS Fargate
  service in the same cluster, internal-only (no ALB/API Gateway
  exposure, no public or authenticated-user-facing route) — reached only
  by the MCP Tool Layer over the internal VPC network. Deliberately
  minimal relative to the main backend's deployment rigor (§2.12), since
  it holds no real data.
- **VPC**: private subnets for ECS tasks, Aurora, and ingestion
  compute; NAT for outbound calls to OpenAI API; DynamoDB and S3 reached
  via VPC endpoints (no public internet hop for those).
- **Multi-AZ**: Aurora and ECS service span ≥2 AZs (NFR-012).
- **Environments**: dev, staging, prod — isolated AWS accounts or, at
  minimum, isolated VPCs/resource namespaces (decision in infra spec).

## 4. Key Architecture Decisions Requiring an ADR

Recorded as stubs here; full ADRs live in `/specs/16-adr/`:
- ADR-001: Postgres+pgvector (self-managed) vs. managed vector DB.
- ADR-002: DynamoDB vs. Postgres for operational (non-vector) data.
- ADR-003: ECS Fargate vs. Lambda for the chat backend/LangGraph host.
- ADR-004: OpenAI now, Ollama-ready abstraction for future swap.
- ADR-005: Single S3 ingestion landing zone for both upload and sync
  sources.
- ADR-006 (new, v0.2, **partially resolved v0.3**): MCP Tool Layer
  deployment shape — in-process module of the chat backend vs. a
  standalone adapter service — still open. **Resolved**: which system
  it integrates with — the Placeholder People-Data Service (§2.12), a
  service we own, not a real vendor HRIS. A follow-up ADR is needed
  later specifically for real-vendor selection (Workday/SuccessFactors/
  etc.) when that becomes real work, separate from this deployment-shape
  question.
- ADR-007 (new, v0.2): Social Service — separate microservice vs. a
  module within the chat backend. Leans toward separate (different
  scaling profile and release cadence than the LangGraph agent, no
  reason to couple their deploys) but not yet decided.
- ADR-008 (new, v0.2): Social data store and feed-delivery pattern —
  DynamoDB with the fan-out-on-write pattern described in §2.11 is the
  working assumption; revisit if feed/social-graph query patterns (e.g.
  "mutual connections," complex ranking) outgrow what a fan-out table
  design supports well.

## 5. Traceability to Requirements

| Component | Key FRs/NFRs |
|---|---|
| Frontend | FR-010–016, FR-020–024, NFR-062, NFR-070–071 |
| API Gateway/Auth | FR-001–004, NFR-030–031 |
| Chat backend/LangGraph | FR-013, FR-070–073, NFR-001–003, NFR-020 |
| Ingestion pipeline | FR-040–054, FR-090–092, NFR-013, NFR-021 |
| Postgres/pgvector | FR-060–065, NFR-004, NFR-022 |
| DynamoDB | FR-020–033, FR-091, NFR-033, NFR-041 |
| OpenAI layer | FR-070, NFR-011, NFR-052 |
| Guardrails | FR-080–083, NFR-034 |
| Observability | NFR-050–052 |
| MCP Tool Layer (new, v0.2) | FR-072, FR-110–118, NFR-090–092, NFR-095 |
| Social Service (new, v0.2) | FR-120–126, NFR-093–094 |
| Placeholder People-Data Service (new, v0.3) | FR-110–118, NFR-095 |
