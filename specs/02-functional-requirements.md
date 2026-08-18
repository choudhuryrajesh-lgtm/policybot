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
  structured lookup tools) in addition to pure document retrieval, as an
  extensibility point for v2.
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

---

## Traceability

Each FR above will be referenced by:
- Architecture/component specs (04–10) describing how it is implemented.
- Test cases in [15-testing-strategy.md](15-testing-strategy.md).
- Golden dataset categories in [12-evaluation-spec.md](12-evaluation-spec.md)
  where applicable (answer-quality-related FRs).
