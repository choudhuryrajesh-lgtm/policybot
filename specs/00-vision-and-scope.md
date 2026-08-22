# 00 — Vision and Scope

## Status
Draft v0.3 — 2026-08-21

**Revision history**:
- **v0.2**: expanded scope to include live employee/org data (via MCP
  tools) and an internal employee social layer. Reversed the v0.1
  Non-Goal that excluded HRIS/employee-record data and the "read-only"
  framing.
- **v0.3**: resolved the four open questions the v0.2 expansion raised
  (§8 Q2, Q7, Q9, Q10) — Cognito groups confirmed as ACL source of
  truth, a placeholder People-Data Service (our own, on ECS) stands in
  for a real HRIS, family/dependent data is mocked for build purposes
  only (real data still legally gated), and social moderation ships a
  minimal report-only default. Specs
  [02](02-functional-requirements.md)–[05](05-domain-model.md) and
  [08](08-generation-spec.md) are updated to match; `11`+ can now be
  written against settled ground.

## 1. Problem Statement

Internal employees need to find accurate answers to questions about company
policy and internal official documents (HR policy, IT policy, compliance,
benefits, SOPs, legal/contract templates, leadership decks, training videos,
etc.). Today this information is scattered across document repositories,
requires manual search, and often produces inconsistent or stale answers.

Separately, employees also need to find and understand **people and
organizational context** — who someone's manager is, what team/department
they're in, where they sit in the org hierarchy, their training/
certification status — and to connect with colleagues directly, which
today means separate directory tools, HRIS portals, and ad hoc channels.

We are building an internal **Retrieval-Augmented Generation (RAG)
chatbot** ("PolicyBot") that lets employees ask natural-language questions
and receive grounded, cited answers sourced from ingested internal
documents (text, images, video) **and** from live organizational/people
data — plus a full internal social layer (profiles, feed, posts,
comments, direct messages) so employees can connect and interact with
each other directly from the same product.

## 2. Goals

- G1: Employees get accurate, cited answers to policy/document questions in
  under a few seconds, instead of manually searching document repositories.
- G2: Reduce load on HR/IT/Legal help-desk teams for repetitive policy
  questions.
- G3: Support ongoing document churn (policies are revised frequently)
  without manual re-indexing effort — re-upload detection via content
  hashing.
- G4: Be demonstrably safe to deploy internally: no hallucinated policy
  advice, no leakage of documents or people-data across access boundaries,
  resistant to prompt injection.
- G5: Be operable long-term: CI/CD-deployed, monitored, evaluated
  continuously against a golden dataset, cost-tracked.
- G6: Employees can ask people/org questions ("who is X's manager", "what
  team is X on", "show me the org chart under Y") and get accurate answers
  sourced live from the system-of-record, not a stale copy. Includes
  self-service questions about one's own sensitive data ("what was my
  salary hike this year", "what did my 360 feedback say") — answered for
  the employee themselves, their immediate manager, or HR, and refused
  for anyone else (see §10 for a concrete walkthrough).
- G7: Employees can build colleague profiles, post and comment in a
  shared internal feed, and message each other directly — a full
  internal social experience, not a stripped-down directory add-on —
  within the same product, subject to moderation and the same
  audit/retention discipline as chat.

## 3. Non-Goals (out of scope for v1)

- Public-facing / customer-facing chatbot (internal employees only).
- The bot **modifying** any system-of-record data (HRIS, performance
  management, LMS) — all employee/org/training/review data is
  **read-only** via MCP tool calls; the HRIS/People system remains
  authoritative and the only place that data is edited. (Revised from
  v0.1: v1 is no longer read-only *overall* — social content, see G7, is
  user-generated and written by the app itself — but it remains read-only
  with respect to HR systems of record specifically.)
- Real-time collaborative editing of documents.
- On-prem / air-gapped deployment (AWS cloud only).
- Fine-tuning a custom LLM (v1 uses OpenAI hosted models via API).
- Multi-tenant SaaS (single internal org only).
- Public/external visibility of any social content — the social layer
  (G7) is internal-only (no third-party integrations, no external
  sharing), but within that boundary it is **not** limited to bare
  primitives: profiles, a feed, posts, comments, and direct messages are
  all in scope (see §6, revised).

## 4. Primary Users / Personas

| Persona | Description | Key need |
|---|---|---|
| Employee (all depts) | General staff asking about policy/benefits/IT/etc., and about colleagues/org structure | Fast, trustworthy answers with source links; accurate people/org lookups; ability to connect with colleagues |
| Manager | Needs policy answers plus some manager-only documents (e.g. compensation bands) and manager-only people data (direct reports' reviews) | Same as employee + access to role-gated docs and role-gated people data |
| HR/IT/Legal admin (content owner) | Uploads/maintains source documents; is also the steward of employee/org data access policy | Reliable ingestion, versioning, re-upload without duplication, visibility into what's indexed; confidence that people-data access rules are correctly enforced |
| Platform engineer (us) | Builds/operates the system | Observability, evals, safe deploys, cost control |
| Security/Compliance reviewer | Approves the system for internal use | Guardrails, audit trail, access control evidence — now including people-data sensitivity (esp. family/dependent data, performance reviews) |

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
- **Safety**: 0 confirmed cross-tenant/cross-permission document or
  people-data leaks; hallucination rate on golden set < 5% (unsupported
  claims presented as fact).
- **Cost**: average cost per conversation turn tracked and kept under an
  agreed ceiling (to be set once OpenAI usage patterns are known).
- **People-data accuracy**: 0 confirmed cases of the bot returning stale
  or incorrect org-chart/manager/role data (live MCP lookup, not a cached
  copy, is the control for this — see §6).
- **Social engagement**: to be set once the feature is scoped in detail
  (see §8, open) — deferred rather than guessed, since the feature itself
  is still being defined.

## 6. High-Level Solution Shape

- **Frontend**: React SPA, deployed independently to S3 + CloudFront, talks
  to backend via authenticated REST/streaming API. Supports multiple
  concurrent chat windows/sessions per user, returning users retain
  history. Now also hosts colleague profile views and the social feature
  surface (G7).
- **Orchestration**: LangGraph-based agent graph (retrieve → grade →
  rewrite/reroute → generate → verify/guardrail) using OpenAI models.
- **Retrieval**: Postgres + pgvector (HNSW/ANN) for dense vectors, hybrid
  with BM25 sparse search, fused via reranking — scoped to *document*
  content.
- **Ingestion**: S3-triggered pipeline supporting text, image, and video
  documents; context-aware chunking; content-hash-based dedup so
  re-uploads of an unchanged or partially-changed document don't blow up
  the index.
- **Org & people data (new, G6)**: **not** ingested/indexed like documents
  — read live via **MCP tools** the LangGraph agent can call against the
  HRIS/People system of record (role, manager, department, org hierarchy,
  training, certifications, performance/360 review data). This keeps
  answers current by construction (no staleness/re-sync problem) and
  keeps write-authority with the HRIS. Activates the tool-calling
  extension point already designed into the generation graph (see
  [08-generation-spec.md](08-generation-spec.md) §8, previously deferred
  to v2 — now v1).
- **Social layer (new, G7)**: full internal social experience — employee
  profiles, a shared feed, posts, comments, and direct messages between
  employees — natively stored (this data has no external system of
  record — the app *is* the source of truth for it), with its own access
  model, moderation, and retention policy — see §8 for what's still open
  here (feed visibility rules and moderation policy in particular are
  unresolved, not the feature's inclusion).
- **Operational data store**: DynamoDB for chat sessions/history,
  long-term memory, document/ingestion metadata, audit log, and (new)
  social content (profiles, posts, comments, messages).
- **Memory**: short-term (per-session) and long-term (per-user, persisted
  across visits) memory, both persisted in DynamoDB.
- **Evaluation**: golden dataset + LangSmith eval pipeline, run in CI as a
  quality gate.
- **Security**: authN/Z (SSO/OIDC), document-level ACLs, role-based
  people-data access (performance/360 review data **and** salary/
  compensation data — self + immediate manager + HR only, never arbitrary
  employee-to-employee, and never the full reporting chain), prompt-
  injection and hallucination guardrails, PII handling — now with
  materially higher stakes given family/dependent and compensation data
  in scope.
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
- **Revised 2026-08-21 (resolves Q7)**: v1 does **not** integrate a real
  external HRIS/People system yet. Instead, a **placeholder People-Data
  Service** — owned and built by this team, deployed in our own ECS
  cluster — stands in for it, seeded with synthetic/mock org data (role,
  manager, hierarchy, training, certifications, reviews, salary, and
  mocked family/dependent fields per Q9 below). The MCP Tool Layer
  ([04-architecture.md](04-architecture.md) §2.10) talks to this
  placeholder through the same interface a real HRIS integration would
  use, so swapping in an actual vendor system (Workday, SuccessFactors,
  etc.) later is a backend swap behind a stable interface, not a
  redesign — the same posture as ADR-004's OpenAI/Ollama abstraction.
  **This means PolicyBot is, for now, actually the system that holds this
  mock data** — it is not yet a pure read-surface onto something else,
  which is the opposite of the v0.1/early-v0.2 assumption. That's fine
  for build/demo purposes; it's exactly why Q9's family-data resolution
  below is scoped to mock data only, not real employee data.

## 8. Open Questions

Resolved 2026-08-17:

1. **SSO/IdP**: Amazon Cognito.
2. **Access restrictions / source of truth — RESOLVED 2026-08-21**:
   Cognito user-pool groups are the confirmed source of truth for
   role/department entitlements, mapped 1:1 to document ACL tags.
   Explicitly acknowledged as a v1 decision that may be upgraded to a
   dedicated entitlement system later if Cognito groups prove too coarse
   (e.g. for the immediate-manager-specific checks in Q8) — that upgrade
   would change only how `access_tags`/manager relationships are
   populated, not the ACL enforcement pattern itself (query/tool-level
   filtering, per FR-064/FR-114/FR-118).
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

Opened 2026-08-21 (v0.2 scope expansion), resolved same day:

7. **HRIS/People system identity — RESOLVED 2026-08-21**: no real vendor
   system for v1. A **placeholder People-Data Service** (our own,
   deployed on ECS) stands in, seeded with mock/synthetic data covering
   every category in this open question — see §7's revised assumption
   above and [04-architecture.md](04-architecture.md) §2.12 for the
   component. Real-vendor integration (Workday/SuccessFactors/etc.) is
   deferred to a future ADR when there's an actual system to integrate
   with; the MCP tool interface is written so that swap doesn't ripple
   into the generation graph or FR/NFR text (FR Group L already treats
   "the HRIS/People system" generically for exactly this reason).
8. **Performance/360 review AND salary/compensation access model —
   RESOLVED 2026-08-21**: visible only to (a) the employee themselves,
   (b) their **immediate manager only** (not the full reporting chain —
   a skip-level manager has no more access than any other employee), and
   (c) HR — never to an arbitrary employee asking about a peer. This now
   covers salary/compensation data (current salary, hike/increase
   history) as well as performance/360 review data — same rule, same
   enforcement posture (query-time/tool-level enforcement, mirroring the
   document-ACL rigor already established for FR-004/FR-064, not weaker
   than it). See §10 for a concrete walkthrough. Implemented as
   FR-114/FR-118 in [02-functional-requirements.md](02-functional-requirements.md),
   BR-09/BR-13 in [01-business-requirements.md](01-business-requirements.md).
9. **Family/dependent information — RESOLVED for build purposes
   2026-08-21, legal question itself still open**: two separate
   questions were being conflated here, and only one is resolved.
   - *Can we build/demo the feature now?* Yes — the placeholder
     People-Data Service (Q7) seeds **synthetic, fabricated**
     family/dependent fields (no real employee data exists anywhere in
     this system yet), so FR-116's tool can be built and tested against
     mock records.
   - *Can we expose real employee family/dependent data in production?*
     **Still no, and still unresolved** — the underlying legal-basis
     question (what fields, under what jurisdictioned legal basis) has
     not been answered, only deferred by the fact that v1 has no real
     data source to expose in the first place. This distinction matters
     enough to restate: **do not treat "we mocked it" as "legal signed
     off on it."** When a real HRIS (Q7) is eventually integrated, the
     family-data legal question must be answered *before* that field is
     connected — mocking now does not grandfather it in later.
10. **Social feature moderation and feed mechanics — RESOLVED (lightweight
    default) 2026-08-21**: rather than block on a full moderation
    program, v1 ships a minimal default: **report-only moderation** (a
    user can report a post/comment/message per FR-123; reports queue in
    `ModerationReports` for manual HR/admin review — no automated
    content moderation beyond the shared guardrail content-safety
    screening already required by NFR-093) and **org-wide feed
    visibility** (FR-126's existing default — no department-scoping or
    follow-based visibility restriction in v1). Explicitly treated as an
    intentionally open/minimal starting point, not a fully-designed
    moderation program — revisit if usage patterns show it's
    insufficient (e.g. abuse volume, review SLA complaints).
11. **Ripple into already-written specs**: [02](02-functional-requirements.md)–
    [05](05-domain-model.md) and [08](08-generation-spec.md) have been
    revised to match the v0.2 scope expansion and the Q2/Q7/Q9/Q10
    resolutions above — see each doc's Status header for its revision
    note. `11-security-spec.md` onward are not yet written; they can now
    be written against the placeholder People-Data Service and Cognito
    groups as settled ground rather than open questions.

## 9. Spec Index

See `/specs` directory for the full spec set; this document is the anchor.
Each functional requirement below traces to later specs via requirement
IDs (`FR-xxx`, `NFR-xxx`).

## 10. Example Use Case: Sensitive Self-Service People-Data Query

Grounds open question 8's now-resolved entitlement rule in a concrete
walkthrough, since "self + immediate manager + HR only" is easy to state
abstractly and easy to get wrong in enforcement.

**Scenario A — asking about yourself (allowed):**
1. Employee logs in via corporate SSO (FR-001); session resolves their
   identity and role (FR-003).
2. Employee asks: *"What was my salary hike this year?"* or *"What did my
   360 feedback say?"*
3. `query_rewrite` (per [08-generation-spec.md](08-generation-spec.md))
   passes the query to `generate`, which calls the salary/review MCP
   tool (FR-117/FR-113) with the requesting user's own identity as the
   subject.
4. The MCP Tool Layer's entitlement check (FR-118/FR-114) evaluates:
   requesting user == subject employee → **allowed**. Data returned,
   answer generated, logged to `AuditLog` (NFR-091).

**Scenario B — a manager asking about their direct report (allowed):**
1. Manager asks: *"What was Priya's salary hike this year?"*
2. Same tool call, subject = Priya.
3. Entitlement check: requesting user == Priya's **immediate** manager →
   **allowed**.

**Scenario C — a peer, or a skip-level manager, asking about someone else
(refused):**
1. Employee (or Priya's manager's own manager — skip-level) asks: *"What
   was Priya's salary hike this year?"*
2. Same tool call, subject = Priya.
3. Entitlement check: requesting user is neither Priya, Priya's immediate
   manager, nor HR → **refused**. Per FR-115/BR-12, the bot states it
   doesn't have permission to share that rather than confirming or
   denying details that could themselves leak information (e.g. it
   should not say "Priya didn't get a hike" vs. staying silent in a way
   that implies she did — the refusal message is the same regardless of
   what the underlying data actually is, so the response itself carries
   no signal).
4. Logged to `AuditLog` as a denied access attempt (NFR-091) — a pattern
   of repeated denied attempts against the same subject is a security
   signal worth alerting on (see
   [13-observability-spec.md](13-observability-spec.md), not yet
   written).

This is the same query-time enforcement pattern already used for
document ACLs (FR-064) — the difference is only which layer owns the
check (retrieval query vs. MCP tool), not the underlying discipline.
