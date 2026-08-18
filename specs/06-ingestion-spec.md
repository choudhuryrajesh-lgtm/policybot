# 06 — Ingestion Spec

## Status
Draft v0.1 — 2026-08-18

Traces from: [02-functional-requirements.md](02-functional-requirements.md)
(FR Groups E, F, J), [03-non-functional-requirements.md](03-non-functional-requirements.md)
(NFR-013, NFR-021), [04-architecture.md](04-architecture.md) §2.4,
[05-domain-model.md](05-domain-model.md)

---

## 1. Purpose & Scope

Defines how raw files landing in the S3 ingestion bucket become retrievable,
access-controlled chunks in Postgres/pgvector plus status/metadata in
DynamoDB. Covers: per-file-type extraction, content hashing/dedup/diff,
chunking, embedding, indexing, access-tag propagation, deletion/tombstoning,
failure isolation, and status reporting. Does not cover retrieval-time
query behavior (see [07-retrieval-spec.md](07-retrieval-spec.md)) or the
admin UI itself (see [04-architecture.md](04-architecture.md) §2.1).

## 2. Pipeline Overview

One Step Functions execution per source file (not per batch) — this is
what makes NFR-013 (one document's failure doesn't block others) hold
structurally rather than by convention.

```
S3 PutObject (/uploads/* or /sync/*)
  → EventBridge rule → Step Functions StartExecution (per object)
    1. Classify        — file extension + content sniff → route
    2. Extract         — §3, per file type
    3. Hash             — §4, whole-doc + per-chunk
    4. Dedup/Diff check — §4, against DynamoDB DocumentMetadata
       ├─ identical whole-doc hash → mark no-op, END (FR-042)
       └─ new or changed → continue
    5. Chunk            — §5, context-aware, only changed regions re-chunked
    6. Embed            — §7, OpenAI embeddings, batched
    7. Index            — §7, Postgres upsert, then DynamoDB status flip
    8. Emit status event — §10, consumed by admin UI
```

Stage compute: stages 1–5 and small-file stage 6/7 run on Lambda; video
transcription/keyframe-vision (stage 2 for `video`) runs on ECS/Batch
(exceeds Lambda's 15-minute limit for longer recordings) — per
[04-architecture.md](04-architecture.md) §2.4.

## 3. File-Type Processing (Stage 2: Extract)

### 3.1 Text documents — PDF, DOCX, PPTX, TXT/MD (FR-040)

- Parsed with a structure-aware library (e.g. `unstructured`) that yields
  typed elements (`Title`, `Heading`, `NarrativeText`, `Table`,
  `ListItem`, `Image`) with a heading hierarchy per element — this
  hierarchy is what powers `section_path` (FR-045) and the chunk
  boundaries in §5.
- Tables are extracted as structured text (row/column-preserving,
  markdown-table serialization) rather than flattened prose, since
  policy/benefits tables (e.g. PTO accrual tiers) lose meaning if
  flattened.
- Any `Image` elements encountered during parsing are routed to §3.2
  rather than dropped.

### 3.2 Embedded images within documents (FR-051)

- Each embedded image is cropped to its own file, uploaded to
  `/derived/{document_id}/{version_hash}/images/{image_id}.png`, and run
  through the same OCR + captioning steps as §3.3.
- The resulting OCR text + caption is **appended to the surrounding
  document chunk's text** (the chunk containing the image's anchor
  position in reading order), not stored as a standalone chunk — this is
  what FR-051 means by "associate ... with the surrounding document
  chunk for context." `source_uri` on that chunk gains a reference to the
  cropped image alongside the page/section location.

### 3.3 Standalone images (FR-050)

- OCR: AWS Textract (`DetectDocumentText` for typed policy pages,
  `AnalyzeDocument` with `TABLES`/`FORMS` when the image is a scanned
  form) — chosen over a vision-LLM for pure text extraction because it's
  purpose-built, cheaper per page, and doesn't consume OpenAI budget for
  what is fundamentally OCR, not reasoning.
- Captioning: OpenAI vision model, prompted to describe diagrams/charts
  in policy-relevant terms (what the diagram conveys, not a generic
  visual description).
- Output: one chunk with `source_type = image`, `text` = OCR text +
  caption (concatenated, caption first), `source_uri` = the image's S3
  location.
- This provider split (Textract for OCR, OpenAI for captioning) is a
  candidate ADR — see §12.

### 3.4 Video (FR-052, FR-053)

- **Transcript**: OpenAI Whisper API on the extracted audio track,
  producing timestamped segments.
- **Keyframes**: scene-change detection (ffmpeg/PySceneDetect) rather
  than fixed-interval sampling, so keyframes align with actual visual
  changes (new slide, new speaker) instead of arbitrary time slices; each
  keyframe captioned via the same OpenAI vision step as §3.3.
- **Chunking**: transcript segments are grouped into chunks by time
  window (target ~60–90s of speech per chunk, snapped to sentence
  boundaries), each chunk carrying `timestamp_start_ms`/`timestamp_end_ms`
  (FR-053) so citations deep-link to the moment. A keyframe caption
  falling within a transcript chunk's time range is appended to that
  chunk's text, same pattern as §3.2.

## 4. Content Hashing, Dedup, and Diff (FR-041–043, BR-05)

- **Document hash**: SHA-256 over the raw uploaded file bytes → stored as
  `document_version_hash`.
- **Chunk hash**: SHA-256 over the chunk's normalized text (whitespace
  collapsed, trimmed) → stored as `chunk_hash`.
- **Dedup (FR-042)**: before extraction, compute the document hash and
  check it against the existing `DocumentMetadata` `LATEST` item for that
  `document_id`. Identical → skip stages 2–7 entirely, mark the upload
  no-op, emit a status event so the admin UI reflects "no changes
  detected" rather than leaving it looking stuck.
- **Diff (FR-043)**: on a changed hash, extraction and chunking (§5)
  still run in full, but the **hash-set diff** happens before embedding:
  1. Load the previous version's `{chunk_hash → chunk_id}` set from
     Postgres (`WHERE document_id = ? AND is_deleted = false`).
  2. For each new chunk: if its `chunk_hash` exists in the previous set,
     **reuse** — update that row's `document_version_hash`,
     `chunk_index`, `section_path` (position/context may have shifted)
     in place; embedding is untouched.
  3. Chunk hashes present only in the new set → insert + embed (new or
     modified content).
  4. Chunk hashes present only in the old set → tombstone
     (`is_deleted = true`).
  - This is a content-addressed diff, not a positional one — a single
    edited paragraph early in a document doesn't invalidate every
    downstream chunk's embedding just because their `chunk_index` shifted.
  - `chunk_id` is generated deterministically as `UUIDv5(document_id +
    chunk_hash)` rather than randomly, so this matching is a plain lookup
    and re-running the same job twice (retry after a crash) is a no-op
    upsert, not a duplicate insert.
- Image and video source files use the same whole-file hash for dedup
  (FR-054); chunk-level diffing still applies to their derived text
  chunks the same way.

## 5. Chunking Strategy (FR-044, FR-045)

- Primary strategy: **structure-aware chunking** using the heading
  hierarchy from §3.1 — a chunk never crosses a `Heading`/`Title`
  boundary above a configured level, so chunks align with actual
  section/paragraph structure rather than arbitrary character counts.
- Target chunk size ~300–800 tokens with ~10–15% overlap between adjacent
  chunks within the same section, to preserve local context across chunk
  boundaries for retrieval.
- Fallback to fixed-size recursive splitting only when no structural
  metadata is available (plain `.txt`, or a parse that yields no headings)
  — per FR-044, structure-aware is prioritized whenever the source
  supports it.
- Metadata stamped on every chunk (FR-045): `document_id`, title,
  `section_path` (heading breadcrumb), `page_number` (paged docs) or
  `timestamp_start_ms`/`timestamp_end_ms` (video), `document_version_hash`,
  `access_tags` (§6).

## 6. Access Control Tag Propagation (FR-004, FR-064)

Per the [05-domain-model.md](05-domain-model.md) §4 decision, access tags
are read once at ingestion time and denormalized onto each Postgres chunk
row rather than joined at query time.

- On every ingestion run (new doc or new version), `access_tags` is read
  from the `DocumentMetadata` item being written and copied onto every
  chunk row produced in that run (new inserts and hash-reused rows alike
  — reuse in §4 step 2 still refreshes `access_tags`, in case tags
  changed alongside content).
- **ACL-only change** (admin retags a document's access without
  re-uploading content) is a distinct, lighter-weight path: it does not
  go through extraction/chunking/embedding at all. It runs a targeted
  `UPDATE documents_chunks SET access_tags = ? WHERE document_id = ? AND
  is_deleted = false`, then appends an `AuditLog` entry
  (`action = acl.change`) per NFR-033. Triggered from the admin API
  directly (S3 event path doesn't apply since no file changed).

## 7. Embedding & Indexing (Stages 6–7)

- Embedding model: OpenAI `text-embedding-3-large` (per
  [00-vision-and-scope.md](00-vision-and-scope.md) §7), called through the
  `EmbeddingProvider` abstraction (per
  [04-architecture.md](04-architecture.md) §2.7) so retries/backoff
  (NFR-011) and cost accounting (NFR-052) are handled consistently with
  the chat path.
- Batched: chunks needing embedding (§4 step 3) are submitted in batches
  to the embeddings API rather than one call per chunk, bounded by the
  provider's per-request token/item limits.
- **Write ordering** (per [05-domain-model.md](05-domain-model.md) §4):
  Postgres chunk/embedding upserts complete first; only after that
  succeeds does the `DocumentMetadata` item's `status` flip to `indexed`
  (and the `LATEST` pointer move to this version) in DynamoDB. A crash
  between these two steps leaves the document `processing` — safe,
  because the previous version's `LATEST` pointer (and its chunks) are
  still what retrieval serves. Retrying the job re-runs idempotently
  (§4's deterministic `chunk_id` makes the Postgres upserts safe to
  repeat).

## 8. Deletion & Tombstoning (FR-046)

- Explicit document delete (admin action): tombstone (`is_deleted =
  true`) every non-deleted chunk row for that `document_id`; update
  `DocumentMetadata` status; append an `AuditLog` entry
  (`action = document.delete`, NFR-033).
- Superseded content on re-upload is handled by §4's diff itself
  (old-only chunk hashes get tombstoned as part of the normal ingestion
  run) — there is no separate "supersede" step.
- Retrieval always filters `WHERE is_deleted = false` (see
  [07-retrieval-spec.md](07-retrieval-spec.md)); tombstoned rows are kept,
  not hard-deleted, so audit/rollback investigation remains possible.

## 9. Failure Handling & Isolation (NFR-013, NFR-011)

- Each Step Functions execution is scoped to one source file — a batch
  upload of many documents fans out to many independent executions, so
  one document's extraction/embedding failure has no code path that can
  affect another's.
- Per-stage retry with exponential backoff, bounded attempt count,
  applied especially at the embedding call (OpenAI rate limits, ties to
  NFR-011) and any OCR/vision/ASR calls.
- On exhausting retries at any stage: `DocumentMetadata.status = failed`,
  `error_detail` populated with the failing stage and error summary,
  status event emitted (§10). No partial chunks are left indexable — the
  write-ordering in §7 guarantees a failed run never flips `status` to
  `indexed`.

## 10. Status Tracking & Admin Visibility (FR-091, FR-092)

- `DocumentMetadata.status` (`queued` \| `processing` \| `indexed` \|
  `failed`) reflects document-version-level state; `IngestionStatus`
  (keyed by `JOB#{job_id}`, i.e. the Step Functions execution id) tracks
  job-level stage progress for finer-grained "what's it doing right now"
  display, per [05-domain-model.md](05-domain-model.md) §3.
- Each pipeline stage emits a status event on entry (updates
  `IngestionStatus.stage`); stage 8 in §2 is this emission, not a
  separate side effect.
- FR-092 (admin can view extracted chunks for QA): satisfied directly by
  querying `documents_chunks WHERE document_id = ? AND
  document_version_hash = ? AND is_deleted = false` — no separate
  admin-facing extraction log is needed since the chunk rows themselves
  carry `section_path`/`page_number`/`source_uri` for review.

## 11. Idempotency & Retry Semantics

Summarizing the guarantees established above, since they're what make
retries and re-uploads safe rather than corrupting:

- Deterministic `chunk_id` (§4) → re-running extraction/chunking on the
  same content always upserts the same rows.
- Whole-document hash dedup (§4) → identical re-upload is a cheap no-op,
  not a re-index.
- Postgres-then-DynamoDB write ordering (§7) → a crash mid-run never
  exposes partially-indexed content as `indexed`.
- Per-document execution isolation (§9) → retrying one failed document
  never re-triggers processing on unrelated documents.

## 12. Open Items

- **OCR/vision/ASR provider split** (§3.3–3.4: Textract + OpenAI vision +
  Whisper) is a reasonable default but not yet ratified — candidate for
  an ADR (`ADR-006` in [04-architecture.md](04-architecture.md) §4's
  numbering) once real per-page/per-minute cost data is available to
  compare against an all-OpenAI-vision alternative.
- Depends on the same open item as
  [05-domain-model.md](05-domain-model.md) §5: if access-control source
  of truth turns out to be a separate entitlement system rather than
  Cognito groups, only §6's tag-read step changes (still reads
  `DocumentMetadata.access_tags`, just populated differently upstream).
- Chunk size/overlap targets in §5 are starting parameters, not tuned —
  expect adjustment once retrieval-quality eval data
  ([12-evaluation-spec.md](12-evaluation-spec.md)) is available.

---

## Traceability

| Section | FRs/NFRs |
|---|---|
| §3 Extraction | FR-040, FR-050, FR-051, FR-052, FR-053 |
| §4 Hashing/Dedup/Diff | FR-041, FR-042, FR-043, FR-054, BR-05 |
| §5 Chunking | FR-044, FR-045 |
| §6 Access tags | FR-004, FR-064, NFR-033 (ACL-change audit) |
| §7 Embedding/Indexing | NFR-011, NFR-052 |
| §8 Deletion | FR-046, NFR-033 |
| §9 Failure handling | NFR-013, NFR-011 |
| §10 Status/admin visibility | FR-091, FR-092 |
