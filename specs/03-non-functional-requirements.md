# 03 — Non-Functional Requirements

## Status
Draft v0.3 — 2026-08-21 (v0.2 added NFR Group J for the people-data/
social scope expansion; v0.3 resolves that group's open questions —
immediate-manager-only entitlement confirmed, real-vs-mock family data
distinction made explicit in NFR-095)

Traces from: [00-vision-and-scope.md](00-vision-and-scope.md),
[02-functional-requirements.md](02-functional-requirements.md)

---

## NFR Group A — Performance

- **NFR-001**: THE SYSTEM SHALL return the first streamed token of a
  response within p95 2.5s of query submission (measured from API receipt,
  excluding client network time).
- **NFR-002**: THE SYSTEM SHALL complete a full typical answer (≤ 500
  output tokens) within p95 8s.
- **NFR-003**: THE SYSTEM SHALL execute hybrid retrieval (BM25 + ANN + fuse
  + rerank) within p95 400ms server-side, excluding LLM generation time.
- **NFR-004**: THE SYSTEM'S HNSW index configuration SHALL be tuned
  (`ef_search`, `m`, `ef_construction`) to sustain the retrieval latency
  target at the expected corpus scale (target: up to ~5M chunks — revisit
  once real corpus size from vision-doc open question is known).
- **NFR-005**: THE SYSTEM SHALL support at least 200 concurrent active chat
  sessions at launch without SLA degradation, horizontally scalable beyond
  that.

## NFR Group B — Availability & Reliability

- **NFR-010**: THE SYSTEM SHALL target 99.5% monthly availability for the
  chat API (excludes scheduled maintenance windows communicated in
  advance).
- **NFR-011**: WHEN the OpenAI API is unavailable or rate-limited, THE
  SYSTEM SHALL retry with exponential backoff (bounded attempts) and THEN
  surface a clear, user-friendly error rather than hanging or crashing.
- **NFR-012**: THE SYSTEM SHALL be deployed across at least two
  Availability Zones for all stateful production components (DB, backend
  compute).
- **NFR-013**: Ingestion pipeline failures on one document SHALL NOT block
  or fail ingestion of other documents in the same batch.

## NFR Group C — Scalability

- **NFR-020**: THE SYSTEM'S backend compute SHALL scale horizontally
  (stateless request handling; session/memory state externalized to
  Postgres/cache, not in-process).
- **NFR-021**: THE SYSTEM'S ingestion pipeline SHALL scale independently
  from the query-serving path (separate worker pool/queue) so large
  ingestion batches (e.g. bulk video upload) do not degrade chat latency.
- **NFR-022**: THE vector store SHALL support index growth to at least 10x
  the launch corpus size without requiring an architecture change (only
  scaling/tuning).

## NFR Group D — Security

- **NFR-030**: All data in transit SHALL be encrypted (TLS 1.2+); all data
  at rest (Postgres, S3) SHALL be encrypted using AWS KMS-managed keys.
- **NFR-031**: THE SYSTEM SHALL enforce least-privilege IAM roles per
  component (no shared/broad credentials between ingestion worker, API,
  and admin functions).
- **NFR-032**: Secrets (OpenAI API key, DB credentials) SHALL be stored in
  AWS Secrets Manager, never in source control or environment files
  committed to the repo.
- **NFR-033**: THE SYSTEM SHALL log all admin actions (document
  upload/delete, ACL change) to an immutable audit log.
- **NFR-034**: THE SYSTEM SHALL be resilient to the OWASP Top 10 for LLM
  Applications (prompt injection, insecure output handling, training data
  poisoning n/a, model DoS, sensitive info disclosure, insecure plugin
  design, excessive agency, overreliance) — see
  [11-security-spec.md](11-security-spec.md) for control mapping.

## NFR Group E — Privacy & Compliance

- **NFR-040**: THE SYSTEM SHALL redact or avoid logging raw PII in
  LangSmith traces and application logs beyond what is operationally
  necessary.
- **NFR-041**: THE SYSTEM SHALL support per-user deletion of chat history
  and long-term memory on request (supports right-to-erasure-style
  requests).
- **NFR-042**: Chat logs and memory SHALL be retained no longer than the
  company's data retention policy permits, with automated expiry.

## NFR Group F — Observability

- **NFR-050**: THE SYSTEM SHALL emit structured logs, metrics, and traces
  to New Relic for all backend components, including per-LangGraph-node
  timing and token usage.
- **NFR-051**: THE SYSTEM SHALL alert on-call when error rate, p95
  latency, hallucination-flag rate, or guardrail-block rate exceed defined
  thresholds (see [13-observability-spec.md](13-observability-spec.md)).
- **NFR-052**: THE SYSTEM SHALL track OpenAI token usage and cost per
  request, aggregable by day/user/session for cost governance.

## NFR Group G — Maintainability / Delivery

- **NFR-060**: THE SYSTEM SHALL be deployable via an automated CI/CD
  pipeline with no manual production deployment steps once a change is
  approved and merged.
- **NFR-061**: THE SYSTEM'S infrastructure SHALL be defined as code
  (Terraform or AWS CDK), with no manually-created production resources.
- **NFR-062**: THE SYSTEM SHALL support independent deployment of the
  frontend (CloudFront/S3) and backend without requiring simultaneous
  releases (versioned, backward-compatible API contract).
- **NFR-063**: THE SYSTEM SHALL maintain automated test coverage gates
  (unit, component, eval) enforced in CI before merge/deploy — see
  [15-testing-strategy.md](15-testing-strategy.md).

## NFR Group H — Usability / Accessibility

- **NFR-070**: THE frontend SHALL meet WCAG 2.1 AA accessibility standards.
- **NFR-071**: THE frontend SHALL be responsive and usable on desktop and
  tablet viewport sizes (internal usage assumed primarily desktop).

## NFR Group I — Accuracy / Quality Targets

(Restated here from vision doc for traceability into design/eval specs.)

- **NFR-080**: Faithfulness/groundedness ≥ 90% on golden eval set.
- **NFR-081**: Answer relevancy ≥ 85% on golden eval set.
- **NFR-082**: Context precision ≥ 0.80, context recall ≥ 0.85 on golden
  eval set.
- **NFR-083**: Hallucination rate (unsupported factual claims) < 5% on
  golden eval set.
- **NFR-084**: These thresholds SHALL be enforced as CI quality gates —
  a build that regresses below threshold fails the pipeline (see
  [12-evaluation-spec.md](12-evaluation-spec.md)).

## NFR Group J — People Data & Social Privacy/Safety

Added in v0.2, tracing to
[00-vision-and-scope.md](00-vision-and-scope.md) G6/G7 and
[01-business-requirements.md](01-business-requirements.md) BR-09–BR-12.

- **NFR-090**: Performance/360 review data (FR-114) and salary/
  compensation data (FR-118) SHALL be enforced with the same or higher
  access-control rigor as manager-only documents (NFR-030–034 baseline),
  scoped to self + **immediate manager only** + HR, with the check
  happening at MCP-tool-call time, not solely via system-prompt
  instruction. A denied request SHALL return an identical refusal
  regardless of the underlying data, so a denial itself cannot be used
  to infer the data's content (see
  [00-vision-and-scope.md](00-vision-and-scope.md) §10 Scenario C).
- **NFR-091**: Every MCP tool call against the HRIS/People system SHALL
  be logged to the immutable audit log (actor, employee record accessed,
  tool, timestamp) — extends NFR-033's audit scope from document/ACL
  actions to people-data reads, which is a new and more sensitive
  category of access to have visibility into.
- **NFR-092**: MCP tool calls against the HRIS/People system SHALL be
  rate-limited and short-TTL cached at the integration layer, so that
  chat-driven query volume cannot degrade or overload a system that
  wasn't designed to serve as an LLM-agent backend.
- **NFR-093**: User-generated social content (posts, comments, direct
  messages) SHALL pass through the same content-safety screening used for
  generated chat output (per FR-081's guardrail pattern) before being
  visible to other users — this is a distinct control from FR-081 itself
  (screening *user* content, not *model* output) but reuses the same
  guardrail infrastructure rather than a separate pipeline.
- **NFR-094**: Social content retention/deletion SHALL follow the same
  policy discipline as chat logs (NFR-042) — exact retention period for
  social content specifically is open pending
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q10.
- **NFR-095**: **Real** employee family/dependent data SHALL NOT be
  requested, stored, cached, or logged by the system in any form, in any
  component — a technical control mirroring BR-10. **Synthetic/mocked**
  family/dependent data (v1's placeholder People-Data Service, per
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q7/Q9) is exempt
  from this control for build/demo purposes, but any record fetched from
  a real HRIS integration in the future MUST be treated as real data
  under this NFR from the moment that integration exists — there is no
  grace period implied by the fact that a mocked version predates it.
