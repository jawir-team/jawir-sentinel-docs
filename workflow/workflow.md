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
- dapat dianalisis ulang hanya melalui governed business event: Checker reject, Signer reject, Execution blocked, atau Execution failed;
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

## 3.1 Participant Cardinality

Participant contract per case:

```text
MAKER
exactly 1 active
creator of the case
immutable assignment

CHECKER
1..N active
at least 1 required Checker before submit

SIGNER
exactly 1 active before submit

EXECUTER
exactly 1 active before submit
```

`MAKER = EXECUTER` tetap diperbolehkan.

## 3.2 Governance Snapshot / Submission Freeze Rule

Selama `DRAFT`:

```text
case core data     editable by Maker
Checker assignment editable
Signer assignment  editable
Executer assignment editable
evidence           may be added
```

Maker dibuat otomatis dari case creator dan tidak dapat di-unassign atau diganti.

Saat `SUBMIT`, backend memvalidasi participant cardinality + SoD dan participant set menjadi **frozen governance context** untuk case tersebut.

Setelah case meninggalkan `DRAFT`:

```text
case core data      read-only
Maker               frozen
Checker(s)          frozen
Signer              frozen
Executer            frozen

evidence            may continue to grow
analysis             versioned
decisions            append-only
executions           historical attempts
audit                append-only
```

Participant assignment/unassignment setelah submit tidak diperbolehkan.

Jika participant harus diganti karena salah assignment, tidak tersedia, pindah tanggung jawab, atau alasan lain:

```text
Close existing case with reason
↓
Existing case remains CLOSED and historically intact
↓
Maker creates a new case
↓
Assign new participant set
↓
Submit new case
```

MVP tidak memiliki reopen, participant replacement, atau role transfer pada case yang sudah berjalan.

New case dapat menambahkan `REFERENCE` evidence yang merujuk case lama bila dibutuhkan untuk traceability, tetapi tidak ada parent/replacement relation khusus pada schema MVP.

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

Case berhenti dari automated governed workflow karena Sentinel tidak dapat melanjutkan secara aman dalam batas MVP.

MVP escalation cause hanya:

```text
VERIFIER_FAIL
TECHNICAL_RETRY_EXHAUSTED
REANALYSIS_LIMIT_REACHED
```

`POLICY_CONFLICT` bukan standalone workflow trigger. Policy conflict dapat berkontribusi pada verifier `FAIL`, dan escalation terjadi melalui `VERIFIER_FAIL`.

Allowed activity:

```text
View case
View analysis history
View audit history
Add evidence
Close case
```

Not allowed:

```text
Approve
Reject
Sign
Execute
Re-analyze
Resume workflow
Reset retry/re-analysis quota
Edit case core data
Change participants
```

MVP tidak memiliki resume/reopen dari `ESCALATION_REQUIRED`. Jika masalah perlu diproses kembali, participant menutup case lama dengan reason lalu Maker membuat case baru; case baru dapat menambahkan REFERENCE evidence ke case lama.

---

# 7. State Transition Table

| Current State | Event | Next State |
|---|---|---|
| DRAFT | SUBMIT | SUBMITTED |
| SUBMITTED | START_ANALYSIS | AI_ANALYSIS |
| AI_ANALYSIS | ANALYSIS_SUCCESS | CHECKING |
| AI_ANALYSIS | ANALYSIS_FAILED | ESCALATION_REQUIRED |
| AI_ANALYSIS | REANALYSIS_LIMIT_REACHED | ESCALATION_REQUIRED |
| CHECKING | ALL_CHECKERS_APPROVED | SIGNING |
| CHECKING | CHECKER_REJECTED | AI_ANALYSIS |
| SIGNING | SIGNER_APPROVED | EXECUTION |
| SIGNING | SIGNER_REJECTED | AI_ANALYSIS |
| EXECUTION | EXECUTION_SUCCESS | DONE |
| EXECUTION | EXECUTION_BLOCKED | AI_ANALYSIS |
| EXECUTION | EXECUTION_FAILED | AI_ANALYSIS |

CLOSE dapat dilakukan oleh active workflow participant dengan role `MAKER`, `CHECKER`, `SIGNER`, atau `EXECUTER`.

Allowed source state:

```text
DRAFT
SUBMITTED
AI_ANALYSIS
CHECKING
SIGNING
EXECUTION
ESCALATION_REQUIRED
```

`DONE` dan `CLOSED` tidak dapat di-close kembali.

---

# 8. Workflow Event Catalog

```text
SUBMIT
START_ANALYSIS
ANALYSIS_SUCCESS
ANALYSIS_FAILED
REANALYSIS_LIMIT_REACHED

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

exactly 1 active Maker
at least 1 active required Checker
exactly 1 active Signer
exactly 1 active Executer

segregation of duties valid
```

Flow:

```text
Validate Case
↓
Validate Participant Cardinality
↓
Validate Segregation of Duties
↓
Freeze Case Core Data + Participant Set
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
Current ACTIVE + READY Policies
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

Technical retry policy is configured at runtime:

```text
AI_TECHNICAL_MAX_RETRIES=<non-negative integer>
```

Semantics:

```text
initial provider/model attempt
+ up to AI_TECHNICAL_MAX_RETRIES retry attempts
= one business analysis cycle / one analysis version
```

Retryable technical failures include timeout, transient provider failure, and invalid structured model response that is explicitly retried by the analysis orchestration.

Verifier `FAIL` is a semantic verification result and is not retried as a technical provider failure.

Verifier terminal failure:

```text
Persist analysis FAILED
Preserve valid structured analysis output
verification_status = FAIL
↓
Audit AI_ANALYSIS_FAILED
metadata.failure_type = VERIFIER_FAIL
↓
ANALYSIS_FAILED
↓
AI_ANALYSIS → ESCALATION_REQUIRED
```

If the technical retry budget is exhausted:

```text
Persist analysis FAILED
↓
Audit AI_ANALYSIS_FAILED
metadata.failure_type = TECHNICAL_RETRY_EXHAUSTED
↓
ANALYSIS_FAILED
↓
AI_ANALYSIS → ESCALATION_REQUIRED
```

Technical retries never consume `MAX_REANALYSIS` and never create a new analysis version.

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
Check MAX_REANALYSIS

if quota available:
  COMMIT
  ↓
  Trigger Re-analysis after commit

if quota exhausted:
  Audit REANALYSIS_LIMIT_REACHED
  AI_ANALYSIS → ESCALATION_REQUIRED
  COMMIT
  ↓
  Do not trigger AI
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
Check MAX_REANALYSIS

if quota available:
  COMMIT
  ↓
  Trigger Re-analysis after commit

if quota exhausted:
  Audit REANALYSIS_LIMIT_REACHED
  AI_ANALYSIS → ESCALATION_REQUIRED
  COMMIT
  ↓
  Do not trigger AI
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
Check MAX_REANALYSIS

if quota available:
  COMMIT
  ↓
  Trigger Re-analysis after commit

if quota exhausted:
  Audit REANALYSIS_LIMIT_REACHED
  AI_ANALYSIS → ESCALATION_REQUIRED
  COMMIT
  ↓
  Do not trigger AI
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
Check MAX_REANALYSIS

if quota available:
  COMMIT
  ↓
  Trigger Re-analysis after commit

if quota exhausted:
  Audit REANALYSIS_LIMIT_REACHED
  AI_ANALYSIS → ESCALATION_REQUIRED
  COMMIT
  ↓
  Do not trigger AI
```

---

# 18. Re-analysis Flow

Re-analysis trigger:

```text
CHECKER_REJECTED
SIGNER_REJECTED
EXECUTION_BLOCKED
EXECUTION_FAILED
```

MVP tidak menyediakan generic/manual re-analysis action atau endpoint. Evidence baru tidak otomatis membuat analysis version baru; reviewer/executer menggunakan business action yang sesuai bila evidence tersebut mengubah decision context.

Flow when quota is available:

```text
Governed Feedback / Execution Result
↓
AI_ANALYSIS
↓
COMMIT triggering business transaction
↓
Build Current Context
↓
Retrieve Current ACTIVE + READY Policy
↓
Generate New Analysis Version
↓
Verify
↓
Set cases.current_analysis_id only on success
↓
CHECKING
```

Flow when quota is exhausted:

```text
Governed Feedback / Execution Result is persisted
↓
AI_ANALYSIS
↓
REANALYSIS_LIMIT_REACHED
↓
ESCALATION_REQUIRED
↓
COMMIT
↓
No new analysis version
No AI call
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

`cases.current_analysis_id` memiliki semantics khusus:

```text
latest successfully COMPLETED analysis
with verification PASS or PASS_WITH_WARNING
that became eligible for human review
```

Failed analysis attempt tidak pernah mengganti `current_analysis_id`.

Karena itu pada `ESCALATION_REQUIRED` dapat terjadi:

```text
current_analysis_id = v1 COMPLETED
latest analysis attempt = v2 FAILED
```

Latest attempt ditentukan dari highest persisted `ai_analyses.version`, bukan dari `current_analysis_id`.

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

## 24.1 User Evidence Authorization

User-created evidence hanya dapat ditambahkan oleh active participant pada state berikut:

```text
DRAFT
→ Maker only

CHECKING
→ any active case participant

SIGNING
→ any active case participant

EXECUTION
→ any active case participant

ESCALATION_REQUIRED
→ any active case participant
```

User evidence tidak dapat ditambahkan pada:

```text
SUBMITTED
AI_ANALYSIS
DONE
CLOSED
```

Rationale:

- `SUBMITTED` adalah transient internal state;
- `AI_ANALYSIS` menjaga analysis context tidak berubah ketika AI cycle sedang berjalan;
- `DONE` dan `CLOSED` adalah terminal/read-only state.

## 24.2 Actor Role

Untuk user-created evidence, request membawa:

```text
actor_role
```

Backend wajib memvalidasi bahwa authenticated user memang memiliki active assignment dengan role tersebut pada case.

Client tidak mengirim authoritative `source_type`. Backend menetapkan:

```text
source_user_id = authenticated user
source_type    = validated actor_role
```

Ini diperlukan terutama ketika satu user memiliki lebih dari satu role yang valid, misalnya `MAKER = EXECUTER`.

`SYSTEM` evidence hanya dapat dibuat oleh internal backend process dan tidak dapat dipilih melalui user-facing evidence endpoint.

## 24.3 Evidence Effect on Workflow

Evidence lama tidak dihapus dari historical decision context.

Menambahkan evidence **tidak** mengubah case state dan **tidak** otomatis memicu re-analysis.

Jika evidence baru mengubah decision context, re-analysis hanya dapat terjadi melalui governed business action:

```text
Checker REJECT
Signer REJECT
Execution BLOCKED
Execution FAILED
```

Pada `ESCALATION_REQUIRED`, evidence dapat ditambahkan untuk audit/manual investigation tetapi tidak otomatis melanjutkan workflow.

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

Semua active workflow participant pada case dapat melakukan close:

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

Allowed source state:

```text
DRAFT
SUBMITTED
AI_ANALYSIS
CHECKING
SIGNING
EXECUTION
ESCALATION_REQUIRED
```

`DONE` dan `CLOSED` tidak dapat di-close kembali.

Close reason wajib berupa penjelasan non-empty mengenai alasan workflow dihentikan.

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

Close mutation harus menggunakan current-state validation dan row locking.

Audit `CASE_CLOSED` minimal menyimpan:

```text
actor_id
actor_role
previous_status
close_reason
closed_at
```

Close event:

```text
Validate Active Participant
↓
Validate Non-Terminal State
↓
Persist Close Reason
↓
Audit CASE_CLOSED
↓
case → CLOSED
```

Setelah case menjadi `CLOSED`, asynchronous AI completion atau business action lain tidak boleh membuka atau memindahkan state case kembali. Analysis/execution record yang sudah terbentuk tetap dipertahankan sebagai historical record dan tidak otomatis diubah menjadi successful result.

`CLOSED` bukan execution success.

Jika case perlu mengganti participant setelah submission, case lama harus di-close dengan alasan yang jelas dan Maker membuat case baru. Participant pada case lama tidak diganti.

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

Automatic business re-analysis limit:

```text
MAX_REANALYSIS = 3
```

Semantics:

```text
Analysis v1
= initial analysis
= reanalysis_count 0

Analysis v2
= re-analysis #1

Analysis v3
= re-analysis #2

Analysis v4
= re-analysis #3
```

Setelah v4 sudah ada, request business action yang membutuhkan analysis version berikutnya akan mencapai limit:

```text
Requested re-analysis #4
↓
Audit REANALYSIS_LIMIT_REACHED
↓
case → ESCALATION_REQUIRED
```

Formula MVP:

```text
reanalysis_count = latest_analysis_version - 1
```

Technical retry terhadap model/provider **tidak** menambah analysis version dan **tidak** menambah re-analysis count.

Config runtime:

```text
AI_TECHNICAL_MAX_RETRIES=<non-negative integer>
```

`AI_TECHNICAL_MAX_RETRIES` berarti jumlah retry **setelah initial attempt**.

Contoh:

```text
AI_TECHNICAL_MAX_RETRIES=2

attempt 1 = initial
attempt 2 = retry #1
attempt 3 = retry #2
```

Contoh technical retry:

```text
Vertex AI timeout
temporary provider error
malformed model response yang di-retry dalam analysis cycle yang sama
```

Satu business analysis cycle hanya memperoleh satu `ai_analyses.version`. Retry teknis tetap berada pada cycle/version yang sama.

Jika seluruh technical attempt habis tanpa usable result, current analysis menjadi `FAILED`, `AI_ANALYSIS_FAILED` diaudit, dan case masuk `ESCALATION_REQUIRED`.

Re-analysis quota exhaustion:

```text
governed business trigger requests next analysis
↓
MAX_REANALYSIS already exhausted
↓
do not allocate another analysis version
↓
Audit REANALYSIS_LIMIT_REACHED
metadata includes latest_analysis_version + max_reanalysis
↓
REANALYSIS_LIMIT_REACHED
↓
case → ESCALATION_REQUIRED
```

MVP tidak memiliki standalone escalation event untuk policy conflict atau generic workflow deadlock.

`ESCALATION_REQUIRED` adalah stopped state tanpa resume. Active participant masih dapat menambah evidence untuk investigation dan dapat menutup case.

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

Seluruh workflow event di atas menggunakan `audit_events.scope_type = CASE`.

Case audit minimal memiliki `case_id`; `analysis_id` dan `actor_role` diisi jika relevan. SYSTEM event dapat memiliki `actor_role = NULL`.

Policy lifecycle audit menggunakan `scope_type = POLICY` dan didefinisikan pada database/architecture contract. Policy event tidak menggunakan fake `case_id`.

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
13. Case core data dan participant set immutable setelah meninggalkan DRAFT.
14. Participant replacement pada in-flight case tidak tersedia; perubahan personel membutuhkan close + new case.
15. Maker assignment adalah creator dan immutable.
16. Re-analysis hanya dapat dipicu oleh CHECKER_REJECTED, SIGNER_REJECTED, EXECUTION_BLOCKED, atau EXECUTION_FAILED.
17. Manual/generic re-analysis tidak tersedia pada MVP.
18. User evidence mutation hanya tersedia pada DRAFT/CHECKING/SIGNING/EXECUTION/ESCALATION_REQUIRED sesuai role authorization.
19. Evidence mutation tidak pernah mengubah workflow state secara langsung.
20. current_analysis_id hanya menunjuk COMPLETED PASS/PASS_WITH_WARNING analysis; FAILED attempt tidak mengganti pointer.
21. AI terminal failure menggunakan ANALYSIS_FAILED dengan cause VERIFIER_FAIL atau TECHNICAL_RETRY_EXHAUSTED.
22. Re-analysis quota exhaustion menggunakan REANALYSIS_LIMIT_REACHED dan tidak membuat analysis version baru.
23. ESCALATION_REQUIRED tidak dapat resume pada MVP; allowed user mutations hanya evidence dan close sesuai authorization.
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
