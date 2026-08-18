# 05 — Domain Model

## Status
Draft v0.1 — 2026-08-17

Traces from: [04-architecture.md](04-architecture.md)

Storage split (per resolved decision): **Postgres/pgvector** owns
retrieval-time entities (chunks + embeddings). **DynamoDB** owns
operational entities (sessions, messages, memory, document metadata,
ingestion status, audit log). No entity is duplicated across both stores
except a minimal cross-reference key (`document_id`).

---

## 1. Entity Overview

```
User (Cognito) ──1:N──▶ ChatSession ──1:N──▶ Message
User (Cognito) ──1:N──▶ LongTermMemoryRecord
Document ──1:N──▶ DocumentVersion ──1:N──▶ Chunk ──1:1──▶ Embedding
Document ──1:N──▶ AccessControlTag
IngestionJob ──N:1──▶ DocumentVersion
AuditLogEntry (references User, Document, or Session as subject)
```

## 2. Postgres / pgvector Schema

### `documents_chunks`
| Column | Type | Notes |
|---|---|---|
| `chunk_id` | uuid PK | |
| `document_id` | text | FK reference to DynamoDB `DocumentMetadata` PK (no DB-level FK across stores; enforced in app layer) |
| `document_version_hash` | text | whole-document content hash this chunk belongs to (FR-041) |
| `chunk_hash` | text | per-chunk content hash (FR-041, used for diffing on re-upload, FR-043) |
| `chunk_index` | int | ordering within document |
| `text` | text | chunk text (for text/OCR/transcript content) |
| `text_tsv` | tsvector, generated from `text` | GIN-indexed, powers BM25-style sparse search (FR-060) |
| `embedding` | vector(3072) | OpenAI `text-embedding-3-large` dim; HNSW-indexed (FR-061) — dim confirmed in ADR/embedding-model choice |
| `source_type` | text enum | `text` \| `image` \| `video_transcript` \| `video_keyframe` |
| `section_path` | text | heading breadcrumb, e.g. "Benefits > Health > Enrollment" (FR-045) |
| `page_number` | int, nullable | for paged docs |
| `timestamp_start_ms` / `timestamp_end_ms` | int, nullable | for video chunks (FR-053) |
| `source_uri` | text | S3 URI to original file/region (image crop, video segment) |
| `access_tags` | text[] | ACL tags copied from document at index time (FR-004, FR-064) — denormalized for retrieval-time filtering without a join |
| `is_deleted` | boolean | tombstone flag (FR-046) instead of hard delete, to support audit |
| `created_at` / `updated_at` | timestamptz | |

Indexes:
- `idx_chunks_embedding_hnsw` — HNSW on `embedding` (params in
  [07-retrieval-spec.md](07-retrieval-spec.md)).
- `idx_chunks_tsv_gin` — GIN on `text_tsv`.
- `idx_chunks_document_id` — btree, for delete/tombstone-by-document.
- `idx_chunks_access_tags` — GIN on `access_tags` (array containment
  filter, FR-064).

## 3. DynamoDB Tables

Single-table-per-entity design chosen for clarity at this corpus/traffic
scale (launch: ~1,000 docs, low session volume); revisit single-table
design only if item-count/cost modeling in the infra spec calls for it.

### `ChatSessions`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{user_id}` | S | partition key |
| `SK` = `SESSION#{session_id}` | S | sort key |
| `title` | S | user-editable or auto-generated (FR-023) |
| `created_at` / `updated_at` | S (ISO8601) | |
| `status` | S | `active` \| `deleted` |

Access pattern: list all sessions for a user (Query on PK) — supports
FR-020–024, and naturally enforces FR-024 (no cross-user access) since
queries are always scoped to the authenticated user's PK.

### `Messages`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `SESSION#{session_id}` | S | |
| `SK` = `MSG#{ISO8601_timestamp}#{message_id}` | S | sortable by time |
| `role` | S | `user` \| `assistant` |
| `content` | S | |
| `citations` | L (list) | doc/chunk refs shown to user (FR-011) |
| `feedback` | S, nullable | `up` \| `down` (FR-016) |
| `langsmith_trace_id` | S, nullable | for FR-101 traceability |

### `LongTermMemory`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{user_id}` | S | |
| `SK` = `MEMORY#{memory_id}` | S | |
| `summary` | S | salient fact/summary (FR-031) |
| `source_session_id` | S | provenance |
| `created_at` | S | |
| `ttl` | N, nullable | optional expiry per retention policy (NFR-042) |

Supports FR-030–033 directly: list/delete scoped to `USER#{user_id}`.

### `DocumentMetadata`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `DOC#{document_id}` | S | logical document identity, stable across re-uploads |
| `SK` = `VERSION#{document_version_hash}` | S | one item per version (FR-041–043) |
| `title` | S | |
| `source_type` | S | `pdf` \| `docx` \| `pptx` \| `image` \| `video` \| ... |
| `s3_uri` | S | landing-zone location |
| `access_tags` | SS (string set) | source of truth for ACLs, copied into Postgres chunks at index time |
| `status` | S | `queued` \| `processing` \| `indexed` \| `failed` (FR-091) |
| `error_detail` | S, nullable | |
| `uploaded_by` | S | user id or `sync-job` |
| `ingested_at` | S | |

Latest-version lookup: query PK, sort descending, take first — plus a
`LATEST` pointer item (`SK = "LATEST"`) updated on successful ingestion,
so the retrieval-time access-tag lookup doesn't need a scan.

### `IngestionStatus` (job-level, distinct from document version status)
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `JOB#{job_id}` | S | Step Functions execution id |
| `document_id` | S | |
| `stage` | S | current pipeline stage (FR-091) |
| `started_at` / `completed_at` | S | |

### `AuditLog`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `DATE#{yyyy-mm-dd}` | S | time-bucketed partition to avoid hot keys |
| `SK` = `{ISO8601_timestamp}#{event_id}` | S | |
| `actor` | S | user id or system principal |
| `action` | S | e.g. `document.upload`, `document.delete`, `acl.change` |
| `subject` | S | document/session id affected |
| `detail` | M (map) | |

Immutable by convention (no update/delete API exposed) — satisfies
NFR-033.

## 4. Cross-Store Consistency Notes

- `document_id` is the only value shared between Postgres (`chunks.
  document_id`) and DynamoDB (`DocumentMetadata.PK`). No two-phase
  commit — ingestion pipeline writes Postgres chunks/embeddings first,
  then flips DynamoDB `DocumentMetadata` status to `indexed` last, so a
  partial failure leaves a document `processing`/`failed` rather than
  falsely `indexed` (see ingestion spec for exact ordering/retry
  semantics).
- Access tags are **read from DynamoDB at ingestion time and denormalized
  onto Postgres chunk rows** rather than joined at query time — trades a
  small re-index cost on ACL change for fast retrieval-time filtering
  (FR-064 requires filtering to happen *in* the retrieval query).
  Consequence: an ACL change on a document must trigger re-tagging of
  its existing chunks (a targeted UPDATE, not a full re-embed) — captured
  as a requirement in [06-ingestion-spec.md](06-ingestion-spec.md).

## 5. Open Item

Access-control source of truth (Cognito groups vs. a separate
entitlement system) is still unconfirmed (see
[00-vision-and-scope.md](00-vision-and-scope.md) §8). This model assumes
Cognito group membership maps directly to `access_tags` values; if a
separate HRIS/entitlement source is confirmed instead, only the
tag-population step in ingestion/ACL-sync changes — the schema above is
unaffected.
