# 03 — Non-Functional Requirements

## Status
Draft v0.1 — 2026-08-17

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
