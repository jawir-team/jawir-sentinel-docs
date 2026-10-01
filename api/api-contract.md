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

## Participant Role

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

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
    "is_admin": false,
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
      "firebase_uid": "firebase-uid",
      "name": "Risk User",
      "email": "risk.user@example.com",
      "status": "ACTIVE",
      "is_admin": false,
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

Request:

```json
{
  "firebase_uid": "firebase-uid",
  "name": "Risk User",
  "email": "risk.user@example.com",
  "unit_id": "uuid",
  "status": "ACTIVE",
  "is_admin": false
}
```

## PATCH `/users/{user_id}`

Request:

```json
{
  "name": "Risk Reviewer",
  "unit_id": "uuid",
  "status": "ACTIVE",
  "is_admin": false
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

The authenticated creator is automatically assigned as the case's single active `MAKER`. Maker assignment cannot be replaced or unassigned.

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
    "status": "DRAFT",
    "index_status": "NOT_STARTED"
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
→ persist closed_by / close_reason / closed_at
→ audit CASE_CLOSED with actor role and previous status
→ CLOSED
```

Once the case is `CLOSED`, asynchronous AI completion or later workflow actions must not transition it out of `CLOSED`.

---

# 9. Case Participants

Participant mutation is valid **only while the case is `DRAFT`**.

Maker is created automatically with the case and cannot be assigned, unassigned, or replaced through the participant API.

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
  "actor_role": "CHECKER",
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

Backend validates `actor_role` against the authenticated user's active case assignment.

Client does not set authoritative `source_type`. Backend derives:

```text
source_user_id = authenticated user
source_type    = validated actor_role
```

`SYSTEM` cannot be selected through this endpoint.

Adding evidence does not change case status and does not automatically trigger re-analysis.

---

## POST `/cases/{case_id}/evidences/upload-url`

Request:

```json
{
  "actor_role": "CHECKER",
  "file_name": "settlement-log.pdf",
  "content_type": "application/pdf"
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
  "actor_role": "CHECKER",
  "file_key": "cases/uuid/evidence/uuid-settlement-log.pdf",
  "title": "Settlement Log",
  "evidence_type": "DOCUMENT"
}
```

The same evidence state/role authorization applies to both signed-URL issuance and final file-evidence registration. Backend revalidates authorization at registration time because case state may have changed after the upload URL was issued.

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
    }
  ]
}
```

---

## GET `/cases/{case_id}/analyses/current`

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

Jika reject:

```text
case_status = AI_ANALYSIS
```

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

Reject response:

```text
case_status = AI_ANALYSIS
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

For `BLOCKED` / `FAILED`:

```text
case_status = AI_ANALYSIS
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
        "status": "ACTIVE"
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

Request:

```json
{
  "version": "1.0",
  "content": "Policy content...",
  "effective_from": "2026-10-01T00:00:00Z",
  "effective_until": null
}
```

Response:

```json
{
  "data": {
    "id": "uuid",
    "version": "1.0",
    "status": "DRAFT"
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

## POST `/policies/{policy_id}/versions/{version_id}/activate`

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

Behavior:

```text
validate target DRAFT
↓
target.index_status = PROCESSING
↓ COMMIT

chunk target content
↓
generate all embeddings
↓
persist complete embedded chunks
↓
target.index_status = READY

↓
atomic activation transaction
↓
current ACTIVE → SUPERSEDED
target DRAFT → ACTIVE
↓
audit policy lifecycle
```

If indexing fails:

```text
target.status       = DRAFT
target.index_status = FAILED
target.index_error  = safe diagnostic summary

current ACTIVE version remains unchanged
```

Indexing/Vertex calls do not run inside an open activation transaction.

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
| POLICY_CONFLICT | 409 |
| REANALYSIS_LIMIT_REACHED | 409 |
| POLICY_INDEXING_FAILED | 502 |
| AI_OUTPUT_INVALID | 502 |
| AI_ANALYSIS_FAILED | 502 |
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
```

## Analysis Success

```text
AI_ANALYSIS
→ CHECKING
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
