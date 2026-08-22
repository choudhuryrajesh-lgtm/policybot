# 00 — Vision and Scope

## Status
Draft v0.2 — 2026-08-21

**Revision note (v0.2)**: expanded scope to include live employee/org
data (via MCP tools) and an internal employee social layer. This
reverses the v0.1 Non-Goal that excluded HRIS/employee-record data and
the "read-only" framing. See §8 for what's still genuinely unresolved,
and the ripple note at the end of this doc's history — specs
[02](02-functional-requirements.md)–[05](05-domain-model.md) and
[08](08-generation-spec.md) were written against v0.1 scope and need a
follow-up revision pass to match.

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
  sourced live from the system-of-record, not a stale copy.
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
  people-data access (esp. performance/360 review data — self + manager
  chain + HR only, never arbitrary employee-to-employee), prompt-injection
  and hallucination guardrails, PII handling — now with materially higher
  stakes given family/dependent and performance data in scope.
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
- Employee/org data is assumed to already exist in a company HRIS/People
  system today (family/dependent info, training, certifications, and
  360/performance reviews are typically HRIS- or LMS-native features) —
  PolicyBot is assumed to be a *read surface* onto that existing system,
  not the first place this data is collected. **This assumption needs
  confirmation** (see §8 Q9) — if any of this data is not already
  collected and governed elsewhere, surfacing it here would make PolicyBot
  the system of record, which is a materially different (and much
  heavier) compliance posture.

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

Opened 2026-08-21 (v0.2 scope expansion — none of these are resolved yet;
listed in rough order of how much they block downstream specs):

7. **HRIS/People system identity**: which system is the source of truth
   for role, manager, department, org hierarchy, training,
   certifications, and performance/360 review data (e.g. Workday,
   SuccessFactors, BambooHR, a custom internal system)? This determines
   what the MCP tool integration in §6 actually connects to and what its
   auth/rate-limit constraints are. **Blocks**: the MCP-tools architecture
   component and its domain model.
8. **Performance/360 review access model**: default assumption pending
   confirmation — visible only to (a) the employee themselves, (b) their
   direct management chain, (c) HR — never to an arbitrary employee
   asking about a peer. **Must be confirmed before this is built** — this
   is the most sensitive data category in scope and the access-control
   design (query-time enforcement, not app-level trust) needs to mirror
   the document-ACL rigor already established for FR-004/FR-064, not be
   weaker than it.
9. **Family/dependent information — compliance basis, unresolved,
   flagging rather than assuming**: what specific fields are meant
   (dependents' names, emergency contacts, something else?), and under
   what legal basis can this be surfaced through a chatbot rather than
   only the system it already lives in? This is qualitatively more
   sensitive than the rest of the people-data scope and several
   jurisdictions treat family/dependent data as a distinct, more
   restricted category of PII. **No default assumption is made here** —
   unlike open question 2 above, this is not the kind of gap where a
   reasonable placeholder is safe to build against. **Must have explicit
   Legal/Compliance sign-off before any FR referencing this is written**,
   let alone implemented. Until resolved, treat family/dependent data as
   out of scope in practice even though G6/§6 lists it directionally.
10. **Social feature moderation and feed mechanics**: G7/§6 confirm the
    feature set (profiles, feed, posts, comments, DMs) is in scope in
    full, not a stripped-down subset — what's still undecided is
    moderation policy (reporting/blocking, content review, prohibited
    content), feed visibility/ranking rules (who sees whose posts —
    org-wide, department-scoped, or follow-based), retention period for
    social content, and whether it's subject to the same audit-log
    discipline as chat (BR-06). **Blocks**: the social-layer domain model
    and any FR/NFR numbering for it.
11. **Ripple into already-written specs**: [02](02-functional-requirements.md)–
    [05](05-domain-model.md) currently reflect v0.1 scope (e.g. `01`
    §3 previously listed HRIS data as explicitly out of scope; `08`
    §8 previously deferred tool-calling to v2). These need a revision
    pass once Q7–Q10 above are resolved enough to write concrete
    requirements against, rather than before — writing FR/NFR/
    architecture content ahead of Q7–Q9 in particular would mean
    designing against a system and a legal basis that don't exist yet.

## 9. Spec Index

See `/specs` directory for the full spec set; this document is the anchor.
Each functional requirement below traces to later specs via requirement
IDs (`FR-xxx`, `NFR-xxx`).
