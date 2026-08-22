# 07 — Retrieval Spec

## Status
Draft v0.1 — 2026-08-18

Traces from: [02-functional-requirements.md](02-functional-requirements.md)
(FR Group G), [03-non-functional-requirements.md](03-non-functional-requirements.md)
(NFR-001–005, NFR-022), [04-architecture.md](04-architecture.md) §2.5,
[05-domain-model.md](05-domain-model.md) §2, [06-ingestion-spec.md](06-ingestion-spec.md)

---

## 1. Purpose & Scope

Defines how a query string is turned into a ranked, access-filtered set of
chunks for generation. Scoped to retrieval mechanics only — **query
rewriting from conversation history (FR-013) happens upstream**, in the
LangGraph query-understanding node (see
[08-generation-spec.md](08-generation-spec.md)); this spec's input is
already a single, finalized query string. Does not cover generation,
groundedness verification, or guardrails (see
[08-generation-spec.md](08-generation-spec.md),
[11-security-spec.md](11-security-spec.md)).

## 2. Query Interface

Exposed to the LangGraph retrieval node as an in-process call within the
chat backend service (per [04-architecture.md](04-architecture.md) §2.3 —
not a separate network hop):

| Input | Notes |
|---|---|
| `query_text` | already rewritten/standalone (FR-013 handled upstream) |
| `user_access_tags` | resolved at session start (FR-003) |
| `top_k_final` | chunks to return post-rerank (default: see §6) |

| Output | Notes |
|---|---|
| ranked `Chunk[]` | each with `chunk_id`, `text`, `score`, `document_id`, `section_path`/`page_number` or timestamp range, `source_uri` — i.e. everything FR-011 citations need |

## 3. Pipeline Overview

```
query_text
  → embed (OpenAI, same model as ingestion — §4.1)          ─┐
  → dense ANN search (pgvector HNSW, top N_dense)            ├─ parallel
  → sparse lexical search (Postgres tsvector, top N_sparse)  ─┘
  → Reciprocal Rank Fusion → fused candidate list (top N_fused)
  → rerank (cross-encoder) → reranked list
  → relevance threshold cutoff
  → top_k_final chunks returned
```

Access-control filtering (§6) is embedded in the dense and sparse search
queries themselves (stage 2/3), not applied as a later stage — per FR-064
this must not be a post-filter.

## 4. Dense Vector Search (FR-061, NFR-004)

### 4.1 Query embedding
`query_text` is embedded with the same model as ingestion
(`text-embedding-3-large`, per [06-ingestion-spec.md](06-ingestion-spec.md)
§7) through the shared `EmbeddingProvider` abstraction — a query embedded
with a different model than the corpus would make cosine distance
meaningless.

### 4.2 ANN search
- Index: pgvector `HNSW` on `documents_chunks.embedding`
  (`idx_chunks_embedding_hnsw`, per
  [05-domain-model.md](05-domain-model.md) §2), `vector_cosine_ops`.
- Build-time params (starting defaults, to be validated against real
  corpus + eval set per §9): `m = 16`, `ef_construction = 64`. `m` trades
  index size/build time for recall; `64` is pgvector's own recommended
  floor for corpora in the low-millions range (NFR-004's ~5M-chunk
  target).
- Query-time param: `ef_search`, set per-request (default `80`) —
  higher recall at higher latency cost; tunable independent of the build
  without a reindex, which is the parameter to adjust first if eval
  recall (NFR-082) comes in low.
- Query shape:
  ```sql
  SELECT chunk_id, text, ..., embedding <=> $query_vec AS distance
  FROM documents_chunks
  WHERE is_deleted = false
    AND access_tags && $user_access_tags
  ORDER BY embedding <=> $query_vec
  LIMIT $n_dense;
  ```
  The `access_tags &&` predicate is in the same query as the ANN
  ordering — see §6 for why, including the recall caveat under
  restrictive ACLs.

## 5. Sparse Lexical Search (FR-060)

- Implementation: Postgres `tsvector`/`tsquery` over
  `documents_chunks.text_tsv` (GIN-indexed, `idx_chunks_tsv_gin`), ranked
  with `ts_rank_cd(..., normalization := 32)` (length-normalizes by
  document size, the closest native approximation to BM25's length
  normalization).
  - Note on terminology: FR-060/domain-model call this "BM25(-style)"
    sparse search; Postgres's own ranking function is not literally BM25.
    If eval results (§9, [12-evaluation-spec.md](12-evaluation-spec.md))
    show this approximation under-serves keyword-heavy queries (policy
    ID lookups, exact clause numbers), swapping in a true-BM25 extension
    (e.g. `pg_search`/ParadeDB) is a contained change — same table, new
    index type, same query contract to §6/§7.
- Same `is_deleted`/`access_tags` predicates as §4.2, same query, ranked
  by `ts_rank_cd` instead of vector distance, `LIMIT $n_sparse`.

## 6. Access Control Filtering (FR-064)

- The ACL predicate (`access_tags && user_access_tags`) lives **inside**
  both the dense and sparse SQL queries (§4.2, §5) — a chunk the user
  can't see is never fetched into the application layer, so there's no
  code path where a count, a rank position, or a "no results" vs.
  "filtered results" distinction could leak the existence of restricted
  content. This is what FR-064 requires ("not ... a post-filter that
  leaks existence/count information").
- Recall caveat: a highly restrictive ACL (small `user_access_tags`
  overlap) shrinks the effective candidate pool before ANN ordering even
  runs, which can reduce recall for pgvector versions without efficient
  filtered-HNSW scanning. Mitigation: `n_dense`/`n_sparse` (§4.2/§5) are
  set generously above `n_fused` (§7) specifically to absorb this, and
  the actual pgvector version in use must support iterative/filtered HNSW
  scans at the target corpus scale — verified as part of §9 load testing,
  not assumed.

## 7. Fusion — Reciprocal Rank Fusion (FR-062)

- Dense results (top `n_dense`) and sparse results (top `n_sparse`) are
  combined via RRF: for each chunk, `score = Σ 1 / (k + rank_i)` over the
  lists it appears in (`rank_i` = 1-based rank in that list, `k = 60`,
  the standard RRF constant — dampens the influence of any single list's
  top rank).
- A chunk appearing in both lists accumulates both terms, naturally
  boosting agreement between lexical and semantic signal.
- Output: fused candidates sorted by RRF score, truncated to `n_fused`
  (default `50`) before reranking.

## 8. Reranking (FR-063)

- Default: a **cross-encoder** reranker (e.g. a `bge-reranker`-family
  model served as an inference endpoint), not an LLM-based reranker —
  chosen for latency: reranking `n_fused = 50` candidates via `n_fused`
  separate OpenAI calls (or one long structured-scoring prompt) cannot
  reliably fit inside the NFR-003 budget (§10), while a cross-encoder
  batch-scores all candidates in one forward pass.
- LLM-based reranking (also permitted by FR-063) remains a config-level
  swap behind the same interface if a future accuracy/latency tradeoff
  favors it — not built as a parallel code path in v1.
- Output: candidates re-sorted by cross-encoder score (roughly
  probability-like, 0–1 after sigmoid).

## 9. Relevance Threshold (FR-065, FR-012)

- After reranking, chunks scoring below a minimum cutoff (starting
  default `0.3`, on the cross-encoder's 0–1 output) are discarded before
  being passed to generation.
- If **all** candidates fall below threshold, retrieval returns an empty
  set — this is the signal the generation graph's relevance-grading node
  (FR-071, [08-generation-spec.md](08-generation-spec.md)) uses to route
  to query-rewrite/retry or the "not enough information" response
  (FR-012).
- The cutoff, along with `ef_search` (§4.2), `n_fused` (§7), and RRF's
  `k` (§7), are the parameters expected to be tuned against the golden
  eval set (NFR-080–084) once it exists — none are load-bearing enough
  to block writing this spec, but none should be treated as final either.

## 10. Performance & Index Tuning (NFR-001–005, NFR-022)

Retrieval's slice of the end-to-end latency budget (NFR-003: p95 400ms
server-side, excluding generation):

| Stage | Budget (p95) |
|---|---|
| Query embedding (OpenAI call) | ~100ms |
| Dense ANN search | ~60ms |
| Sparse search (parallel with dense) | ~30ms |
| Fusion (in-process) | <5ms |
| Rerank (cross-encoder, `n_fused=50`) | ~150ms |
| **Total** | **~345ms**, leaving headroom under 400ms |

- Scalability (NFR-022): HNSW index growth to 10x launch corpus is a
  storage/build-time cost, not an architecture change — `m`/
  `ef_construction` may need to increase at that scale, revisited once
  real corpus growth data exists, consistent with the "tune, don't
  redesign" posture set in [04-architecture.md](04-architecture.md) §2.5.
- Concurrency (NFR-005): retrieval queries are stateless/parallelizable
  across the 200+ concurrent-session target — no session-affinity
  requirement on the Postgres connection pool.

## 11. Open Items

- Reranker model choice (self-hosted cross-encoder vs. a managed
  reranking API) is a candidate ADR — deferred until latency/cost data
  from a real deployment is available, same posture as the OCR/vision
  provider split flagged in
  [06-ingestion-spec.md](06-ingestion-spec.md) §12.
- Filtered-HNSW recall behavior under restrictive ACLs (§6) needs load
  testing with realistic access-tag distributions before the
  `n_dense`/`n_sparse` over-fetch multipliers can be finalized.
- All numeric defaults in this spec (`m`, `ef_construction`, `ef_search`,
  `n_dense`, `n_sparse`, `n_fused`, RRF `k`, relevance cutoff) are
  starting points pending [12-evaluation-spec.md](12-evaluation-spec.md)
  results.

---

## Traceability

| Section | FRs/NFRs |
|---|---|
| §4 Dense search | FR-061, NFR-004 |
| §5 Sparse search | FR-060 |
| §6 Access control | FR-064 |
| §7 Fusion | FR-062 |
| §8 Reranking | FR-063 |
| §9 Relevance threshold | FR-065, FR-012 |
| §10 Performance | NFR-001, NFR-002, NFR-003, NFR-005, NFR-022 |
