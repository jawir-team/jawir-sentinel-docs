# JAWIR Sentinel Workflow

**Specification Version:** 1.0  
**Status:** MVP Baseline  
**Repository Path:** `jawir-sentinel-docs/workflow/workflow.md`

Dokumen ini mendefinisikan workflow resmi JAWIR Sentinel untuk MVP.

Workflow ini menjadi contract bersama antara:

```text
jawir-sentinel-fe
jawir-sentinel-be
jawir-sentinel-docs
```

Backend adalah source of truth untuk state transition dan workflow validation.

Frontend hanya mempresentasikan state dan mengirim business action.

AI tidak memiliki authority untuk mengubah workflow state.

---

# 1. Workflow Objective

JAWIR Sentinel menggunakan governed decision workflow untuk memastikan operational case:

- dianalisis dengan context dan evidence yang relevan;
- diperiksa terhadap active SOP/policy;
- direview oleh required Checker;
- diotorisasi oleh Signer;
- dieksekusi oleh Executer;
- dapat dianalisis ulang ketika terdapat reject, blocker, failure, atau evidence baru;
- memiliki audit trail lengkap.

Prinsip utama:

> **AI assists the decision. Humans remain accountable for the decision.**

---

# 2. Workflow Roles

Role ditentukan **per case**.

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

## 2.1 Maker

Maker bertanggung jawab untuk:

- membuat case;
- mengisi case detail;
- memberikan initial context;
- menambahkan evidence;
- menentukan urgency;
- menentukan participant;
- submit case.

Maker tidak memberikan review atau authorization terhadap case miliknya.

---

## 2.2 Checker

Checker bertanggung jawab untuk:

- memeriksa AI analysis;
- memeriksa facts, assumptions, dan unknowns;
- memeriksa policy grounding;
- memeriksa risk analysis;
- memeriksa compliance analysis;
- memeriksa recommendation;
- memeriksa evidence;
- memberikan `APPROVE` atau `REJECT`.

Satu case dapat memiliki lebih dari satu Checker.

Semua required Checker harus `APPROVE` sebelum case dapat masuk ke `SIGNING`.

---

## 2.3 Signer

Signer bertanggung jawab untuk:

- memeriksa current analysis;
- memeriksa Checker decisions;
- memeriksa recommendation;
- memeriksa policy reference;
- memeriksa evidence;
- memberikan authorization.

Signer dapat:

```text
APPROVE
REJECT
```

Signer approval memberikan authorization untuk execution.

---

## 2.4 Executer

Executer bertanggung jawab untuk menjalankan approved action.

Execution result:

```text
SUCCESS
BLOCKED
FAILED
```

Jika execution berhasil:

```text
case → DONE
```

Jika execution blocked atau failed:

```text
case → AI_ANALYSIS
```

---

# 3. Segregation of Duties

MVP rules:

```text
Maker ≠ Checker
Maker ≠ Signer

Checker ≠ Signer
Checker ≠ Executer

Signer ≠ Executer

Maker = Executer allowed
```

Rules divalidasi oleh backend.

Frontend dapat membantu mencegah invalid assignment pada UI, tetapi backend tetap menjadi authority.

---

# 4. Main Workflow

```text
┌──────────────┐
│    MAKER     │
└──────┬───────┘
       │
       │ Create + Submit
       ▼
┌──────────────┐
│ AI ANALYSIS  │
└──────┬───────┘
       │
       │ Analysis Success
       ▼
┌──────────────┐
│ CHECKER(S)   │
└──────┬───────┘
       │
       │ All Required Approve
       ▼
┌──────────────┐
│    SIGNER    │
└──────┬───────┘
       │
       │ Approve
       ▼
┌──────────────┐
│   EXECUTER   │
└──────┬───────┘
       │
       │ Success
       ▼
┌──────────────┐
│     DONE     │
└──────────────┘
```

---

# 5. Case States

Case memiliki state berikut:

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

---

# 6. State Definition

## 6.1 DRAFT

Case masih dapat diedit oleh Maker.

Allowed activity:

```text
Edit case
Add evidence
Assign participant
Remove participant
Submit
Close
```

Case belum dianalisis AI.

---

## 6.2 SUBMITTED

Case telah berhasil disubmit.

State ini bersifat transisi internal sebelum AI analysis dimulai.

Normal flow:

```text
SUBMITTED
→ AI_ANALYSIS
```

Frontend tidak perlu memberikan business action khusus pada state ini.

---

## 6.3 AI_ANALYSIS

AI sedang:

- membangun context;
- mengambil relevant active policy;
- menganalisis case;
- melakukan risk/compliance assessment;
- menghasilkan recommendation;
- menjalankan verifier.

User tidak dapat memberikan Checker/Signer decision pada state ini.

---

## 6.4 CHECKING

Current AI analysis siap direview.

Required Checker dapat:

```text
APPROVE
REJECT
```

Jika semua required Checker approve:

```text
CHECKING → SIGNING
```

Jika satu Checker reject:

```text
CHECKING → AI_ANALYSIS
```

---

## 6.5 SIGNING

Seluruh required Checker sudah approve current analysis.

Signer dapat:

```text
APPROVE
REJECT
```

Approve:

```text
SIGNING → EXECUTION
```

Reject:

```text
SIGNING → AI_ANALYSIS
```

---

## 6.6 EXECUTION

Current analysis telah mendapat authorization dari Signer.

Assigned Executer melakukan action.

Result:

```text
SUCCESS
BLOCKED
FAILED
```

Success:

```text
EXECUTION → DONE
```

Blocked/Failed:

```text
EXECUTION → AI_ANALYSIS
```

---

## 6.7 DONE

Approved action telah berhasil dieksekusi.

`DONE` adalah terminal state untuk MVP.

Case menjadi read-only.

---

## 6.8 CLOSED

Case dihentikan tanpa menyelesaikan execution flow.

Contoh reason:

```text
Duplicate case
False alarm
Issue resolved externally
No action required
```

Close harus menyimpan:

```text
closed_by
close_reason
closed_at
```

`CLOSED` berbeda dari `DONE`.

---

## 6.9 ESCALATION_REQUIRED

Case membutuhkan manual intervention karena automated governed loop tidak dapat dilanjutkan.

Trigger:

```text
Re-analysis limit reached
Repeated AI analysis failure
Unresolved policy conflict
Workflow deadlock
```

Case tidak otomatis melanjutkan workflow sampai human intervention dilakukan.

---

# 7. State Transition Table

| Current State | Event | Next State |
|---|---|---|
| DRAFT | SUBMIT | SUBMITTED |
| SUBMITTED | START_ANALYSIS | AI_ANALYSIS |
| AI_ANALYSIS | ANALYSIS_SUCCESS | CHECKING |
| AI_ANALYSIS | ANALYSIS_FAILED_LIMIT | ESCALATION_REQUIRED |
| CHECKING | ALL_CHECKERS_APPROVED | SIGNING |
| CHECKING | CHECKER_REJECTED | AI_ANALYSIS |
| SIGNING | SIGNER_APPROVED | EXECUTION |
| SIGNING | SIGNER_REJECTED | AI_ANALYSIS |
| EXECUTION | EXECUTION_SUCCESS | DONE |
| EXECUTION | EXECUTION_BLOCKED | AI_ANALYSIS |
| EXECUTION | EXECUTION_FAILED | AI_ANALYSIS |

`CLOSE` mengikuti authorization dan state rule yang divalidasi backend.

---

# 8. Workflow Event Catalog

```text
SUBMIT
START_ANALYSIS
ANALYSIS_SUCCESS
ANALYSIS_FAILED_LIMIT

CHECKER_APPROVED
CHECKER_REJECTED
ALL_CHECKERS_APPROVED

SIGNER_APPROVED
SIGNER_REJECTED

EXECUTION_STARTED
EXECUTION_SUCCESS
EXECUTION_BLOCKED
EXECUTION_FAILED

CLOSE
```

Backend memetakan business action ke workflow event.

---

# 9. Submit Flow

Initial state:

```text
DRAFT
```

Preconditions:

```text
authenticated user = Maker
case status = DRAFT
case detail valid
required participant available
segregation of duties valid
```

Flow:

```text
Validate Case
↓
Validate Participants
↓
Validate Segregation of Duties
↓
Persist Submission
↓
Audit CASE_SUBMITTED
↓
DRAFT → SUBMITTED
↓
START_ANALYSIS
↓
SUBMITTED → AI_ANALYSIS
↓
Trigger AI Analysis
```

Result:

```text
case.status = AI_ANALYSIS
```

---

# 10. AI Analysis Flow

Input:

```text
Current Case
Current Evidence
Current ACTIVE Policies
Reviewer Feedback
Execution Feedback
Current Workflow State
```

Pipeline:

```text
Context Builder
↓
Policy Retrieval
↓
Case Analysis
↓
Risk & Compliance Analysis
↓
Recommendation
↓
Verification
↓
Persist Analysis Version
```

If verifier:

```text
PASS
PASS_WITH_WARNING
```

then:

```text
AI_ANALYSIS → CHECKING
```

If repeated analysis failure reaches configured limit:

```text
AI_ANALYSIS → ESCALATION_REQUIRED
```

---

# 11. Checker Flow

Initial state:

```text
CHECKING
```

A Checker decision is always tied to:

```text
analysis_id
```

## 11.1 Checker Approve

Preconditions:

```text
case.status = CHECKING
actor assigned as active CHECKER
analysis_id = case.current_analysis_id
actor has not decided on current analysis
```

Flow:

```text
Insert APPROVE Decision
↓
Audit CHECKER_APPROVED
↓
Check Required Checker Completion
```

If not all required Checker approved:

```text
Remain CHECKING
```

If all required Checker approved:

```text
Audit / Apply ALL_CHECKERS_APPROVED
↓
CHECKING → SIGNING
```

---

## 11.2 Checker Reject

Preconditions:

```text
case.status = CHECKING
actor assigned as active CHECKER
analysis_id = case.current_analysis_id
reason provided
```

Flow:

```text
Insert REJECT Decision
↓
Store Reviewer Feedback
↓
Store Additional Evidence
↓
Audit CHECKER_REJECTED
↓
CHECKING → AI_ANALYSIS
↓
Trigger Re-analysis
```

A reject dari satu Checker langsung menghentikan current checking round.

Checker lain tidak perlu melanjutkan approval terhadap analysis version yang sudah rejected.

---

# 12. Checker Round Rule

Checker decisions berlaku hanya untuk satu analysis version.

Example:

```text
Analysis v1
├── Risk Checker APPROVED
└── Dev Checker REJECTED
```

Result:

```text
Analysis v1 review round ends
↓
AI Re-analysis
↓
Analysis v2
```

Decision pada v1 tidak dibawa ke v2.

Untuk v2:

```text
Risk Checker → PENDING
Dev Checker  → PENDING
```

Semua required Checker harus mereview analysis v2 kembali.

---

# 13. Signer Flow

Initial state:

```text
SIGNING
```

Preconditions:

```text
actor assigned as SIGNER
analysis_id = case.current_analysis_id
all required Checker approved current analysis
```

---

## 13.1 Signer Approve

Flow:

```text
Insert APPROVE Decision
↓
Create Decision Snapshot
↓
Audit SIGNER_APPROVED
↓
SIGNING → EXECUTION
```

Decision Snapshot mencatat:

```text
case ID
case version/context
analysis ID
analysis version
policy versions
evidence references
Checker approvals
Signer
recommendation
timestamp
```

Result:

```text
case.status = EXECUTION
```

---

## 13.2 Signer Reject

Flow:

```text
Insert REJECT Decision
↓
Store Reviewer Feedback
↓
Audit SIGNER_REJECTED
↓
SIGNING → AI_ANALYSIS
↓
Trigger Re-analysis
```

Setelah new analysis dibuat:

```text
AI_ANALYSIS → CHECKING
```

Workflow harus melewati Checker lagi sebelum kembali ke Signer.

---

# 14. Execution Flow

Initial state:

```text
EXECUTION
```

Preconditions:

```text
actor assigned as EXECUTER
current analysis approved by Signer
case.status = EXECUTION
```

Executer memulai execution:

```text
Create execution
status = IN_PROGRESS
Audit EXECUTION_STARTED
```

---

# 15. Execution Success

Flow:

```text
Execution IN_PROGRESS
↓
Executer submits SUCCESS
↓
Persist action_taken
Persist result
↓
Audit EXECUTION_SUCCESS
↓
EXECUTION → DONE
↓
Audit CASE_DONE
```

Result:

```text
case.status = DONE
```

---

# 16. Execution Blocked

`BLOCKED` berarti action belum dapat dilakukan.

Contoh:

```text
Required dependency unavailable
Required upstream file unavailable
Required system access unavailable
External party not ready
```

Flow:

```text
Execution IN_PROGRESS
↓
Executer submits BLOCKED
↓
Store blocker
Store execution evidence
↓
Audit EXECUTION_BLOCKED
↓
EXECUTION → AI_ANALYSIS
↓
Trigger Re-analysis
```

Execution blocker menjadi evidence untuk analysis version berikutnya.

---

# 17. Execution Failed

`FAILED` berarti action sudah dicoba tetapi tidak berhasil.

Flow:

```text
Execution IN_PROGRESS
↓
Executer submits FAILED
↓
Store action_taken
Store result
Store blocker
Store execution evidence
↓
Audit EXECUTION_FAILED
↓
EXECUTION → AI_ANALYSIS
↓
Trigger Re-analysis
```

---

# 18. Re-analysis Flow

Re-analysis trigger:

```text
CHECKER_REJECTED
SIGNER_REJECTED
EXECUTION_BLOCKED
EXECUTION_FAILED
MANUAL_REANALYZE
```

Flow:

```text
New Feedback / Evidence
↓
AI_ANALYSIS
↓
Build Current Context
↓
Retrieve Current ACTIVE Policy
↓
Generate New Analysis Version
↓
Verify
↓
Set cases.current_analysis_id
↓
CHECKING
```

Analysis lama tidak dihapus.

---

# 19. Analysis Version Rule

Example:

```text
Analysis v1
↓ rejected

Analysis v2
↓ approved + execution blocked

Analysis v3
↓ approved + execution success
```

Setiap version menyimpan independent references ke:

```text
Evidence
Policy Version
AI Model
Prompt Version
Verification Result
```

Approval selalu terikat ke exact `analysis_id`.

---

# 20. Stale Analysis Rule

Decision request harus membawa:

```text
analysis_id
```

Backend membandingkan:

```text
request.analysis_id
        ==
case.current_analysis_id
```

Jika berbeda:

```http
409 Conflict
```

Error:

```text
STALE_ANALYSIS
```

Workflow tidak berubah.

Frontend harus refetch current case dan current analysis.

Approval tidak boleh di-retry otomatis.

---

# 21. Concurrent Decision Rule

Contoh:

```text
Risk Checker membuka Analysis v2
Dev Checker membuka Analysis v2

Dev Checker REJECT
↓
case → AI_ANALYSIS
↓
Analysis v3 generated

Risk Checker kemudian menekan APPROVE pada v2
```

Request Risk Checker harus ditolak:

```text
STALE_ANALYSIS
```

Karena:

```text
requested analysis = v2
current analysis   = v3
```

---

# 22. Approval Validity Rule

Approval valid hanya jika:

```text
decision.analysis_id = cases.current_analysis_id
```

Decision dari analysis lama tetap disimpan untuk audit tetapi tidak memiliki authority terhadap current analysis.

---

# 23. Policy Change During Workflow

Analysis menyimpan exact policy version yang digunakan.

Jika active policy berubah setelah analysis dibuat:

```text
Existing analysis remains historically valid
```

Tetapi jika re-analysis terjadi:

```text
New analysis uses current applicable ACTIVE policy
```

Current policy version tidak mengganti reference pada historical analysis.

---

# 24. Evidence Rule

Evidence dapat berasal dari:

```text
MAKER
CHECKER
SIGNER
EXECUTER
SYSTEM
```

Evidence type:

```text
COMMENT
DOCUMENT
LOG
SCREENSHOT
REFERENCE
EXECUTION_RESULT
```

Evidence lama tidak dihapus dari historical decision context.

New evidence dapat memicu re-analysis melalui allowed business action.

---

# 25. Reviewer Feedback Rule

Reviewer feedback adalah context/evidence.

Reviewer feedback **bukan authoritative policy**.

AI harus memperlakukan feedback sebagai:

```text
REVIEWER_FEEDBACK
```

Jika feedback bertentangan dengan active policy:

```text
AI surfaces conflict
```

AI tidak mengubah policy authority berdasarkan reviewer opinion.

---

# 26. Close Flow

`CLOSED` digunakan ketika workflow tidak perlu dilanjutkan.

Required data:

```text
closed_by
close_reason
closed_at
```

Examples:

```text
Duplicate
False alarm
Resolved externally
No action required
```

Close event:

```text
Persist Close Reason
↓
Audit CASE_CLOSED
↓
case → CLOSED
```

`CLOSED` bukan execution success.

---

# 27. DONE vs CLOSED

## DONE

```text
Authorized action executed successfully
```

## CLOSED

```text
Workflow stopped without successful execution completion
```

Keduanya harus tampil berbeda di UI dan audit history.

---

# 28. Escalation Flow

Automatic re-analysis limit:

```text
MAX_REANALYSIS = 3
```

Jika limit tercapai:

```text
Audit REANALYSIS_LIMIT_REACHED
↓
case → ESCALATION_REQUIRED
```

Other escalation trigger:

```text
Repeated AI failure
Unresolved policy conflict
Workflow deadlock
```

`ESCALATION_REQUIRED` membutuhkan manual human intervention.

---

# 29. AI Authority Boundary

AI boleh:

```text
Analyze
Retrieve Policy
Classify Facts / Assumptions / Unknowns
Assess Risk
Assess Compliance
Recommend
Generate Alternatives
Identify Missing Information
Verify Analysis
Re-analyze
```

AI tidak memiliki authority untuk:

```text
Approve
Reject on behalf of human
Sign
Execute
Assign Participant
Change Workflow State
Activate Policy
Close Case
Delete Audit History
```

---

# 30. Backend Authority Boundary

Backend bertanggung jawab atas:

```text
Authentication
Authorization
Participant Assignment Validation
Segregation of Duties
State Transition
Approval Validity
Analysis Version Validity
Transaction Integrity
Policy Lifecycle
Audit Persistence
```

---

# 31. Frontend Responsibility

Frontend bertanggung jawab untuk:

```text
Display Current State
Display Available Action
Collect User Input
Send Business Action
Render Backend Result
Handle Conflict / Stale State
```

Frontend tidak menghitung next state secara authoritative.

---

# 32. Audit Events

Workflow-related audit event:

```text
CASE_CREATED
CASE_UPDATED
CASE_SUBMITTED
CASE_CLOSED
CASE_DONE

PARTICIPANT_ASSIGNED
PARTICIPANT_UNASSIGNED

EVIDENCE_ADDED

AI_ANALYSIS_STARTED
AI_ANALYSIS_COMPLETED
AI_ANALYSIS_FAILED
REANALYSIS_LIMIT_REACHED

CHECKER_APPROVED
CHECKER_REJECTED

SIGNER_APPROVED
SIGNER_REJECTED

EXECUTION_STARTED
EXECUTION_BLOCKED
EXECUTION_FAILED
EXECUTION_SUCCESS
```

Audit event bersifat append-only pada application layer.

---

# 33. Transaction Boundaries

Critical workflow mutation harus atomic.

Example Checker Reject:

```text
BEGIN

Validate state
Validate actor
Validate current analysis
Insert decision
Insert reviewer feedback evidence
Insert audit event
Update case state → AI_ANALYSIS

COMMIT

Trigger AI re-analysis
```

External AI request tidak dijalankan di dalam open database transaction.

---

# 34. Full Happy Path

```text
DRAFT
│
│ Maker Submit
▼
SUBMITTED
│
│ Start Analysis
▼
AI_ANALYSIS
│
│ Analysis v1 Success
▼
CHECKING
│
│ All Required Checker Approve
▼
SIGNING
│
│ Signer Approve
▼
EXECUTION
│
│ Execution Success
▼
DONE
```

---

# 35. Checker Reject Path

```text
AI_ANALYSIS
│
│ Analysis v1
▼
CHECKING
│
│ Checker Reject
▼
AI_ANALYSIS
│
│ Analysis v2
▼
CHECKING
```

---

# 36. Signer Reject Path

```text
CHECKING
│
│ All Checker Approve
▼
SIGNING
│
│ Signer Reject
▼
AI_ANALYSIS
│
│ Analysis v2
▼
CHECKING
│
│ All Checker Re-review
▼
SIGNING
```

---

# 37. Execution Blocked Path

```text
SIGNING
│
│ Signer Approve
▼
EXECUTION
│
│ BLOCKED
▼
AI_ANALYSIS
│
│ Analysis v3
▼
CHECKING
│
│ All Checker Approve
▼
SIGNING
│
│ Signer Approve
▼
EXECUTION
```

---

# 38. MVP Demo Workflow

MVP demo menggunakan flow berikut:

```text
Maker creates case
↓
Maker assigns:
- Risk Checker
- Development Checker
- Manager Signer
- Maker as Executer
↓
Maker submits case
↓
AI Analysis v1 generated
↓
Risk Checker rejects
↓
AI Analysis v2 generated
↓
Risk Checker approves
↓
Development Checker approves
↓
Signer approves
↓
Executer starts execution
↓
Executer marks BLOCKED
↓
AI Analysis v3 generated
↓
Risk Checker approves
↓
Development Checker approves
↓
Signer approves
↓
Executer marks SUCCESS
↓
Case becomes DONE
```

Workflow demo harus memperlihatkan:

```text
Analysis Versioning
Policy Grounding
Checker Reject
Re-analysis
Multi-Checker Approval
Signer Authorization
Execution Blocker
Second Re-analysis
Successful Execution
Audit History
```

---

# 39. Workflow Invariants

Invariant berikut harus selalu benar:

```text
1. Case hanya memiliki satu current analysis.
2. Approval hanya valid untuk current analysis.
3. SIGNING hanya dapat terjadi setelah semua required Checker approve.
4. EXECUTION hanya dapat terjadi setelah Signer approve.
5. DONE hanya dapat terjadi setelah execution SUCCESS.
6. BLOCKED/FAILED selalu kembali ke AI_ANALYSIS.
7. Reject selalu menghasilkan re-analysis sebelum review berikutnya.
8. Historical analysis tidak di-overwrite.
9. Historical decision tidak dihapus.
10. AI tidak dapat mengubah workflow state.
11. Frontend tidak dapat mengubah workflow state secara langsung.
12. Segregation of duties selalu divalidasi backend.
```

---

# 40. Related Documents

API contract:

```text
jawir-sentinel-docs/api/api-contract.md
```

Backend implementation:

```text
jawir-sentinel-be/README.md
```

Frontend implementation:

```text
jawir-sentinel-fe/README.md
```

Dokumen ini menjadi source of truth untuk workflow behavior JAWIR Sentinel MVP.
