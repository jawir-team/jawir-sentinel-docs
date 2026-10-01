# JAWIR Sentinel API Contract

**Specification Version:** 1.0  
**Status:** MVP Baseline  
**Base Path:** `/api/v1`

Dokumen ini adalah contract resmi antara:

```text
jawir-sentinel-fe
        ↕
jawir-sentinel-be
```

Frontend dan backend harus mengikuti request, response, enum, error, dan behavior yang didefinisikan di sini.

---

# 1. Authentication

Semua endpoint authenticated menggunakan:

```http
Authorization: Bearer <firebase-id-token>
```

Backend memvalidasi Firebase ID Token dan memetakan Firebase identity ke internal Sentinel user.

Jika token tidak valid:

```http
401 Unauthorized
```

---

## Authorization Model

System role:

```text
USER
ADMIN
```

System role is stored on the internal user and is separate from case workflow roles.

Authorization matrix:

| Capability | USER | ADMIN |
|---|---|---|
| Read units / case types / safe user directory | Yes | Yes |
| Create/update units | No | Yes |
| Create/update users | No | Yes |
| Create case types | No | Yes |
| Create/version/activate policies | No | Yes |
| Create case | Yes | Yes |
| Read case | If participant | Yes |
| Case workflow action | Only assigned case role | Only assigned case role |
| Override SoD / workflow state | No | No |

`ADMIN` alone never grants Maker/Checker/Signer/Executer authority.

---

# 2. Standard Response

## Success

```json
{
  "data": {}
}
```

## Error

```json
{
  "error": {
    "code": "INVALID_STATE_TRANSITION",
    "message": "Case must be in CHECKING state.",
    "details": {}
  }
}
```

## Pagination

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 120
  }
}
```

---

# 3. Core Enum

## Case Status

```text
DRAFT
SUBMITTED
AI_ANALYSIS
CHECKING
SIGNING
EXECUTION
DONE
CLOSED
ESCALATION_REQUIRED
```

## Urgency

```text
LOW
MEDIUM
HIGH
CRITICAL
```

## System Role

```text
USER
ADMIN
```

## Participant Role

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

One active user may hold only one participant role on a case.

## Decision

```text
APPROVE
REJECT
```

## Execution Status

```text
IN_PROGRESS
SUCCESS
BLOCKED
FAILED
```

## Policy Status

```text
DRAFT
ACTIVE
SUPERSEDED
```

## Policy Index Status

```text
NOT_STARTED
PROCESSING
READY
FAILED
```

## Analysis Status

```text
GENERATING
COMPLETED
FAILED
```

## AI Policy Status

```text
POLICY_FOUND
POLICY_PARTIAL
NO_POLICY_FOUND
INSUFFICIENT_EVIDENCE
POLICY_CONFLICT
```

## Verification Status

```text
PASS
PASS_WITH_WARNING
FAIL
```

## Evidence Quality

```text
LOW
MEDIUM
HIGH
```

## Uncertainty

```text
LOW
MEDIUM
HIGH
```

---

# 4. Current User

## GET `/me`

Response:

```json
{
  "data": {
    "id": "uuid",
    "name": "Operations User",
    "email": "ops.user@example.com",
    "system_role": "USER",
    "unit": {
      "id": "uuid",
      "code": "OPS",
      "name": "Operations"
    }
  }
}
```

---

# 5. Units

## GET `/units`

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "code": "OPS",
      "name": "Operations",
      "description": "Financial operations unit"
    }
  ]
}
```

## POST `/units`

Authorization: `ADMIN` only.

Request:

```json
{
  "code": "RISK",
  "name": "Risk Management",
  "description": "Operational risk unit"
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "code": "RISK",
    "name": "Risk Management",
    "description": "Operational risk unit"
  }
}
```

---

# 6. Users

## GET `/users`

Available to any authenticated ACTIVE user because Maker needs a participant directory.

The list response intentionally does **not** expose `firebase_uid`.

Query parameters:

```text
unit_id
status
page
limit
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Risk User",
      "email": "risk.user@example.com",
      "status": "ACTIVE",
      "system_role": "USER",
      "unit": {
        "id": "uuid",
        "code": "RISK",
        "name": "Risk Management"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

## POST `/users`

Authorization: `ADMIN` only.

Request:

```json
{
  "firebase_uid": "firebase-uid",
  "name": "Risk User",
  "email": "risk.user@example.com",
  "unit_id": "uuid",
  "status": "ACTIVE",
  "system_role": "USER"
}
```

## PATCH `/users/{user_id}`

Authorization: `ADMIN` only.

Request:

```json
{
  "name": "Risk Reviewer",
  "unit_id": "uuid",
  "status": "ACTIVE",
  "system_role": "USER"
}
```

---

# 7. Case Types

## GET `/case-types`

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "code": "SETTLEMENT_EXCEPTION",
      "name": "Settlement Exception",
      "description": "Settlement or reconciliation related operational exception"
    }
  ]
}
```

## POST `/case-types`

Authorization: `ADMIN` only.

Request:

```json
{
  "code": "SETTLEMENT_EXCEPTION",
  "name": "Settlement Exception",
  "description": "Settlement or reconciliation related operational exception"
}
```

---

# 8. Cases

## POST `/cases`

Create draft case.

The authenticated creator is automatically assigned as the case's single active `MAKER` and becomes the immutable `owner`.

```text
created_by = authenticated user
owner_id   = authenticated user
MAKER      = authenticated user
```

Maker/owner cannot be replaced in MVP.

Request:

```json
{
  "case_type_id": "uuid",
  "title": "Settlement reconciliation mismatch",
  "description": "37 transactions failed reconciliation.",
  "urgency": "HIGH"
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "case_number": "CASE-2026-000001",
    "status": "DRAFT"
  }
}
```

---

## GET `/cases`

Query parameters:

```text
status
urgency
case_type_id
assigned_to_me
page
limit
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "case_number": "CASE-2026-000001",
      "title": "Settlement reconciliation mismatch",
      "urgency": "HIGH",
      "status": "CHECKING",
      "case_type": {
        "id": "uuid",
        "code": "SETTLEMENT_EXCEPTION",
        "name": "Settlement Exception"
      },
      "created_at": "2026-10-01T10:00:00Z",
      "updated_at": "2026-10-01T10:05:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

---

## GET `/cases/{case_id}`

Authorization: active case participant or `ADMIN`. ADMIN read access does not grant workflow mutation authority.

Response:

```json
{
  "data": {
    "id": "uuid",
    "case_number": "CASE-2026-000001",
    "title": "Settlement reconciliation mismatch",
    "description": "37 transactions failed reconciliation.",
    "urgency": "HIGH",
    "status": "CHECKING",
    "case_type": {
      "id": "uuid",
      "code": "SETTLEMENT_EXCEPTION",
      "name": "Settlement Exception"
    },
    "maker": {
      "id": "uuid",
      "name": "Operations User"
    },
    "owner": {
      "id": "uuid",
      "name": "Operations User"
    },
    "participants": [
      {
        "id": "uuid",
        "user_id": "uuid",
        "name": "Risk User",
        "role": "CHECKER",
        "required": true,
        "status": "ACTIVE"
      }
    ],
    "current_analysis": {
      "id": "uuid",
      "version": 2,
      "verification_status": "PASS"
    },
    "created_at": "2026-10-01T10:00:00Z",
    "updated_at": "2026-10-01T10:05:00Z"
  }
}
```

---

`current_analysis` may be `null` when no COMPLETED PASS/PASS_WITH_WARNING analysis has ever become reviewable. It is not guaranteed to be the latest analysis attempt.

---

## PATCH `/cases/{case_id}`

Only valid while case is `DRAFT`.

Request:

```json
{
  "case_type_id": "uuid",
  "title": "Updated title",
  "description": "Updated description",
  "urgency": "CRITICAL"
}
```

---

## POST `/cases/{case_id}/submit`

Request:

```json
{}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "AI_ANALYSIS"
  }
}
```

Preconditions:

```text
case.status = DRAFT
authenticated user = Maker

exactly 1 active Maker
at least 1 active required Checker
exactly 1 active Signer
exactly 1 active Executer

segregation of duties valid
```

Behavior:

```text
validate participant cardinality + SoD
→ freeze case core data + participant set
→ DRAFT
→ SUBMITTED
→ AI_ANALYSIS
```

After submission, case core fields and participant assignments are immutable. Backend triggers AI analysis.

---

## POST `/cases/{case_id}/close`

Any active case participant with role `MAKER`, `CHECKER`, `SIGNER`, or `EXECUTER` may close the case.

Allowed source states:

```text
DRAFT
SUBMITTED
AI_ANALYSIS
CHECKING
SIGNING
EXECUTION
ESCALATION_REQUIRED
```

`DONE` and `CLOSED` cannot be closed again.

`reason` is required and must be non-empty.

Request:

```json
{
  "reason": "Duplicate incident"
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "CLOSED",
    "close_reason": "Duplicate incident",
    "closed_at": "2026-10-01T11:00:00Z"
  }
}
```

Behavior:

```text
validate active participant
→ validate non-terminal state
→ lock case
→ ensure no ai_analyses.status = GENERATING
→ ensure no executions.status = IN_PROGRESS
→ persist closed_by / close_reason / closed_at
→ audit CASE_CLOSED with actor role and previous status
→ CLOSED
```

If analysis generation or execution is still active, close returns `409 INVALID_STATE_TRANSITION`; the client must wait for the process to reach a terminal state.

Once the case is `CLOSED`, no later workflow action may transition it out of `CLOSED`.

---

# 9. Case Participants

Participant mutation is valid **only while the case is `DRAFT`**.

Maker is created automatically with the case and cannot be assigned, unassigned, or replaced through the participant API.

Strict SoD: Maker, every Checker, Signer, and Executer are distinct users. A user with an existing ACTIVE participant role cannot be assigned another ACTIVE role on the same case.

Cardinality at submit:

```text
MAKER     exactly 1 active
CHECKER   1..N active, at least 1 required
SIGNER    exactly 1 active
EXECUTER  exactly 1 active
```

## POST `/cases/{case_id}/participants`

Allowed role through this endpoint:

```text
CHECKER
SIGNER
EXECUTER
```

Request:

```json
{
  "user_id": "uuid",
  "role": "CHECKER",
  "required": true
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "case_id": "uuid",
    "user_id": "uuid",
    "role": "CHECKER",
    "required": true,
    "status": "ACTIVE"
  }
}
```

If the case is no longer `DRAFT`:

```text
409 INVALID_STATE_TRANSITION
```

## DELETE `/cases/{case_id}/participants/{participant_id}`

Only valid for `CHECKER`, `SIGNER`, or `EXECUTER` assignments while the case is `DRAFT`.

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "INACTIVE"
  }
}
```

After submission, the participant set is frozen. If a participant must be replaced, close the current case with an explicit reason and create a new case with the new participant set. MVP has no participant replacement or reopen flow for an in-flight/closed case.

---

# 10. Evidence

## GET `/cases/{case_id}/evidences`

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "evidence_type": "COMMENT",
      "source_type": "MAKER",
      "source_user": {
        "id": "uuid",
        "name": "Operations User"
      },
      "title": "Additional context",
      "content": "Upstream batch arrived 20 minutes late.",
      "file_path": null,
      "created_at": "2026-10-01T10:00:00Z"
    }
  ]
}
```

## POST `/cases/{case_id}/evidences`

User-created evidence request:

```json
{
  "evidence_type": "COMMENT",
  "title": "Additional context",
  "content": "Upstream batch arrived 20 minutes late."
}
```

Authorization:

```text
DRAFT               → MAKER only
CHECKING             → any active case participant
SIGNING              → any active case participant
EXECUTION            → any active case participant
ESCALATION_REQUIRED  → any active case participant

SUBMITTED            → forbidden
AI_ANALYSIS           → forbidden
DONE                  → forbidden
CLOSED                → forbidden
```

Because strict SoD allows only one active workflow role per user on a case, client does not send `actor_role` or authoritative `source_type`.

Backend derives:

```text
source_user_id = authenticated user
source_type    = authenticated user's single ACTIVE case role
```

No active assignment → FORBIDDEN.

`SYSTEM` is internal-only and cannot be selected through this endpoint.

Adding evidence does not change case status and does not automatically trigger re-analysis.

---

## POST `/cases/{case_id}/evidences/upload-url`

Request:

```json
{
  "file_name": "settlement-log.pdf",
  "mime_type": "application/pdf"
}
```

Response:

```json
{
  "data": {
    "upload_url": "signed-upload-url",
    "file_key": "cases/uuid/evidence/uuid-settlement-log.pdf"
  }
}
```

---

## POST `/cases/{case_id}/evidences/file`

Request:

```json
{
  "file_key": "cases/uuid/evidence/uuid-settlement-log.pdf",
  "title": "Settlement Log",
  "evidence_type": "DOCUMENT"
}
```

The same evidence state/role authorization applies to both signed-URL issuance and final file-evidence registration. Backend revalidates authorization at registration time because case state may have changed after the upload URL was issued.

The source role is derived server-side from the authenticated user's unique active case assignment; no actor-role field is accepted from the client.

MVP file MIME allowlist:

```text
application/pdf
image/jpeg
image/png
```

Backend also validates that `file_key` belongs to the requested case evidence prefix, the GCS object exists, and the stored object MIME matches the allowed type. Client cannot register an arbitrary object path as case evidence.

Registered file evidence stores `mime_type` and can be passed directly from its GCS URI to Gemini during analysis.

---

# 11. AI Analyses

## GET `/cases/{case_id}/analyses`

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "version": 1,
      "status": "COMPLETED",
      "verification_status": "PASS"
    },
    {
      "id": "uuid",
      "version": 2,
      "status": "COMPLETED",
      "verification_status": "PASS_WITH_WARNING"
    },
    {
      "id": "uuid",
      "version": 3,
      "status": "FAILED",
      "verification_status": "FAIL"
    }
  ]
}
```

---

## GET `/cases/{case_id}/analyses/current`

This endpoint resolves `cases.current_analysis_id`.

Semantics:

```text
current_analysis_id
= latest COMPLETED analysis with PASS / PASS_WITH_WARNING
  that became eligible for human review
```

A newer FAILED analysis does not replace this pointer. Use `GET /cases/{case_id}/analyses` to inspect the latest attempt/version.

If `current_analysis_id` is NULL because no analysis has completed successfully, return:

```text
404 ANALYSIS_NOT_FOUND
```

Client interpretation is state-aware: while the case is `AI_ANALYSIS`, this means no reviewable analysis exists yet and is a normal loading condition, not a missing-case page.

Response:

```json
{
  "data": {
    "id": "uuid",
    "version": 2,
    "status": "COMPLETED",
    "summary": "Settlement exception requires controlled handling.",
    "facts": [
      {
        "statement": "37 transactions are unmatched.",
        "source_type": "CASE",
        "source_ref": null
      }
    ],
    "assumptions": [],
    "unknowns": [
      {
        "item": "Availability of missing approval evidence",
        "impact": "May affect whether settlement can proceed"
      }
    ],
    "policy_status": "POLICY_PARTIAL",
    "risk_analysis": [
      {
        "type": "OPERATIONAL",
        "level": "HIGH",
        "reason": "Settlement cutoff is approaching.",
        "evidence_refs": ["uuid"],
        "policy_refs": ["uuid"]
      }
    ],
    "compliance_analysis": {
      "status": "REQUIRES_REVIEW",
      "reason": "Some approval evidence is missing.",
      "policy_refs": ["uuid"]
    },
    "recommendation": {
      "type": "POLICY_BASED",
      "summary": "Hold affected transactions until approval evidence is verified.",
      "actions": [
        {
          "order": 1,
          "action": "Hold affected transactions.",
          "reason": "Required approval evidence is incomplete.",
          "policy_refs": ["uuid"],
          "evidence_refs": ["uuid"]
        }
      ],
      "potential_benefits": [
        "Prevents unauthorized settlement."
      ],
      "potential_risks": [
        "Settlement may miss operational cutoff."
      ]
    },
    "alternatives": [],
    "missing_information": [],
    "evidence_quality": "HIGH",
    "uncertainty": "MEDIUM",
    "verification": {
      "status": "PASS_WITH_WARNING",
      "issues": []
    },
    "policy_references": [
      {
        "policy_id": "uuid",
        "policy_code": "SOP-OPS-001",
        "policy_title": "Settlement Exception Handling",
        "policy_version_id": "uuid",
        "version": "1.0",
        "section": "4.2",
        "excerpt": "Affected transactions must be held until required approval evidence is complete."
      }
    ],
    "evidence_references": [
      {
        "evidence_id": "uuid",
        "usage_type": "SUPPORTING_FACT"
      }
    ],
    "created_at": "2026-10-01T10:01:00Z"
  }
}
```

---

## GET `/cases/{case_id}/analyses/{analysis_id}`

Response shape sama dengan current analysis.

Historical analysis bersifat read-only.

Analysis result-field semantics:

```text
status = GENERATING
→ structured result fields may be null

status = COMPLETED
→ all required structured result fields are present
→ verification.status = PASS | PASS_WITH_WARNING

status = FAILED before valid analysis output
→ unavailable structured result fields are null
→ verification may be null

status = FAILED after verifier FAIL
→ schema-valid analysis fields may be present
→ verification.status = FAIL
```

`null` means the value was not produced. An empty list/object means a valid output was produced and is intentionally empty.

Clients must not interpret null as `NO_POLICY_FOUND`, empty evidence, PASS/FAIL, or any other business conclusion.

Failure and escalation semantics:

```text
Verifier FAIL
→ analysis.status = FAILED
→ verification.status = FAIL
→ AI_ANALYSIS_FAILED
→ failure_type = VERIFIER_FAIL
→ case.status = ESCALATION_REQUIRED

Technical retry exhausted
→ analysis.status = FAILED
→ AI_ANALYSIS_FAILED
→ failure_type = TECHNICAL_RETRY_EXHAUSTED
→ case.status = ESCALATION_REQUIRED

Re-analysis quota exhausted
→ no new analysis version is created
→ REANALYSIS_LIMIT_REACHED
→ case.status = ESCALATION_REQUIRED
```

For `ESCALATION_REQUIRED`, MVP exposes no resume/retry/reanalyze action. Active participants may add evidence according to the Evidence authorization matrix or close the case.

---

## Manual Re-analysis

MVP does **not** expose:

```http
POST /cases/{case_id}/reanalyze
```

A new analysis version may only be triggered by governed business actions:

```text
CHECKER_REJECTED
SIGNER_REJECTED
EXECUTION_BLOCKED
EXECUTION_FAILED
```

Adding evidence alone does not automatically create a new analysis version.

---

# 12. Checker Decision

## POST `/cases/{case_id}/checker-decisions`

### Approve

```json
{
  "analysis_id": "uuid",
  "decision": "APPROVE",
  "comment": "Reasoning and evidence are acceptable."
}
```

### Reject

```json
{
  "analysis_id": "uuid",
  "decision": "REJECT",
  "reason": "Relevant SOP was not considered.",
  "comment": "Include SOP-RISK-004 before continuing.",
  "evidence_ids": ["uuid"]
}
```

Response:

```json
{
  "data": {
    "decision_id": "uuid",
    "decision": "APPROVE",
    "case_status": "CHECKING"
  }
}
```

Jika semua required Checker sudah approve:

```text
case_status = SIGNING
```

If reject is accepted, the decision is persisted successfully.

Normal result:

```text
case_status = AI_ANALYSIS
→ re-analysis starts after commit
```

If the business re-analysis quota is already exhausted:

```text
CHECKER_REJECTED is still persisted
REANALYSIS_LIMIT_REACHED is audited in the same business transaction
case_status = ESCALATION_REQUIRED
no new analysis version is created
no AI call is triggered
```

This is a successful business action, not an HTTP conflict.

---

# 13. Checker Status

## GET `/cases/{case_id}/checker-status`

Response:

```json
{
  "data": {
    "analysis_id": "uuid",
    "required": 2,
    "approved": 1,
    "rejected": 0,
    "pending": 1,
    "checkers": [
      {
        "user_id": "uuid",
        "name": "Risk User",
        "status": "APPROVED"
      },
      {
        "user_id": "uuid",
        "name": "Development User",
        "status": "PENDING"
      }
    ]
  }
}
```

---

# 14. Signer Decision

## POST `/cases/{case_id}/signer-decision`

### Approve

```json
{
  "analysis_id": "uuid",
  "decision": "APPROVE",
  "comment": "Authorized for execution."
}
```

### Reject

```json
{
  "analysis_id": "uuid",
  "decision": "REJECT",
  "reason": "Operational impact is too high.",
  "comment": "Provide a lower-impact alternative."
}
```

Response:

```json
{
  "data": {
    "decision_id": "uuid",
    "decision": "APPROVE",
    "case_status": "EXECUTION"
  }
}
```

Reject is persisted successfully.

Normal result:

```text
case_status = AI_ANALYSIS
```

If re-analysis quota is exhausted:

```text
SIGNER_REJECTED remains persisted
case_status = ESCALATION_REQUIRED
no new analysis version is created
```

---

# 15. Execution

## POST `/cases/{case_id}/executions`

Request:

```json
{
  "analysis_id": "uuid"
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "IN_PROGRESS",
    "started_at": "2026-10-01T10:20:00Z"
  }
}
```

---

## POST `/cases/{case_id}/executions/{execution_id}/result`

### Success

```json
{
  "status": "SUCCESS",
  "action_taken": "Affected transactions were isolated.",
  "result": "Reconciliation completed successfully."
}
```

### Blocked

```json
{
  "status": "BLOCKED",
  "action_taken": "Attempted reconciliation retry.",
  "blocker": "Upstream settlement file is unavailable.",
  "evidence_ids": ["uuid"]
}
```

### Failed

```json
{
  "status": "FAILED",
  "action_taken": "Triggered reconciliation retry.",
  "result": "Retry failed.",
  "blocker": "Dependency service unavailable.",
  "evidence_ids": ["uuid"]
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "SUCCESS",
    "case_status": "DONE"
  }
}
```

For `BLOCKED` / `FAILED`, the execution result is always persisted if the request is otherwise valid.

Normal result:

```text
case_status = AI_ANALYSIS
```

If re-analysis quota is exhausted:

```text
execution BLOCKED/FAILED result + evidence remain persisted
case_status = ESCALATION_REQUIRED
no new analysis version is created
no AI call is triggered
```

---

# 16. Policies

## GET `/policies`

Query parameters:

```text
status
case_type_id
domain
page
limit
```

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "code": "SOP-OPS-001",
      "title": "Settlement Exception Handling",
      "domain": "SETTLEMENT",
      "case_type": {
        "id": "uuid",
        "code": "SETTLEMENT_EXCEPTION",
        "name": "Settlement Exception"
      },
      "active_version": {
        "id": "uuid",
        "version": "1.0",
        "status": "ACTIVE",
        "index_status": "READY"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

---

## POST `/policies`

Authorization: `ADMIN` only.

Request:

```json
{
  "code": "SOP-OPS-001",
  "title": "Settlement Exception Handling",
  "domain": "SETTLEMENT",
  "case_type_id": "uuid",
  "description": "Operational procedure for settlement exceptions."
}
```

---

# 17. Policy Versions

## POST `/policies/{policy_id}/versions`

Authorization: `ADMIN` only.

Request:

```json
{
  "version": "1.0",
  "content": "Policy content...",
  "effective_from": "2026-10-01T00:00:00Z",
  "effective_until": null
}
```

Validation:

```text
if effective_from and effective_until are both set:
effective_until > effective_from
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "version": "1.0",
    "status": "DRAFT",
    "index_status": "NOT_STARTED"
  }
}
```

---

## GET `/policies/{policy_id}/versions/{version_id}`

Response:

```json
{
  "data": {
    "id": "uuid",
    "policy_id": "uuid",
    "version": "1.0",
    "status": "ACTIVE",
    "index_status": "READY",
    "index_error": null,
    "index_attempt_id": "uuid",
    "index_started_at": "2026-10-01T09:13:00Z",
    "index_recoverable": false,
    "indexed_at": "2026-10-01T09:14:00Z",
    "content": "Policy content...",
    "effective_from": "2026-10-01T00:00:00Z",
    "effective_until": null,
    "created_at": "2026-10-01T09:00:00Z",
    "approved_at": "2026-10-01T09:15:00Z"
  }
}
```

---

`index_recoverable` is a derived response field, not a database column:

```text
index_status = PROCESSING
AND now() - index_started_at > POLICY_INDEX_LEASE_SECONDS
```

It allows ADMIN UI to offer a safe recovery action without duplicating backend lease logic.

---

## POST `/policies/{policy_id}/versions/{version_id}/activate`

Authorization: `ADMIN` only.

Request:

```json
{}
```

Successful response:

```json
{
  "data": {
    "id": "uuid",
    "version": "1.0",
    "status": "ACTIVE",
    "index_status": "READY",
    "indexed_at": "2026-10-01T09:14:00Z"
  }
}
```

Preconditions:

```text
target.status = DRAFT
effective_from IS NULL OR effective_from <= now()
effective_until IS NULL OR effective_until > now()
```

Future-effective and expired versions cannot be activated in MVP. There is no scheduled activation.

Behavior when indexing is required:

```text
claim index_attempt_id
set index_started_at = now()
target.index_status = PROCESSING
↓ COMMIT

chunk target content
↓
generate all embeddings
↓
finalize only if index_attempt_id still matches
↓
target.index_status = READY

↓
atomic activation transaction
↓
revalidate target DRAFT + READY + effective NOW
↓
current ACTIVE → SUPERSEDED
target DRAFT → ACTIVE
↓
audit policy lifecycle
```

PROCESSING recovery:

```text
lease age <= POLICY_INDEX_LEASE_SECONDS
→ 409 INVALID_STATE_TRANSITION (indexing still in progress)

lease age > POLICY_INDEX_LEASE_SECONDS
→ claim a new index_attempt_id
→ retry indexing
→ old attempt cannot finalize because its token no longer matches
```

If the current attempt fails:

```text
target.status       = DRAFT
target.index_status = FAILED
target.index_error  = safe diagnostic summary

current ACTIVE version remains unchanged
```

Indexing/Vertex calls do not run inside an open database transaction.

Only `ACTIVE + READY + effective` policy versions are eligible for retrieval.

MVP does not expose an endpoint to mutate DRAFT policy content after creation. A changed policy body is created as a new policy version.

---

# 18. Case History

## GET `/cases/{case_id}/history`

Response:

```json
{
  "data": [
    {
      "id": "uuid",
      "event_type": "CASE_CREATED",
      "actor": {
        "id": "uuid",
        "name": "Operations User"
      },
      "actor_role": "MAKER",
      "analysis_id": null,
      "analysis_version": null,
      "metadata": {},
      "created_at": "2026-10-01T10:00:00Z"
    },
    {
      "id": "uuid",
      "event_type": "CHECKER_REJECTED",
      "actor": {
        "id": "uuid",
        "name": "Risk User"
      },
      "actor_role": "CHECKER",
      "analysis_id": "uuid",
      "analysis_version": 1,
      "metadata": {
        "reason": "Relevant SOP was not considered."
      },
      "created_at": "2026-10-01T10:10:00Z"
    }
  ]
}
```

Escalation-related safe metadata may include:

```json
{
  "event_type": "AI_ANALYSIS_FAILED",
  "metadata": {
    "failure_type": "VERIFIER_FAIL"
  }
}
```

or:

```json
{
  "event_type": "REANALYSIS_LIMIT_REACHED",
  "metadata": {
    "latest_analysis_version": 4,
    "max_reanalysis": 3
  }
}
```

Raw provider payload, prompt/evidence content, stack trace, and credentials are never exposed through history metadata.

---

# 19. Dashboard

## GET `/dashboard/summary`

Response:

```json
{
  "data": {
    "my_cases": 5,
    "need_my_review": 2,
    "need_my_signature": 1,
    "need_my_execution": 1,
    "status_counts": {
      "DRAFT": 1,
      "AI_ANALYSIS": 1,
      "CHECKING": 3,
      "SIGNING": 1,
      "EXECUTION": 1,
      "DONE": 8,
      "CLOSED": 2,
      "ESCALATION_REQUIRED": 0
    }
  }
}
```

---

# 20. Error Mapping

| Error Code | HTTP Status |
|---|---:|
| INVALID_REQUEST | 400 |
| UNAUTHORIZED | 401 |
| FORBIDDEN | 403 |
| SEGREGATION_OF_DUTIES_VIOLATION | 403 |
| USER_NOT_FOUND | 404 |
| CASE_NOT_FOUND | 404 |
| POLICY_NOT_FOUND | 404 |
| ANALYSIS_NOT_FOUND | 404 |
| INVALID_STATE_TRANSITION | 409 |
| STALE_ANALYSIS | 409 |
| POLICY_INDEXING_FAILED | 502 |
| INTERNAL_ERROR | 500 |

---

# 21. Concurrency Contract

Decision request selalu membawa:

```text
analysis_id
```

Backend memvalidasi:

```text
request.analysis_id == case.current_analysis_id
```

Jika berbeda:

```http
409 Conflict
```

Response:

```json
{
  "error": {
    "code": "STALE_ANALYSIS",
    "message": "Analysis version is no longer current.",
    "details": {}
  }
}
```

Frontend harus melakukan refetch dan meminta user mereview analysis terbaru.

Approval tidak boleh di-retry otomatis.

---

# 22. State Transition Contract

Frontend tidak dapat meminta arbitrary status.

Endpoint berikut tidak tersedia:

```http
POST /cases/{case_id}/change-status
```

Status berubah hanya melalui business action:

```text
submit
checker approve
checker reject
signer approve
signer reject
execution success
execution blocked
execution failed
close
```

---

# 23. Workflow Contract

## Submit

```text
DRAFT
→ AI_ANALYSIS
+ analysis GENERATING
+ transactional outbox AI_ANALYSIS_REQUESTED
```

The API returns after the database transaction commits; Gemini runs asynchronously through RabbitMQ.

## Analysis Success

```text
AI_ANALYSIS
→ CHECKING
```

## Analysis Terminal Failure

```text
AI_ANALYSIS
→ ESCALATION_REQUIRED
```

Cause:
```text
VERIFIER_FAIL
TECHNICAL_RETRY_EXHAUSTED
```

## Re-analysis Limit Reached

```text
governed reject/block/fail action succeeds
→ no next analysis version
→ ESCALATION_REQUIRED
```

## All Required Checker Approve

```text
CHECKING
→ SIGNING
```

## Checker Reject

```text
CHECKING
→ AI_ANALYSIS
```

## Signer Approve

```text
SIGNING
→ EXECUTION
```

## Signer Reject

```text
SIGNING
→ AI_ANALYSIS
```

## Execution Success

```text
EXECUTION
→ DONE
```

## Execution Blocked / Failed

```text
EXECUTION
→ AI_ANALYSIS
```

---

# 24. Contract Ownership

Dokumen ini berada pada:

```text
jawir-sentinel-docs/api/api-contract.md
```

Perubahan request, response, enum, endpoint, error code, atau behavior harus diperbarui pada dokumen ini terlebih dahulu.

Implementasi kemudian disesuaikan pada:

```text
jawir-sentinel-be
jawir-sentinel-fe
```
