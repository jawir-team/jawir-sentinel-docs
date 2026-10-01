# Kontrak API JAWIR Sentinel

**Versi Spesifikasi:** 1.0  
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

## Model Authorization

System role:

```text
USER
ADMIN
```

System role disimpan pada internal user dan terpisah dari case workflow role.

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

# 2. Response Standar

## Berhasil

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

# 3. Enum Inti

## Status Case

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

Satu active user hanya boleh memiliki satu participant role pada sebuah case.

## Decision

```text
APPROVE
REJECT
```

## Status Execution

```text
IN_PROGRESS
SUCCESS
BLOCKED
FAILED
```

## Status Policy

```text
DRAFT
ACTIVE
SUPERSEDED
```

## Status Index Policy

```text
NOT_STARTED
PROCESSING
READY
FAILED
```

## Status Analysis

```text
GENERATING
COMPLETED
FAILED
```

## Status Policy AI

```text
POLICY_FOUND
POLICY_PARTIAL
NO_POLICY_FOUND
INSUFFICIENT_EVIDENCE
POLICY_CONFLICT
```

## Status Verifikasi

```text
PASS
PASS_WITH_WARNING
FAIL
```

## Kualitas Evidence

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

# 4. User Saat Ini

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

Otorisasi: hanya `ADMIN`.

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

Tersedia untuk semua authenticated ACTIVE user karena Maker membutuhkan participant directory.

List response sengaja **tidak** mengekspos `firebase_uid`.

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

Otorisasi: hanya `ADMIN`.

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

Otorisasi: hanya `ADMIN`.

Safety invariant:

```text
The system must always retain at least one ACTIVE ADMIN.
```

PATCH yang akan menurunkan system role atau menonaktifkan **ACTIVE ADMIN terakhir** ditolak dengan `409 INVALID_STATE_TRANSITION`. Ini termasuk self-demotion/self-deactivation ketika caller adalah ACTIVE ADMIN terakhir.

User deactivation guard:

```text
status ACTIVE → INACTIVE
is forbidden while the target user has any ACTIVE case_participant
on a non-terminal case.
```

Non-terminal berarti seluruh case state selain `DONE` dan `CLOSED`.

Guard ini mencegah frozen workflow assignment menjadi tidak dapat dijalankan. Case harus selesai atau di-CLOSE dengan aman terlebih dahulu.

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

Otorisasi: hanya `ADMIN`.

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

Authenticated creator otomatis menjadi satu-satunya active `MAKER` dan immutable `owner` pada case.

```text
created_by = authenticated user
owner_id   = authenticated user
MAKER      = authenticated user
```

Maker/owner tidak dapat diganti pada MVP.

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

Otorisasi: active case participant atau `ADMIN`. Read access ADMIN tidak memberi workflow mutation authority.

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

`current_analysis` dapat bernilai `null` jika belum pernah ada COMPLETED PASS/PASS_WITH_WARNING analysis yang reviewable. Nilai ini tidak selalu menunjuk analysis attempt terbaru.

---

## PATCH `/cases/{case_id}`

Hanya valid ketika case berstatus `DRAFT`.

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

Prasyarat:

```text
case.status = DRAFT
authenticated user = Maker

exactly 1 active Maker
at least 1 active required Checker
exactly 1 active Signer
exactly 1 active Executer

segregation of duties valid
```

Behaviatau:

```text
validate participant cardinality + SoD
→ freeze case core data + participant set
→ DRAFT
→ SUBMITTED
→ AI_ANALYSIS
```

Setelah submission, core field case dan participant assignment menjadi immutable. Backend menyiapkan AI analysis secara asynchronous.

---

## POST `/cases/{case_id}/close`

Active case participant dengan role `MAKER`, `CHECKER`, `SIGNER`, atau `EXECUTER` dapat menutup case.

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

Behaviatau:

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

Jika analysis generation atau execution masih aktif, close mengembalikan `409 INVALID_STATE_TRANSITION`; client harus menunggu proses mencapai terminal state.

Setelah case menjadi `CLOSED`, workflow action berikutnya tidak boleh memindahkan case keluar dari `CLOSED`.

---

# 9. Case Participants

Participant mutation valid **hanya ketika case berstatus `DRAFT`**.

Maker dibuat otomatis bersama case dan tidak dapat di-assign, di-unassign, atau diganti melalui participant API.

Strict SoD: Maker, setiap Checker, Signer, dan Executer harus menggunakan user berbeda. User yang sudah memiliki ACTIVE participant role tidak dapat diberi ACTIVE role lain pada case yang sama.

Cardinality at submit:

```text
MAKER     exactly 1 active
CHECKER   1..N active, at least 1 required
SIGNER    exactly 1 active
EXECUTER  exactly 1 active
```

## POST `/cases/{case_id}/participants`

Role yang diizinkan melalui endpoint ini:

```text
CHECKER
SIGNER
EXECUTER
```

Prasyarat:

```text
case.status = DRAFT
target user.status = ACTIVE
target user has no other ACTIVE role on this case
strict SoD remains valid
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

Jika case sudah bukan `DRAFT`:

```text
409 INVALID_STATE_TRANSITION
```

## DELETE `/cases/{case_id}/participants/{participant_id}`

Hanya valid untuk assignment `CHECKER`, `SIGNER`, atau `EXECUTER` selama case berstatus `DRAFT`.

Response:

```json
{
  "data": {
    "id": "uuid",
    "status": "INACTIVE"
  }
}
```

Setelah submission, participant set dibekukan. Jika participant harus diganti, tutup case saat ini dengan reason yang jelas lalu buat case baru dengan participant set baru. MVP tidak memiliki participant replacement atau reopen flow untuk case yang sedang berjalan maupun sudah CLOSED.

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
      "mime_type": null,
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

Otorisasi:

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

Karena strict SoD hanya mengizinkan satu active workflow role per user pada satu case, client tidak mengirim `actor_role` atau authoritative `source_type`.

Backend melakukan derivasi:

```text
source_user_id = authenticated user
source_type    = authenticated user's single ACTIVE case role
```

No active assignment → FORBIDDEN.

`SYSTEM` is internal-only and cannot be selected through this endpoint.

Menambahkan evidence tidak mengubah case status dan tidak otomatis memicu re-analysis.

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

Authorization state/role evidence yang sama berlaku saat signed-URL issuance maupun final file-evidence registration. Backend memvalidasi ulang authorization saat registration karena case state dapat berubah setelah upload URL diterbitkan.

Source role di-derive server-side dari unique active case assignment authenticated user; client tidak mengirim actor-role field.

MVP file MIME allowlist:

```text
application/pdf
image/jpeg
image/png
```

Backend juga memvalidasi bahwa `file_key` berada pada evidence prefix milik case, GCS object benar-benar ada, dan MIME object sesuai allowlist. Client tidak dapat mendaftarkan arbitrary object path sebagai case evidence.

File evidence yang terdaftar menyimpan `mime_type` dan dapat dikirim langsung dari GCS URI ke Gemini saat analysis.

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

Endpoint ini me-resolve `cases.current_analysis_id`.

Semantik:

```text
current_analysis_id
= latest COMPLETED analysis with PASS / PASS_WITH_WARNING
  that became eligible for human review
```

Analysis FAILED yang lebih baru tidak mengganti pointer ini. Gunakan `GET /cases/{case_id}/analyses` untuk melihat attempt/version terbaru.

Jika `current_analysis_id` NULL karena belum ada analysis yang selesai dengan sukses, kembalikan:

```text
404 ANALYSIS_NOT_FOUND
```

Interpretasi client bergantung pada state: ketika case berstatus `AI_ANALYSIS`, kondisi ini berarti belum ada reviewable analysis dan harus dianggap sebagai loading normal, bukan halaman case yang hilang.

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

Bentuk response sama dengan analysis saat ini.

Analysis historis bersifat read-only.

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

Client tidak boleh menafsirkan null sebagai `NO_POLICY_FOUND`, evidence kosong, PASS/FAIL, atau business conclusion lain.

Semantik failure dan escalation:

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

Untuk `ESCALATION_REQUIRED`, MVP tidak menyediakan action resume/retry/reanalyze. Active participant dapat menambah evidence sesuai authorization matrix Evidence atau menutup case.

---

## Manual Re-analysis

MVP **tidak** menyediakan:

```http
POST /cases/{case_id}/reanalyze
```

Analysis version baru hanya dapat dipicu oleh governed business action:

```text
CHECKER_REJECTED
SIGNER_REJECTED
EXECUTION_BLOCKED
EXECUTION_FAILED
```

Menambahkan evidence saja tidak otomatis membuat analysis version baru.

---

# 12. Decision Checker

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

Jika reject diterima, decision dipersist dengan sukses.

Hasil normal:

```text
case_status = AI_ANALYSIS
→ re-analysis starts after commit
```

Jika business re-analysis quota sudah habis:

```text
CHECKER_REJECTED is still persisted
REANALYSIS_LIMIT_REACHED is audited in the same business transaction
case_status = ESCALATION_REQUIRED
no new analysis version is created
no AI call is triggered
```

Ini adalah business action yang berhasil, bukan HTTP conflict.

---

# 13. Status Checker

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

# 14. Decision Signer

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

Reject dipersist dengan sukses.

Hasil normal:

```text
case_status = AI_ANALYSIS
```

Jika re-analysis quota habis:

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

### Berhasil

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

Untuk `BLOCKED` / `FAILED`, execution result selalu dipersist selama request valid.

Hasil normal:

```text
case_status = AI_ANALYSIS
```

Jika re-analysis quota habis:

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

Otorisasi: hanya `ADMIN`.

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

Otorisasi: hanya `ADMIN`.

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

Field ini memungkinkan UI ADMIN menawarkan recovery action yang aman tanpa menduplikasi backend lease logic.

---

## POST `/policies/{policy_id}/versions/{version_id}/activate`

Otorisasi: hanya `ADMIN`.

Request:

```json
{}
```

Response berhasil:

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

Prasyarat:

```text
target.status = DRAFT
effective_from IS NULL OR effective_from <= now()
effective_until IS NULL OR effective_until > now()
```

Version future-effective dan expired tidak dapat diaktifkan pada MVP. Tidak ada scheduled activation.

Perilaku saat indexing diperlukan:

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

Jika attempt saat ini gagal:

```text
target.status       = DRAFT
target.index_status = FAILED
target.index_error  = safe diagnostic summary

current ACTIVE version remains unchanged
```

Indexing/Vertex call tidak dijalankan di dalam open database transaction.

Hanya policy version `ACTIVE + READY + effective` yang eligible untuk retrieval.

MVP tidak menyediakan endpoint untuk mengubah content DRAFT policy setelah dibuat. Perubahan policy body dibuat sebagai policy version baru.

---

# 18. History Case

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

Safe metadata terkait escalation dapat mencakup:

```json
{
  "event_type": "AI_ANALYSIS_FAILED",
  "metadata": {
    "failure_type": "VERIFIER_FAIL"
  }
}
```

atau:

```json
{
  "event_type": "REANALYSIS_LIMIT_REACHED",
  "metadata": {
    "latest_analysis_version": 4,
    "max_reanalysis": 3
  }
}
```

Raw provider payload, prompt/evidence content, stack trace, dan credential tidak pernah diekspos melalui history metadata.

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

# 20. Mapping Error

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

# 21. Kontrak Concurrency

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

# 22. Kontrak Transisi State

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

# 23. Kontrak Workflow

## Submit

```text
DRAFT
→ AI_ANALYSIS
+ analysis GENERATING
+ transactional outbox AI_ANALYSIS_REQUESTED
```

API mengembalikan response setelah database transaction commit; Gemini berjalan asynchronous melalui RabbitMQ.

## Analysis Berhasil

```text
AI_ANALYSIS
→ CHECKING
```

## Kegagalan Terminal Analysis

```text
AI_ANALYSIS
→ ESCALATION_REQUIRED
```

Cause:
```text
VERIFIER_FAIL
TECHNICAL_RETRY_EXHAUSTED
```

## Batas Re-analysis Tercapai

```text
governed reject/block/fail action succeeds
→ no next analysis version
→ ESCALATION_REQUIRED
```

## Semua Required Checker Approve

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

## Execution Berhasil

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

# 24. Kepemilikan Kontrak

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
