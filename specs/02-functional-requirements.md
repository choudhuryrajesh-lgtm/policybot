# 02 — Functional Requirements

## Status
Draft v0.1 — 2026-08-17

Traces from: [01-business-requirements.md](01-business-requirements.md)
Format: EARS (Easy Approach to Requirements Syntax). Each requirement has a
stable ID referenced by later specs, tasks, and tests. Do not renumber
existing IDs once merged — append new ones.

---

## FR Group A — Authentication & Authorization

- **FR-001**: WHEN a user opens the chat application, THE SYSTEM SHALL
  require authentication via the corporate SSO/OIDC provider before any
  chat or document access is permitted.
- **FR-002**: WHEN a user's SSO session expires, THE SYSTEM SHALL prompt
  re-authentication and preserve the user's in-progress draft message.
- **FR-003**: THE SYSTEM SHALL resolve each authenticated user's role(s)
  and document-access group(s) from the identity provider (or a mapped
  entitlement store) on each session start.
- **FR-004**: WHEN retrieving candidate chunks for a query, THE SYSTEM
  SHALL filter out any chunk whose source document's access-control list
  does not include the requesting user's roles/groups, before those chunks
  are passed to the LLM.

## FR Group B — Chat Interaction

- **FR-010**: WHEN a user submits a question, THE SYSTEM SHALL stream the
  answer back token-by-token (or in incremental chunks) rather than waiting
  for full generation.
- **FR-011**: WHEN an answer is generated from retrieved context, THE
  SYSTEM SHALL include inline or end-of-answer citations linking to the
  specific source document(s) and location (page/section/timestamp for
  video).
- **FR-012**: IF no retrieved chunk meets the minimum relevance threshold,
  THEN THE SYSTEM SHALL respond that it does not have enough information,
  and SHALL NOT fabricate an answer.
- **FR-013**: WHEN a user asks a follow-up question, THE SYSTEM SHALL use
  the conversation history within the current session as context for
  retrieval and generation (conversational RAG / query rewriting).
- **FR-014**: THE SYSTEM SHALL support markdown rendering (tables, lists,
  code blocks) in bot responses.
- **FR-015**: WHEN a response is being generated, THE SYSTEM SHALL allow
  the user to cancel/stop generation.
- **FR-016**: WHEN a user gives a thumbs-up/thumbs-down on a response, THE
  SYSTEM SHALL persist that feedback linked to the message and session for
  later evaluation/analysis.

## FR Group C — Multi-Session & Chat Windows

- **FR-020**: THE SYSTEM SHALL allow a user to have multiple concurrent
  chat sessions ("chat windows"), each with independent history and
  context.
- **FR-021**: WHEN a user creates a new chat session, THE SYSTEM SHALL
  initialize it with no prior message history (but see FR-032 for
  long-term memory carryover).
- **FR-022**: THE SYSTEM SHALL persist each session's message history so a
  user can close the app and resume a session later.
- **FR-023**: THE SYSTEM SHALL allow a user to list, rename, and delete
  their own chat sessions.
- **FR-024**: THE SYSTEM SHALL NOT allow a user to view, list, or access
  another user's chat sessions under any circumstance.

## FR Group D — Long-Term Memory

- **FR-030**: WHEN a user returns in a new session after a prior
  conversation, THE SYSTEM SHALL be able to draw on relevant long-term
  memory (e.g. previously stated role, preferences, recurring topics) if
  the user references it, without requiring the user to repeat context.
- **FR-031**: THE SYSTEM SHALL periodically summarize and persist salient
  facts from a session into a per-user long-term memory store, distinct
  from the raw message log.
- **FR-032**: WHEN generating a response, THE SYSTEM SHALL consult
  long-term memory as an additional (clearly lower-priority than
  document-grounded retrieval) context source.
- **FR-033**: THE SYSTEM SHALL allow a user to view and delete their own
  stored long-term memory ("what does the bot remember about me").

## FR Group E — Document Ingestion (Text)

- **FR-040**: THE SYSTEM SHALL support ingesting PDF, DOCX, PPTX, and plain
  text/markdown documents.
- **FR-041**: WHEN a document is ingested, THE SYSTEM SHALL compute a
  content hash (whole-document) and a per-chunk hash, and SHALL store both.
- **FR-042**: WHEN a document is re-uploaded with a content hash identical
  to an existing indexed document, THE SYSTEM SHALL skip re-processing and
  mark the upload as a no-op duplicate.
- **FR-043**: WHEN a document is re-uploaded with a modified content hash
  but the document ID/logical identity is unchanged, THE SYSTEM SHALL
  diff at the chunk level (via per-chunk hash) and SHALL only re-chunk,
  re-embed, and re-index the changed regions, leaving unchanged chunks'
  embeddings intact.
- **FR-044**: THE SYSTEM SHALL use context-aware (semantic) chunking as the
  primary chunking strategy, prioritized over fixed-size chunking, so that
  chunks respect section/paragraph/heading boundaries.
- **FR-045**: THE SYSTEM SHALL preserve document structure metadata per
  chunk (source doc ID, title, section heading path, page number,
  version/hash, access-control tags).
- **FR-046**: WHEN an ingested document is deleted or superseded, THE
  SYSTEM SHALL remove or tombstone its chunks/embeddings so they are no
  longer retrievable.

## FR Group F — Document Ingestion (Image & Video)

- **FR-050**: THE SYSTEM SHALL support ingesting standalone images
  (scanned policy pages, diagrams, PNG/JPG) via OCR text extraction plus an
  image-captioning/description step, both stored as retrievable text with
  a link back to the source image.
- **FR-051**: WHEN a document (e.g. PPTX/PDF) contains embedded images,
  THE SYSTEM SHALL extract and process those images through the same
  OCR/captioning pipeline and associate the resulting text with the
  surrounding document chunk for context.
- **FR-052**: THE SYSTEM SHALL support ingesting video files by extracting
  (a) an audio transcript via speech-to-text, and (b) periodic/scene-change
  keyframes captioned via a vision model.
- **FR-053**: THE SYSTEM SHALL chunk video transcripts with timestamp
  ranges so that citations can deep-link to the relevant moment in the
  video.
- **FR-054**: THE SYSTEM SHALL apply the same content-hash-based dedup
  (FR-041–043) to image and video source files.

## FR Group G — Retrieval

- **FR-060**: WHEN a user query is received, THE SYSTEM SHALL perform
  hybrid retrieval combining sparse lexical search (BM25) and dense vector
  search (cosine similarity over embeddings).
- **FR-061**: THE SYSTEM SHALL use an HNSW (ANN) index on the vector
  column to serve dense similarity search at production scale/latency.
- **FR-062**: THE SYSTEM SHALL fuse BM25 and dense search results (e.g. via
  Reciprocal Rank Fusion) before optional reranking.
- **FR-063**: THE SYSTEM SHALL apply a reranking step (cross-encoder or
  LLM-based) to the fused candidate set before selecting final context for
  generation.
- **FR-064**: THE SYSTEM SHALL enforce the access-control filter (FR-004)
  as part of the retrieval query (not as a post-filter that leaks
  existence/count information).
- **FR-065**: THE SYSTEM SHALL apply a minimum relevance-score threshold
  below which retrieved chunks are discarded rather than passed to the
  LLM.

## FR Group H — Generation / Orchestration (LangGraph)

- **FR-070**: THE SYSTEM SHALL implement the RAG pipeline as a LangGraph
  graph with distinct nodes for: query understanding/rewrite, retrieval,
  relevance grading, generation, and post-generation verification
  (groundedness/guardrail check).
- **FR-071**: IF the relevance-grading node determines retrieved context is
  insufficient, THEN THE SYSTEM SHALL route to a query-rewrite/retry node
  (bounded retry count) before falling back to a "not enough information"
  response.
- **FR-072**: THE SYSTEM SHALL support tool-calling within the graph (e.g.
  structured lookup tools) in addition to pure document retrieval.
  (Revised in v0.2: the org/people-data tools in FR Group L are the
  concrete v1 use of this, not a deferred v2 placeholder — see
  [00-vision-and-scope.md](00-vision-and-scope.md) §6 and
  [08-generation-spec.md](08-generation-spec.md) §8.)
- **FR-073**: THE SYSTEM SHALL log every graph node's inputs/outputs
  (with PII redaction where applicable) to LangSmith for tracing and
  evaluation.

## FR Group I — Guardrails, Security, Safety

- **FR-080**: THE SYSTEM SHALL screen incoming user messages for
  prompt-injection patterns (e.g. instructions attempting to override the
  system prompt, exfiltrate the system prompt, or cause the agent to
  ignore retrieval grounding) before they reach the generation node.
- **FR-081**: THE SYSTEM SHALL screen outgoing responses for policy
  violations (PII leakage, unsafe content, contradiction of source
  documents) before returning them to the user.
- **FR-082**: IF a prompt-injection or policy-violation attempt is
  detected, THEN THE SYSTEM SHALL block or sanitize the request/response,
  log the event, and SHALL NOT silently comply.
- **FR-083**: THE SYSTEM SHALL never reveal its system prompt, internal
  tool definitions, or retrieval implementation details in response to a
  user request to do so.

## FR Group J — Administration & Ingestion Management

- **FR-090**: THE SYSTEM SHALL provide an admin interface/API for content
  owners to upload, re-upload, tag (access control), and delete source
  documents.
- **FR-091**: THE SYSTEM SHALL show ingestion status (queued, processing,
  indexed, failed) per document, with error detail on failure.
- **FR-092**: THE SYSTEM SHALL allow an admin to view which chunks/pages
  were extracted from a given source document, for QA purposes.

## FR Group K — Evaluation Hooks

- **FR-100**: THE SYSTEM SHALL support running a golden Q&A dataset through
  the full pipeline in an offline/CI context and produce per-question and
  aggregate metrics (see [12-evaluation-spec.md](12-evaluation-spec.md)).
- **FR-101**: THE SYSTEM SHALL tag production traces sent to LangSmith with
  environment, version, and graph-node metadata sufficient to reproduce an
  evaluation run.

## FR Group L — Org & People Data (MCP Tools)

Added in v0.2, tracing to [00-vision-and-scope.md](00-vision-and-scope.md)
G6 and [01-business-requirements.md](01-business-requirements.md) BO-07.
**Source system resolved v0.3**: for v1, "the HRIS/People system" below
refers to a placeholder People-Data Service we build ourselves
(deployed on ECS, seeded with mock data — see
[04-architecture.md](04-architecture.md) §2.12), not a real vendor
system. FRs are still written generically against "the HRIS/People
system" rather than naming the placeholder explicitly, since that's
exactly what keeps a future real-vendor swap from requiring any FR
rewrites.

- **FR-110**: THE SYSTEM SHALL expose employee/org lookups (role, manager,
  department, org hierarchy) to the generation graph via **read-only**
  MCP tool calls against the HRIS/People system of record — never via an
  ingested or cached copy of that data (avoids the staleness problem
  ingested-document retrieval would have for data that changes as often
  as org structure does).
- **FR-111**: WHEN a user asks about another employee's manager, team, or
  position in the org hierarchy, THE SYSTEM SHALL answer using a live MCP
  tool call for that specific request, not a previously cached answer
  from an earlier session or another user's query.
- **FR-112**: THE SYSTEM SHALL expose training-status and
  certification-record lookups via the same MCP tool integration, scoped
  read-only, subject to the same per-employee visibility default as
  FR-110 (self and immediate manager; broader HR visibility per FR-114's
  pattern) pending confirmation of the exact policy.
- **FR-113**: THE SYSTEM SHALL expose performance/360 review feedback
  lookups via MCP tool call, gated per FR-114.
- **FR-114**: WHEN a user requests performance/360 review data about an
  employee, THE SYSTEM SHALL return that data only if the requesting user
  is the subject employee, the subject's **immediate manager** (not the
  full reporting chain — a skip-level manager is not entitled by virtue
  of being upstream in the hierarchy), or a member of HR — and SHALL
  otherwise decline, regardless of how the request is phrased (BR-09,
  **resolved** [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q8).
  This check SHALL happen at the tool-call layer (the tool itself
  refuses/redacts), not only as a system-prompt instruction to the LLM —
  an instruction alone is not a security boundary (see
  [11-security-spec.md](11-security-spec.md) for the general
  guardrail-vs-prompt-instruction distinction this follows).
- **FR-115**: IF an MCP tool call to the HRIS/People system fails, times
  out, or returns no matching record, THEN THE SYSTEM SHALL state that it
  doesn't have that information rather than fabricating an answer from
  general knowledge or a stale assumption (BR-12).
- **FR-116**: THE SYSTEM MAY expose family/dependent information **only
  when the underlying data is synthetic/mocked** (v1's placeholder
  People-Data Service, never a real HRIS record — see
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q9, resolved
  v0.3). Gated by the same entitlement rule as FR-114/FR-118 (self,
  immediate manager, HR). THE SYSTEM SHALL NOT expose **real** employee
  family/dependent data through any tool, response, or fallback path —
  enforced structurally by there being no real-data source connected in
  v1, not by relying on the model to decline. Before a real HRIS is ever
  integrated (Q7), this FR SHALL be revisited and SHALL NOT be treated as
  pre-cleared by the fact that a mocked version was built — Legal/
  Compliance sign-off on the real data is a separate, still-unresolved
  question.
- **FR-117**: THE SYSTEM SHALL expose salary/compensation lookups
  (current salary, hike/increase history) via MCP tool call, gated per
  FR-118.
- **FR-118**: WHEN a user requests salary/compensation data about an
  employee, THE SYSTEM SHALL return that data only if the requesting user
  is the subject employee, the subject's immediate manager, or a member
  of HR — and SHALL otherwise decline (BR-13). Same enforcement posture
  as FR-114 (tool-call layer, not prompt instruction) and same
  non-leaking-refusal requirement: a decline SHALL read identically
  regardless of what the underlying data is, so the refusal itself
  cannot be used to infer, e.g., whether a raise occurred (see
  [00-vision-and-scope.md](00-vision-and-scope.md) §10 Scenario C).

## FR Group M — Social Layer

Added in v0.2, tracing to [00-vision-and-scope.md](00-vision-and-scope.md)
G7 and [01-business-requirements.md](01-business-requirements.md) BO-08.
**Moderation/feed scope resolved v0.3** ([00-vision-and-scope.md](00-vision-and-scope.md)
§8 Q10) to a deliberately minimal default: report-only moderation
(FR-123, manual review, no automated content moderation beyond the
shared guardrail screening in NFR-093) and org-wide feed visibility
(FR-126). Treated as a starting point, not a finished moderation
program — see Q10's resolution note for what would trigger revisiting
it.

- **FR-120**: THE SYSTEM SHALL let a user create and edit their own
  employee profile, combining read-only fields sourced live from the
  HRIS/People system (FR-110) with user-editable fields (bio, contact
  preferences) that are native to this system, not written back to the
  HRIS.
- **FR-121**: THE SYSTEM SHALL let a user create posts and comment on
  posts, visible to other employees in a shared internal feed.
- **FR-122**: THE SYSTEM SHALL let a user send and receive direct
  messages with other employees.
- **FR-123**: THE SYSTEM SHALL let a user report another user's post,
  comment, or message for moderation review. (Exact moderation workflow —
  auto-hide pending review vs. stays visible, reviewer role — is open per
  Q10; this FR covers only the report action itself.)
- **FR-124**: THE SYSTEM SHALL let a user delete their own posts,
  comments, and messages.
- **FR-125**: THE SYSTEM SHALL log all moderation actions (report,
  content removal, user block) to the immutable audit log (extends
  NFR-033's document/ACL audit scope to social content).
- **FR-126**: Feed visibility SHALL be enforced at query time, not
  applied as a client-side filter over an unrestricted read — **confirmed
  v0.3 default** (Q10) is org-wide visibility to all authenticated
  employees with no narrower scoping; if a department- or follow-based
  scoping is adopted later instead, only the feed query's filter
  predicate changes, not the storage model (see
  [05-domain-model.md](05-domain-model.md)).

---

## Traceability

Each FR above will be referenced by:
- Architecture/component specs (04–10) describing how it is implemented.
- Test cases in [15-testing-strategy.md](15-testing-strategy.md).
- Golden dataset categories in [12-evaluation-spec.md](12-evaluation-spec.md)
  where applicable (answer-quality-related FRs).
