# 01 — Business Requirements

## Status
Draft v0.3 — 2026-08-21 (v0.2: scope expansion in
[00-vision-and-scope.md](00-vision-and-scope.md) — employee/org data and
an internal social layer. v0.3: Q2/Q7/Q9/Q10 resolutions — Cognito
groups confirmed, placeholder People-Data Service replaces "real HRIS"
assumption, family data mocked for build only, social moderation
defaults to report-only)

Traces from: [00-vision-and-scope.md](00-vision-and-scope.md)

## 1. Business Objectives

| ID | Objective | Rationale |
|---|---|---|
| BO-01 | Reduce time-to-answer for policy/document questions from minutes/hours (manual search or ticket) to seconds | Direct productivity gain, primary value driver |
| BO-02 | Reduce inbound ticket volume to HR/IT/Legal for repetitive questions | Frees up headcount for higher-value work |
| BO-03 | Ensure answers are consistent and traceable to an authoritative source | Reduces risk of employees acting on outdated/incorrect policy info |
| BO-04 | Keep the corpus current without manual re-indexing overhead | Policies change often; content owners must be able to re-upload cheaply |
| BO-05 | Deploy and operate the system at production-grade reliability and security bar | This is an internal-facing but compliance-sensitive system (HR/legal content) |
| BO-06 | Keep OpenAI API and AWS infra spend predictable and visible | Cost governance for a system with per-query LLM cost |
| BO-07 | Let employees find accurate, current org/people context (manager, team, hierarchy) without leaving the chat | Reduces reliance on separate directory tools; answers stay current by querying the system of record live rather than a stale copy |
| BO-08 | Improve internal employee connectivity via a full internal social layer (profiles, feed, posts, comments, direct messaging) | Gives employees one place for both information lookup and colleague interaction — value driver distinct from BO-01–07, tracked separately (see §5) |

## 2. Stakeholders

| Stakeholder | Interest |
|---|---|
| HR leadership | Accurate policy answers, reduced ticket load, audit trail; now also steward of what people-data categories are safe to expose (BR-09, BR-10) |
| IT leadership | Security posture, SSO integration, infra ownership |
| Legal/Compliance | No hallucinated legal/compliance claims, access control, audit logging, data retention; **new and elevated concern**: family/dependent data exposure (BR-10) and performance-review access control (BR-09) |
| Employees | Fast, trustworthy self-service answers; accurate org/people lookups; ability to connect with colleagues |
| Platform/Engineering team | Buildable, maintainable, observable system within timeline/budget |

## 3. Scope of Content (v1)

In scope:
- HR policy documents (PDF/DOCX)
- IT policy and security policy documents
- Benefits guides
- SOPs and internal process documents
- Compliance/legal reference documents (non-contractual, informational)
- Slide decks (e.g. town hall decks) — text + embedded images
- Training/town-hall videos (audio transcript + visual keyframes)
- Scanned/image-based policy pages (requires OCR)
- **Employee/org data, read-only, via live MCP tool calls to the
  People-data source of record** (revised from v0.1, which excluded this
  category entirely; source is a placeholder service we build for v1,
  not a real HRIS — see [00-vision-and-scope.md](00-vision-and-scope.md)
  §7/§8 Q7): role, manager, department, org hierarchy, training status,
  certification records, performance/360 review feedback (gated per
  BR-09), salary/compensation data (gated per BR-13), and mocked
  family/dependent fields (BR-10, build-only). Not ingested/indexed like
  documents — queried live, so no staleness/re-sync problem, regardless
  of whether the source is today's placeholder or a future real HRIS.
- **Internal employee social content**: profiles, a shared feed, posts,
  comments, and direct messages between employees — full social
  interaction, not a directory-only subset — natively created and stored
  by this system (no external system of record), unlike everything else
  in this list.

Out of scope (v1):
- **Modifying** any HRIS/LMS/performance-management system-of-record data
  — all employee/org/training/review data is read-only through the bot;
  edits happen only in the source system.
- **Real** family/dependent information — buildable now only against
  mocked/synthetic data (see the in-scope bullet above and BR-10); real
  employee family/dependent data stays out of scope until
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 open question 9's
  legal-basis question is resolved with actual Legal/Compliance sign-off.
- Legally binding contracts requiring signature workflows.
- Real-time/ephemeral Slack or email content.

## 4. Business Rules

| ID | Rule |
|---|---|
| BR-01 | The bot must only answer using retrieved, ingested content — it must not answer from the LLM's general world knowledge for policy-specific questions. |
| BR-02 | Every substantive answer must include citations to the source document(s) (title, section/page, link). |
| BR-03 | If no sufficiently relevant content is retrieved, the bot must say it doesn't know rather than guessing. |
| BR-04 | Documents with restricted access (e.g. manager-only compensation bands) must only be retrievable by users with the corresponding role/permission. |
| BR-05 | A document re-uploaded with identical content must not create duplicate index entries (content-hash dedup). A document re-uploaded with partial changes must only re-embed the changed sections. |
| BR-06 | All conversations are logged for audit and quality-evaluation purposes, subject to the company's data retention policy. |
| BR-07 | Users can see and manage (rename/delete) their own chat sessions; users cannot see other users' chat sessions. |
| BR-08 | The system must degrade gracefully if the LLM provider (OpenAI) is unavailable or rate-limited (clear error, retry/backoff, no silent failure). |
| BR-09 | Performance/360 review feedback about an employee is only visible, via the bot, to that employee themselves, their **immediate manager** (not the full reporting chain — a skip-level manager has no more access than any other employee), and HR — never to an arbitrary employee querying about a peer. Enforcement must happen at query time (mirroring BR-04's document-ACL rigor for FR-004/FR-064), not as an app-level convention that a prompt could be talked around. **Resolved 2026-08-21** — see [00-vision-and-scope.md](00-vision-and-scope.md) §8 Q8, §10. |
| BR-10 | Family/dependent information: **buildable now against mocked/synthetic data only** (v1 has no real HRIS integration — see [00-vision-and-scope.md](00-vision-and-scope.md) §7/§8 Q7/Q9), but real employee family/dependent data must not be exposed via the bot in any form until Legal/Compliance sign-off exists for a specific field set and legal basis. Mocking the data resolves the *build* blocker only — it does not resolve, and must not be read as resolving, the legal question. This is a hard gate on real data, not a default-open item like BR-04's ACL defaults. |
| BR-11 | Social content (posts, comments, direct messages) is user-generated, subject to moderation, and logged/retained under the same audit discipline as chat content (BR-06) — a DM is not exempt from the retention/audit posture just because it isn't a policy Q&A. |
| BR-12 | The bot must not fabricate or infer org/people data (a manager, a team assignment) when the live HRIS lookup fails or returns nothing — same "say it doesn't know" posture as BR-03, applied to people data, not just documents. |
| BR-13 | Salary/compensation data (current salary, hike/increase history) about an employee is only visible, via the bot, to that employee themselves, their immediate manager, and HR — same entitlement model and same query-time enforcement posture as BR-09, never to an arbitrary employee. A denial must not itself leak information (e.g. the refusal must read identically whether or not the subject actually received a hike) — see [00-vision-and-scope.md](00-vision-and-scope.md) §10 Scenario C. |

## 5. Success Criteria (business-level acceptance)

- HR/IT/Legal content owners sign off that a representative sample of golden
  Q&A pairs (see [12-evaluation-spec.md](12-evaluation-spec.md)) are
  answered correctly and with correct citations.
- Security/Compliance signs off on the access-control model and guardrail
  test results (see [11-security-spec.md](11-security-spec.md)) —
  **explicitly including** the performance-review access model (BR-09),
  the salary/compensation access model (BR-13), and the family-data gate
  (BR-10) as separate sign-off line items, not folded into general
  document-ACL sign-off.
- The system passes a staging soak test and is deployed to prod via the
  CI/CD pipeline with no manual steps (see
  [14-infra-and-cicd-spec.md](14-infra-and-cicd-spec.md)).
- Social-feature adoption/engagement numeric targets are still deferred
  (no baseline usage data exists yet to set them against) — but the
  feature boundary itself is no longer undefined:
  [00-vision-and-scope.md](00-vision-and-scope.md) §8 open question 10
  resolved to a minimal report-only moderation default and org-wide feed
  visibility, so this is a "no target number yet" gap, not a "scope
  undefined" gap.

## 6. Assumptions Requiring Business Sign-off

1. Employees authenticate via existing corporate SSO — no separate account
   system.
2. Content owners (HR/IT/Legal) are responsible for uploading and
   maintaining source-of-truth documents through an admin ingestion
   interface (or a sync connector, TBD per open question in vision doc).
3. The bot is advisory only; for legally binding decisions employees are
   directed to the authoritative document or a human contact — this will be
   reflected in system prompt behavior and UI disclaimers.
4. Retention period for chat logs will follow the company's existing data
   retention policy (default assumption: 12 months, unless corrected).
5. **Revised, resolved 2026-08-21**: for v1, employee/org data (role,
   manager, hierarchy, training, certifications, reviews, salary) is
   **not** sourced from a real HRIS — it's served by a placeholder
   People-Data Service this team builds and seeds with mock data (see
   [00-vision-and-scope.md](00-vision-and-scope.md) §7/§8 Q7). PolicyBot
   is therefore, for now, the de facto system of record for this mock
   data, not a read-surface onto something else. A real HRIS integration
   is future work, deferred to when a vendor is actually selected.
6. **Revised, resolved 2026-08-21**: family/dependent data may be
   **mocked** for build/demo purposes under assumption 5 above — this
   unblocks development but is explicitly not a substitute for
   Legal/Compliance sign-off (BR-10), which remains required before any
   *real* employee family/dependent data is ever connected.
