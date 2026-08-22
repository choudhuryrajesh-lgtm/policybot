# 08 — Generation / Orchestration Spec

## Status
Draft v0.1 — 2026-08-18

Traces from: [02-functional-requirements.md](02-functional-requirements.md)
(FR Groups B, D, H, plus FR-080–083), [03-non-functional-requirements.md](03-non-functional-requirements.md)
(NFR-001–002, NFR-040), [04-architecture.md](04-architecture.md) §2.3, §2.7,
§2.8, [07-retrieval-spec.md](07-retrieval-spec.md)

---

## 1. Purpose & Scope

Defines the LangGraph agent graph that turns a user turn into a streamed,
cited, guardrailed answer: node structure, state schema, retry/routing
logic, streaming/cancellation behavior, long-term memory integration, and
tracing. Retrieval mechanics themselves are out of scope (consumed here
via the interface in [07-retrieval-spec.md](07-retrieval-spec.md) §2);
guardrail *detection logic* (what counts as injection/PII/policy
violation) is out of scope (see
[11-security-spec.md](11-security-spec.md)) — this spec defines where
guardrail checks sit in the graph and what happens on a hit, not how a
hit is detected.

## 2. Graph Overview (FR-070)

```
user_message
  → input_guardrail        (FR-080) ── injection detected → block, log, END
  → query_rewrite           (FR-013) ── uses session history
  → memory_lookup           (FR-032) ── parallel with retrieve; non-blocking
  → retrieve                 (calls 07-retrieval-spec interface)
  → relevance_grade          (FR-071)
       ├─ sufficient ───────────────────────────────┐
       ├─ insufficient, retry_count < max ─┐         │
       │      └──→ query_rewrite (retry)   │         │
       │           (bounded loop)          │         ▼
       └─ insufficient, retry exhausted ─→ not_enough_info (FR-012) → END
                                                       │
                                                       ▼
                                                   generate          (FR-010, FR-011, FR-014, FR-032)
                                                       │
                                                       ▼
                                             output_guardrail        (FR-081, FR-082, FR-083)
                                                       │
                                                       ▼
                                             stream_to_client
```

`memory_summarize` (FR-031) is **not** a node in this per-turn graph — see
§7.

## 3. Node Detail

| Node | Responsibility | Notes |
|---|---|---|
| `input_guardrail` | Screen the raw user message for prompt-injection patterns before anything else runs (FR-080) | Detection logic: [11-security-spec.md](11-security-spec.md). On hit: log event, return a fixed refusal, do not proceed (FR-082) |
| `query_rewrite` | Given current message + session history, produce a standalone query string (FR-013) | Small/fast model — structured rewrite task, not answer quality; on retry (see §5), receives the prior insufficient query + a hint of what was missing |
| `memory_lookup` | Query `LongTermMemory` (`PK = USER#{user_id}`) for entries relevant to the current query (FR-032) | Cheap keyword/recency lookup, not a full semantic search — memory is a small per-user set, doesn't need HNSW; runs in parallel with `retrieve`, joined before `relevance_grade` |
| `retrieve` | Call [07-retrieval-spec.md](07-retrieval-spec.md) §2 interface with the rewritten query + `user_access_tags` | Single call per graph pass (or per retry pass) |
| `relevance_grade` | Classify retrieved chunks as sufficient/insufficient to answer (FR-071) | Small/fast model, structured (sufficient \| insufficient + brief reason); reason feeds the retry's rewrite hint |
| `not_enough_info` | Fixed-shape "I don't have enough information" response, no LLM call | Terminal node — satisfies FR-012/BR-03 without risking a model paraphrasing its way into a soft hallucination |
| `generate` | Produce the answer from retrieved chunks (primary context) + memory context (secondary, lower-priority, FR-032) | Primary generation-quality model; streamed (§6); system prompt enforces BR-01 (grounded-only), FR-083 (never reveal system prompt/tools/retrieval internals) |
| `output_guardrail` | Screen generated output for PII leakage, unsafe content, contradiction of sources (FR-081) before it reaches the client | Detection logic: [11-security-spec.md](11-security-spec.md); runs incrementally, not just once — see §6 |
| `stream_to_client` | Emit guardrail-cleared increments over SSE | — |

## 4. State Schema

LangGraph state object carried across nodes:

| Field | Set by | Notes |
|---|---|---|
| `session_id`, `user_id`, `user_access_tags` | graph entry | from authenticated request context |
| `conversation_history` | graph entry | recent turns from `Messages` (bounded window, not full history) |
| `raw_query` | graph entry | verbatim user input |
| `rewritten_query` | `query_rewrite` | overwritten on retry |
| `memory_context` | `memory_lookup` | list of relevant `LongTermMemory` summaries, possibly empty |
| `retrieved_chunks` | `retrieve` | from [07-retrieval-spec.md](07-retrieval-spec.md) output |
| `relevance_verdict`, `relevance_reason` | `relevance_grade` | drives routing |
| `retry_count` | graph entry / incremented on loop | bounded by `max_retries` (§5) |
| `generation_output` | `generate` | streamed text, accumulated |
| `citations` | `generate` | derived from `retrieved_chunks` actually cited (§6.2) |
| `guardrail_flags` | `input_guardrail` / `output_guardrail` | for logging, not shown to user |
| `trace_id` | graph entry | LangSmith trace id, persisted to `Messages.langsmith_trace_id` |

## 5. Bounded Retry (FR-071)

- `max_retries = 1` (i.e. at most one requery beyond the initial attempt)
  — chosen to bound added latency: each retry pass costs another
  `query_rewrite` + `retrieve` + `relevance_grade` round trip against the
  NFR-002 8s full-answer budget, so an unbounded or high retry count
  risks blowing that budget on genuinely unanswerable questions.
- On retry, `query_rewrite` receives `relevance_reason` from the failed
  attempt as additional context, so the reformulation is informed rather
  than a blind repeat.
- Exhausting retries routes to `not_enough_info`, never to `generate`
  with a `relevance_verdict = insufficient` state — `generate` is only
  ever reached with chunks judged sufficient (or memory-only context is
  explicitly insufficient on its own, per §7's priority rule).

## 6. Streaming & Cancellation (FR-010, FR-015)

### 6.1 Design tension: token-level streaming vs. pre-return output screening
FR-010 requires token-by-token/incremental streaming; FR-081 requires
screening output "before ... returned to the user." Taken literally
together these conflict — a true per-token stream can't be screened
before each token reaches the client. Resolution: **increment-buffered,
guardrail-gated streaming** — `generate` emits at sentence-boundary
granularity (not per-token, not full-response) into an internal buffer;
each completed increment passes through `output_guardrail` before
`stream_to_client` releases it. This keeps the perceived streaming
behavior FR-010 is after (progressive display, not a full-answer wait)
while giving FR-081 a real checkpoint before content reaches the user.
Trade-off: added latency per increment (guardrail check cost) and a
violation caught mid-answer truncates the stream from that point rather
than never having shown anything — both accepted as the practical
resolution; flagged in §9 as needing product/security sign-off on the
UX for a mid-stream truncation.

### 6.2 Citations (FR-011)
`generate`'s system prompt requires inline citation markers tied to
`retrieved_chunks` (e.g. `[1]`, `[2]`) as it writes. At stream completion,
markers actually used are resolved against `retrieved_chunks` metadata
(title, section/page or timestamp range, link) into the `citations` list
persisted on the `Messages` item — chunks retrieved but not cited are not
included, so citations reflect what was actually used, not everything
fetched.

### 6.3 Cancellation (FR-015)
Client-initiated cancel (e.g. `DELETE
/sessions/{id}/messages/{message_id}/generation` or an abort on the SSE
connection) propagates to: (a) the in-flight OpenAI streaming call
(aborted, not awaited to completion), (b) the LangGraph execution (halted
after the current node), (c) a partial `Messages` write with whatever
increments already passed `output_guardrail`, flagged as
`status = cancelled` (avoids the ambiguity of a partial answer looking
like a complete one on resume).

## 7. Long-Term Memory Integration (FR-030–033)

- **Consult** (`memory_lookup`, FR-032): runs every turn, parallel with
  `retrieve`. `memory_context` is passed to `generate` explicitly labeled
  as lower-priority than `retrieved_chunks` in the system prompt —
  memory can supply *user-specific* context (stated role, recurring
  topics) but must never substitute for document grounding on a
  policy-substantive question (BR-01). Memory alone, with no sufficient
  document chunks, still routes to `not_enough_info` (§5) — it is
  additive context for `generate`, not an alternate path around
  `relevance_grade`.
- **Summarization** (FR-031): explicitly **not** part of this per-turn
  graph — running an extra LLM summarization call synchronously would
  add latency for no benefit to the current answer. Instead: an
  async, fire-and-forget job triggered after each assistant turn
  completes (or batched at session end), which summarizes salient new
  facts from the turn and upserts into `LongTermMemory`. Runs outside the
  request/response path, so it has no NFR-001/002 latency impact.
- **View/delete** (FR-033): a plain CRUD read/delete against
  `LongTermMemory` scoped to `PK = USER#{user_id}` — no graph
  involvement, handled directly by the API layer.

## 8. Provider Abstraction & Tool-Calling (FR-072, ADR-004)

- Every node's LLM call goes through the `LLMProvider` interface defined
  in [04-architecture.md](04-architecture.md) §2.7 (OpenAI in v1,
  Ollama-ready) — no node calls the OpenAI SDK directly, so the future
  swap stays a DI/config change.
- FR-072 (tool-calling extensibility): the graph is wired with
  LangGraph's tool-calling support at the `generate` node, but **no
  tools beyond document retrieval are registered in v1** — this is
  documented as the v2 extension point, not built now. Adding a tool
  later means registering it at this node, not restructuring the graph.

## 9. Tracing (FR-073, FR-101)

- Every node is wrapped with LangSmith tracing (inputs/outputs per node),
  with PII redaction applied before logging (NFR-040) — redaction rules
  themselves live in [11-security-spec.md](11-security-spec.md).
- Traces are tagged with environment, deployed version, and graph-node
  metadata (FR-101), and `trace_id` is persisted onto the `Messages` item
  so a production conversation can be pulled back up in LangSmith for
  debugging or fed into an eval run
  ([12-evaluation-spec.md](12-evaluation-spec.md)).

## 10. Open Items

- §6.1's mid-stream-truncation-on-guardrail-hit UX needs explicit
  product/security sign-off — this spec resolves the FR-010/FR-081
  tension technically, but the user-facing behavior (does a truncated
  stream show an error, a redacted continuation, or nothing) is a product
  decision not yet made.
- `max_retries = 1` (§5) and the small/fast vs. primary model split (§3)
  are starting parameters pending latency/quality data from
  [12-evaluation-spec.md](12-evaluation-spec.md), consistent with the
  numeric-defaults posture in [07-retrieval-spec.md](07-retrieval-spec.md)
  §11.
- Bounded conversation-history window size for `query_rewrite` (§4) is
  not yet specified — needs a concrete turn/token cap once typical
  session length data exists.

---

## Traceability

| Section | FRs/NFRs |
|---|---|
| §2–3 Graph structure | FR-070, FR-071, FR-012 |
| §5 Bounded retry | FR-071 |
| §6 Streaming/citations/cancel | FR-010, FR-011, FR-014, FR-015, FR-081 |
| §7 Long-term memory | FR-030, FR-031, FR-032, FR-033 |
| §8 Provider abstraction/tools | FR-072 |
| §9 Tracing | FR-073, FR-101, NFR-040 |
| (guardrail placement, detection deferred) | FR-080, FR-082, FR-083 |
