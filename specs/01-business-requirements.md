# 01 — Business Requirements

## Status
Draft v0.1 — 2026-08-17

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

## 2. Stakeholders

| Stakeholder | Interest |
|---|---|
| HR leadership | Accurate policy answers, reduced ticket load, audit trail |
| IT leadership | Security posture, SSO integration, infra ownership |
| Legal/Compliance | No hallucinated legal/compliance claims, access control, audit logging, data retention |
| Employees | Fast, trustworthy self-service answers |
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

Out of scope (v1):
- Individual employee records / HRIS data (PII-heavy transactional data)
- Legally binding contracts requiring signature workflows
- Real-time/ephemeral Slack or email content

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

## 5. Success Criteria (business-level acceptance)

- HR/IT/Legal content owners sign off that a representative sample of golden
  Q&A pairs (see [12-evaluation-spec.md](12-evaluation-spec.md)) are
  answered correctly and with correct citations.
- Security/Compliance signs off on the access-control model and guardrail
  test results (see [11-security-spec.md](11-security-spec.md)).
- The system passes a staging soak test and is deployed to prod via the
  CI/CD pipeline with no manual steps (see
  [14-infra-and-cicd-spec.md](14-infra-and-cicd-spec.md)).

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
