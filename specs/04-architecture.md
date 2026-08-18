# 04 — Architecture

## Status
Draft v0.1 — 2026-08-17

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

## 3. Deployment Topology (AWS)

- **Frontend**: S3 (static assets) + CloudFront (CDN/TLS) + Route53.
  Independent CI/CD pipeline (see
  [14-infra-and-cicd-spec.md](14-infra-and-cicd-spec.md)).
- **Backend**: ECS Fargate service behind an internal ALB, fronted by
  API Gateway (HTTP API + VPC Link) or ALB directly with Cognito
  authorizer — final choice in infra spec.
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
