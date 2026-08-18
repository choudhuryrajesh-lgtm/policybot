# 00 — Vision and Scope

## Status
Draft v0.1 — 2026-08-17

## 1. Problem Statement

Internal employees need to find accurate answers to questions about company
policy and internal official documents (HR policy, IT policy, compliance,
benefits, SOPs, legal/contract templates, leadership decks, training videos,
etc.). Today this information is scattered across document repositories,
requires manual search, and often produces inconsistent or stale answers.

We are building an internal **Retrieval-Augmented Generation (RAG) chatbot**
("PolicyBot") that lets employees ask natural-language questions and receive
grounded, cited answers sourced only from ingested internal documents —
including text, images (scanned policy pages, diagrams), and video
(town halls, training recordings).

## 2. Goals

- G1: Employees get accurate, cited answers to policy/document questions in
  under a few seconds, instead of manually searching document repositories.
- G2: Reduce load on HR/IT/Legal help-desk teams for repetitive policy
  questions.
- G3: Support ongoing document churn (policies are revised frequently)
  without manual re-indexing effort — re-upload detection via content
  hashing.
- G4: Be demonstrably safe to deploy internally: no hallucinated policy
  advice, no leakage of documents across access boundaries, resistant to
  prompt injection.
- G5: Be operable long-term: CI/CD-deployed, monitored, evaluated
  continuously against a golden dataset, cost-tracked.

## 3. Non-Goals (out of scope for v1)

- Public-facing / customer-facing chatbot (internal employees only).
- Write actions (the bot does not modify HR records, submit tickets, or
  execute workflows on the user's behalf) — v1 is read-only Q&A.
- Real-time collaborative editing of documents.
- On-prem / air-gapped deployment (AWS cloud only).
- Fine-tuning a custom LLM (v1 uses OpenAI hosted models via API).
- Multi-tenant SaaS (single internal org only).

## 4. Primary Users / Personas

| Persona | Description | Key need |
|---|---|---|
| Employee (all depts) | General staff asking about policy/benefits/IT/etc. | Fast, trustworthy answers with source links |
| Manager | Needs policy answers plus some manager-only documents (e.g. compensation bands) | Same as employee + access to role-gated docs |
| HR/IT/Legal admin (content owner) | Uploads/maintains source documents | Reliable ingestion, versioning, re-upload without duplication, visibility into what's indexed |
| Platform engineer (us) | Builds/operates the system | Observability, evals, safe deploys, cost control |
| Security/Compliance reviewer | Approves the system for internal use | Guardrails, audit trail, access control evidence |

## 5. Success Metrics

- **Answer quality**: ≥ 90% faithfulness/groundedness score on golden
  eval set (LangSmith), ≥ 85% answer relevancy.
- **Retrieval quality**: context precision ≥ 0.8, context recall ≥ 0.85 on
  golden set.
- **Latency**: p95 end-to-end response start (first token) < 2.5s; full
  answer < 8s for typical queries.
- **Adoption**: ≥ 40% of target employee population uses the bot at least
  once/month within 3 months of launch.
- **Deflection**: ≥ 20% reduction in repetitive HR/IT policy tickets within
  6 months.
- **Safety**: 0 confirmed cross-tenant/cross-permission document leaks;
  hallucination rate on golden set < 5% (unsupported claims presented as
  fact).
- **Cost**: average cost per conversation turn tracked and kept under an
  agreed ceiling (to be set once OpenAI usage patterns are known).

## 6. High-Level Solution Shape

- **Frontend**: React SPA, deployed independently to S3 + CloudFront, talks
  to backend via authenticated REST/streaming API. Supports multiple
  concurrent chat windows/sessions per user, returning users retain history.
- **Orchestration**: LangGraph-based agent graph (retrieve → grade →
  rewrite/reroute → generate → verify/guardrail) using OpenAI models.
- **Retrieval**: Postgres + pgvector (HNSW/ANN) for dense vectors, hybrid
  with BM25 sparse search, fused via reranking.
- **Ingestion**: S3-triggered pipeline supporting text, image, and video
  documents; context-aware chunking; content-hash-based dedup so
  re-uploads of an unchanged or partially-changed document don't blow up
  the index.
- **Operational data store**: DynamoDB for chat sessions/history,
  long-term memory, document/ingestion metadata, and audit log.
- **Memory**: short-term (per-session) and long-term (per-user, persisted
  across visits) memory, both persisted in DynamoDB.
- **Evaluation**: golden dataset + LangSmith eval pipeline, run in CI as a
  quality gate.
- **Security**: authN/Z (SSO/OIDC), document-level ACLs, prompt-injection
  and hallucination guardrails, PII handling.
- **Observability**: New Relic APM + custom RAG metrics/dashboards/alerts.
- **Delivery**: CI/CD pipeline to AWS, environment-promoted (dev → staging →
  prod), infra as code.

## 7. Constraints & Assumptions

- LLM provider is OpenAI (API key available); embedding model likely
  `text-embedding-3-large` (final choice recorded in an ADR).
- Deployment target is AWS.
- Frontend is deployed independently from backend (separate pipelines,
  separate release cadence) — CloudFront/S3 for the SPA.
- Vector store is self-managed Postgres/pgvector (not a managed vector DB),
  per explicit requirement — driven by cost and operational simplicity
  trade-off (see ADR).
- Identity provider is Amazon Cognito.
- Operational (non-vector) data store is DynamoDB.
- Launch corpus: ~1,000 documents (≤10 pages each), 10 videos; design
  ceiling ~1000x that for future growth.

## 8. Open Questions

Resolved 2026-08-17:

1. **SSO/IdP**: Amazon Cognito.
2. **Access restrictions / source of truth**: still open — no answer yet.
   Default assumption until answered: Cognito user-pool groups are the
   source of truth for role/department entitlements, mapped 1:1 to
   document ACL tags. **Must be confirmed before
   [11-security-spec.md](11-security-spec.md) is finalized.**
3. **Corpus size**: launch with ~1,000 documents (≤10 pages each, so
   ≤10,000 pages) and 10 videos. Plan for up to ~1000x growth over time
   (target design ceiling: ~1M documents / ~10M pages, thousands of
   videos). Index/infra sizing (NFR-004, NFR-022) should be designed to
   scale to this ceiling without an architecture change, even though
   launch scale is small.
4. **Data residency / storage**: split store —
   - **DynamoDB** holds all non-vector operational data: chat sessions,
     message history, long-term memory records, document/ingestion
     metadata and status, audit log.
   - **Postgres + pgvector (HNSW/ANN)** remains the vector store for
     chunk embeddings and hybrid BM25+dense retrieval, per the original
     requirement — DynamoDB does not have a production-grade ANN index
     equivalent, so it is not used for embeddings.
   - No specific regulatory data-residency constraint was raised beyond
     this storage split; both stores will run in the same AWS region as
     the rest of the stack (region TBD in infra spec).
5. **Ingestion ownership**: both — (a) direct upload by content owners via
   an admin UI/API, and (b) sync of existing/legacy documents. Both paths
   land raw files in **S3**, which triggers the ingestion pipeline
   (S3 event → processing) rather than having two separate ingestion code
   paths.
6. **OpenAI budget ceiling**: deferred — no ceiling set yet. Cost tracking
   (NFR-052) will still be built in from day one so a ceiling can be set
   once real usage data exists. Note: an internal Ollama (fine-tuned)
   deployment is being considered as a future/alternate LLM provider — the
   generation layer will be built behind a provider-agnostic interface
   (see [08-generation-spec.md](08-generation-spec.md)) so this can be
   swapped in later without a redesign; **OpenAI is the provider for v1**.

## 9. Spec Index

See `/specs` directory for the full spec set; this document is the anchor.
Each functional requirement below traces to later specs via requirement
IDs (`FR-xxx`, `NFR-xxx`).
