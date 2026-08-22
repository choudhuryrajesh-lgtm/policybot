# 05 — Domain Model

## Status
Draft v0.3 — 2026-08-21 (v0.2 added §3.1 social-layer tables; explicitly
does **not** add an Employee/Org entity to PolicyBot's own stores — that
data is read live via MCP tools. v0.3 resolves the open questions this
raised: Cognito confirmed as ACL source of truth, feed visibility
confirmed org-wide, and notes the Placeholder People-Data Service's own
separate mock schema)

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

  -- new, v0.2 --
User (Cognito) ──1:1──▶ Profile
User (Cognito) ──1:N──▶ Post ──1:N──▶ Comment
User (Cognito) ──N:N──▶ User (Follows)
User (Cognito) ──1:N──▶ DirectMessage (via DMThread)
Employee/Org data is NOT a local entity — resolved live per-request via
  the MCP Tool Layer against the Placeholder People-Data Service (v1) —
  see §3.1
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

## 3.1 Social Layer Tables (new, v0.2)

**No `Employee`/`Org` entity is defined in PolicyBot's own data stores,
deliberately.** Role, manager, department, org hierarchy, training,
certifications, performance/360 review, salary/compensation, and (mocked)
family/dependent data are all read live via the MCP Tool Layer
([04-architecture.md](04-architecture.md) §2.10) from the Placeholder
People-Data Service (§2.12) — storing a local copy in Postgres/DynamoDB
would recreate exactly the staleness problem G6 exists to avoid, and
would still be true once a real HRIS eventually replaces the placeholder.
The tables below hold only data **native** to PolicyBot (no external
system of record for it).

**Resolved v0.3** ([00-vision-and-scope.md](00-vision-and-scope.md) §8
Q7): the Placeholder People-Data Service has its own minimal schema,
separate from everything in this document — it's a different service's
data, not PolicyBot's. Recorded here only for reference since nothing
else documents it: a single `EmployeeRecord` shape (`employee_id`,
`name`, `role`, `manager_id`, `department`, `org_path`, `training[]`,
`certifications[]`, `reviews[]` (360/performance entries), `salary`
(current + hike history), `family_members[]` (mocked, synthetic only —
see NFR-095)) seeded as static/synthetic data, e.g. a single table in
whatever lightweight store §2.12 uses. This is intentionally not
normalized or indexed to the rigor of `documents_chunks` or the
DynamoDB tables below — it exists to be replaced by a real HRIS schema
later, not to be a long-lived part of this domain model.

### `Profiles`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{user_id}` | S | |
| `SK` = `PROFILE` | S | singleton item per user |
| `bio` | S | user-editable, native to this system |
| `avatar_uri` | S | S3 location |
| `contact_prefs` | M | user-editable |
| `updated_at` | S | |

Role/manager/department are intentionally **not** attributes here —
always fetched live via the MCP Tool Layer when a profile is rendered
(FR-120), so a profile view can never show stale org data.

### `Posts`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{author_id}` | S | author's posts, for "view my posts" |
| `SK` = `POST#{ISO8601_timestamp}#{post_id}` | S | |
| `content` | S | |
| `visibility` | S | `org` (confirmed v0.3 default, FR-126) — narrower scoping (department/follow-based) is additive if adopted later |
| `comment_count` | N | denormalized counter |
| `created_at` | S | |

### `Comments`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `POST#{post_id}` | S | |
| `SK` = `COMMENT#{ISO8601_timestamp}#{comment_id}` | S | |
| `author_id` | S | |
| `content` | S | |

### `Follows`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{follower_id}` | S | |
| `SK` = `FOLLOWS#{followed_user_id}` | S | supports "who do I follow" |
| `created_at` | S | |

A GSI (`GSI1PK = USER#{followed_user_id}`, `GSI1SK = FOLLOWER#{follower_id}`)
supports the reverse "who follows me" query without a table scan.

### `Feed`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `USER#{viewer_id}` | S | the *viewer's* feed, not the author's |
| `SK` = `FEED#{ISO8601_timestamp}#{post_id}` | S | |
| `author_id` | S | |
| `post_id` | S | pointer back to `Posts` — feed items are pointers, not copies, so an edit/delete on the source post doesn't require updating every fanned-out copy |

Populated by a DynamoDB Streams trigger on `Posts` inserts, which fans a
new post out to every follower's `Feed` partition (per
[04-architecture.md](04-architecture.md) §2.11) — eventually consistent
by design.

### `DirectMessages`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `DM#{thread_id}` | S | `thread_id` = sorted, joined pair of the two user ids, so both participants derive the same key |
| `SK` = `MSG#{ISO8601_timestamp}#{message_id}` | S | |
| `sender_id` | S | |
| `content` | S | |

Kept as a distinct table from the chat `Messages` table
([05-domain-model.md](05-domain-model.md) §3 — same doc) to avoid
conflating user-to-bot chat history with user-to-user messaging; they
have different access patterns, retention needs (NFR-094), and
moderation requirements (FR-123).

### `ModerationReports`
| Attribute | Type | Notes |
|---|---|---|
| `PK` = `REPORT#{report_id}` | S | |
| `subject_type` | S | `post` \| `comment` \| `message` |
| `subject_id` | S | |
| `reporter_id` | S | |
| `reason` | S | |
| `status` | S | `open` \| `reviewed` \| `actioned` |
| `created_at` | S | |

Exact review workflow (who reviews, SLA) is open per
[00-vision-and-scope.md](00-vision-and-scope.md) §8 Q10 — this table
supports the report action (FR-123) regardless of how that workflow ends
up defined.

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

## 5. Open Items

- **Resolved v0.3**: access-control source of truth is confirmed as
  Cognito user-pool groups (see
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q2). `access_tags`
  population as described in this document stands as final for v1, not
  a default pending confirmation — may be revisited if Cognito groups
  prove too coarse later, per Q2's resolution note.
- **Resolved v0.3**: `Follows`/`Feed` (§3.1) org-wide feed visibility
  (FR-126's default) is confirmed, not provisional (see
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q10). If a
  narrower visibility model is adopted later, the `Feed` fan-out
  trigger's follower-resolution logic changes, but the table schema
  itself does not.
- **Resolved v0.3 for build purposes**: family/dependent data now has a
  schema — see the `EmployeeRecord.family_members[]` note above — but
  only as **synthetic data in the Placeholder People-Data Service**, a
  different service's schema, not PolicyBot's own. No schema for real
  family/dependent data exists anywhere in this document, and none
  should be added without first resolving the Legal/Compliance question
  in [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q9 (still open
  — mocking the data did not resolve it, see BR-10).
