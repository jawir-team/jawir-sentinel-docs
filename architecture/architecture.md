# JAWIR Sentinel Architecture

**Specification Version:** 1.0  
**Status:** MVP Baseline  
**Repository Path:** `jawir-sentinel-docs/architecture/architecture.md`

Dokumen ini mendefinisikan arsitektur resmi JAWIR Sentinel MVP.

Arsitektur dirancang untuk:

- menjaga workflow tetap governed;
- memastikan backend menjadi business authority;
- memisahkan UI, business logic, AI reasoning, dan persistence;
- mendukung auditability;
- mendukung analysis versioning;
- menjaga policy grounding;
- memudahkan deployment pada Google Cloud;
- tetap sederhana untuk tim kecil dan competition timeline.

---

# 1. Architecture Principle

Prinsip utama sistem:

> **AI generates intelligence. Backend enforces control. Humans hold authority. Data preserves accountability.**

Konsekuensinya:

```text
Frontend
→ presents state and captures intent

Backend
→ validates rules and controls workflow

AI
→ analyzes, retrieves, recommends, verifies

Database
→ persists current state and historical accountability
```

AI tidak menjadi workflow authority.

Frontend tidak menjadi workflow authority.

Backend adalah satu-satunya komponen yang mengontrol business transition.

---

# 2. System Context

JAWIR Sentinel terdiri dari:

```text
Human User
    │
    ▼
Frontend Web Application
    │
    ▼
Backend API
    │
    ├── PostgreSQL / Cloud SQL
    ├── Google Cloud Storage
    ├── Vertex AI Gemini
    └── Firebase Authentication
```

External managed services:

```text
Firebase Authentication
Google Cloud Run
Google Cloud SQL
Google Cloud Storage
Vertex AI
```

---

# 3. High-Level Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                        HUMAN USERS                          │
│                                                             │
│ Maker       Checker       Signer       Executer      Admin  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  JAWIR SENTINEL FRONTEND                    │
│                                                             │
│ Next.js + TypeScript                                        │
│ Cloud Run                                                   │
│                                                             │
│ - Authentication UI                                         │
│ - Dashboard                                                 │
│ - Case UI                                                   │
│ - AI Analysis UI                                            │
│ - Review UI                                                 │
│ - Execution UI                                              │
│ - Policy Management UI                                      │
│ - Audit Timeline                                            │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS / JSON
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   JAWIR SENTINEL BACKEND                    │
│                                                             │
│ Go + Chi                                                    │
│ Cloud Run                                                   │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ API / Auth Middleware                                   │ │
│ └──────────────────────────┬──────────────────────────────┘ │
│                            │                                │
│ ┌──────────────────────────▼──────────────────────────────┐ │
│ │ Application / Domain Services                          │ │
│ │                                                       │ │
│ │ Case        Workflow        Review       Execution     │ │
│ │ Policy      Evidence        Audit        User/Unit     │ │
│ └──────────────────────────┬──────────────────────────────┘ │
│                            │                                │
│ ┌──────────────────────────▼──────────────────────────────┐ │
│ │ AI Orchestration                                       │ │
│ │                                                       │ │
│ │ Context Builder                                       │ │
│ │ Policy Retrieval                                      │ │
│ │ Gemini Analysis                                       │ │
│ │ Structured Validation                                 │ │
│ │ Gemini Verification                                   │ │
│ └──────────────────────────┬──────────────────────────────┘ │
└───────────────┬────────────┼───────────────┬────────────────┘
                │            │               │
                ▼            ▼               ▼
      ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐
      │ Cloud SQL      │ │ Cloud Storage│ │ Vertex AI Gemini │
      │ PostgreSQL     │ │ Evidence     │ │ Analysis         │
      │ + pgvector     │ │ Files        │ │ Verification     │
      └────────────────┘ └──────────────┘ └──────────────────┘
```

---

# 4. Repository Architecture

GitHub Organization:

```text
JAWIR
├── jawir-sentinel-be
├── jawir-sentinel-fe
└── jawir-sentinel-docs
```

Responsibility:

```text
jawir-sentinel-be
→ backend implementation
→ business logic
→ AI orchestration
→ persistence
→ deployment backend

jawir-sentinel-fe
→ frontend implementation
→ user interaction
→ state presentation
→ deployment frontend

jawir-sentinel-docs
→ product specification
→ workflow contract
→ database contract
→ API contract
→ architecture contract
```

Docs repository menjadi single source of truth untuk contract lintas repository.

---

# 5. Deployment Architecture

```text
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │ sentinel-web   │
              │ Cloud Run      │
              └───────┬────────┘
                      │ HTTPS
                      ▼
              ┌────────────────┐
              │ sentinel-api   │
              │ Cloud Run      │
              └───┬─────┬──────┘
                  │     │
          ┌───────┘     └──────────────┐
          ▼                            ▼
┌──────────────────┐        ┌────────────────────┐
│ Cloud SQL        │        │ Vertex AI Gemini   │
│ PostgreSQL       │        │ + Embedding Model  │
│ pgvector         │        └────────────────────┘
└────────┬─────────┘
         │
         │
         ▼
┌──────────────────┐
│ Cloud Storage    │
│ Evidence Files   │
└──────────────────┘
```

Authentication:

```text
Browser
  ↓
Firebase Authentication
  ↓
Firebase ID Token
  ↓
sentinel-api verifies token
```

---

# 6. Frontend Architecture

Frontend stack:

```text
Next.js
TypeScript
App Router
Tailwind CSS
TanStack Query
React Hook Form
Zod
Firebase Authentication
```

Frontend architecture:

```text
Page / Route
    │
    ▼
Feature Component
    │
    ▼
Feature Hook
    │
    ▼
API Service
    │
    ▼
Backend API
```

Frontend responsibilities:

```text
Navigation
Authentication UX
Form Input
Server State Rendering
Workflow Action Presentation
Error / Conflict Handling
File Upload UX
```

Frontend tidak menyimpan business truth secara independen.

Server state menggunakan TanStack Query.

Local state hanya digunakan untuk:

```text
Dialog
Tabs
Temporary Form State
UI Preference
```

---

# 7. Backend Architecture

Backend menggunakan modular monolith.

```text
Go Application
│
├── Auth
├── User
├── Unit
├── Case Type
├── Case
├── Workflow
├── Policy
├── Evidence
├── Analysis
├── Review
├── Execution
├── Audit
└── AI
```

MVP hanya memiliki satu backend deployable service:

```text
sentinel-api
```

Tidak ada microservice split pada MVP.

---

# 8. Backend Layering

Logical layers:

```text
HTTP Handler
    ↓
Application Service
    ↓
Domain / Workflow Rule
    ↓
Repository / SQL
    ↓
PostgreSQL
```

AI path:

```text
Application Service
    ↓
AI Orchestrator
    ↓
Context Builder
    ↓
Policy Retriever
    ↓
Vertex AI
    ↓
Structured Output Validator
    ↓
Verifier
```

HTTP handler tidak berisi core business rule.

SQL repository tidak menentukan workflow transition.

Workflow rules berada pada application/domain layer.

---

# 9. Backend Package Boundary

```text
internal/
├── auth
├── unit
├── user
├── casetype
├── case
├── workflow
├── policy
├── evidence
├── analysis
├── review
├── execution
├── audit
└── ai
```

Dependency direction:

```text
handler
  ↓
service
  ↓
workflow/domain
  ↓
repository
```

Cross-module call dilakukan melalui service interface yang jelas.

---

# 10. Workflow Architecture

Workflow state machine berada di backend.

```text
internal/workflow/
├── states.go
├── events.go
├── guards.go
└── transition.go
```

Flow:

```text
Business Action
    ↓
Validate Current State
    ↓
Validate Actor
    ↓
Validate Analysis Version
    ↓
Validate Segregation of Duties
    ↓
Persist Business Event
    ↓
Apply Transition
    ↓
Audit
```

Frontend tidak mengirim:

```text
next_state
```

Frontend mengirim business action seperti:

```text
submit
approve
reject
execute
close
```

---

# 11. Authentication Architecture

Identity provider:

```text
Firebase Authentication
```

Flow:

```text
User Login
  ↓
Firebase Authentication
  ↓
Firebase ID Token
  ↓
Frontend API Request
  ↓
Backend Middleware
  ↓
Verify Token
  ↓
Extract firebase_uid
  ↓
Lookup users.firebase_uid
  ↓
Attach Internal User Context
```

Backend authorization tidak bergantung pada frontend state.

---

# 12. Authorization Architecture

Authorization rule menggunakan:

```text
Authenticated User
Case Participant Assignment
Participant Role
Current Case State
Current Analysis Version
is_admin
Segregation of Duties
```

Example:

```text
Checker Approve Request
↓
authenticated?
↓
assigned CHECKER?
↓
case CHECKING?
↓
analysis current?
↓
decision not already submitted?
↓
allow
```

---

# 13. Data Architecture

Primary database:

```text
PostgreSQL
```

Cloud service:

```text
Google Cloud SQL
```

Extension:

```text
pgvector
```

Data terbagi menjadi:

```text
Master Data
Workflow Current State
Historical Decision Data
AI Structured Data
Vector Retrieval Data
Audit Data
```

---

# 14. Current State Data

Current state:

```text
cases.status
cases.current_analysis_id
case_participants.status
policy_versions.status
executions.status
```

Data ini dapat berubah sesuai workflow.

---

# 15. Historical Data

Historical data:

```text
ai_analyses
decisions
executions
audit_events
policy_versions
analysis_policy_refs
analysis_evidence_refs
case_evidences
```

Historical data tidak di-overwrite untuk menggantikan decision context lama.

---

# 16. AI Architecture

AI subsystem:

```text
AI Orchestrator
│
├── Context Builder
├── Policy Retriever
├── Analysis Generator
├── Output Validator
└── Verifier
```

Model provider:

```text
Vertex AI Gemini
```

Embedding provider:

```text
Vertex AI Embedding Model
```

---

# 17. AI Context Builder

Context Builder mengumpulkan:

```text
Current Case
Current Evidence
Current Workflow State
Current ACTIVE Policy
Latest Reviewer Feedback
Latest Execution Feedback
Previous Analysis Summary
```

Context tidak mengambil seluruh data repository.

Hanya context relevan yang dikirim ke model.

---

# 18. Policy Retrieval Architecture

Policy retrieval menggunakan:

```text
PostgreSQL
+
pgvector
```

Flow:

```text
Case
  ↓
Extract Retrieval Context
  ↓
Filter by Case Type / Domain
  ↓
Filter ACTIVE + Effective Policy
  ↓
Generate Query Embedding
  ↓
Vector Similarity Search
  ↓
Top Relevant Chunks
  ↓
Build Policy Context
```

Policy source of authority:

```text
policy_versions.status = ACTIVE
```

Reviewer feedback bukan policy authority.

---

# 19. Policy Ingestion Architecture

Policy creation:

```text
Create Policy
  ↓
Create DRAFT Version
  ↓
Activate Version
  ↓
Version becomes ACTIVE
  ↓
Chunk Content
  ↓
Generate Embeddings
  ↓
Insert policy_chunks
  ↓
Ready for Retrieval
```

Historical old version:

```text
ACTIVE
→ SUPERSEDED
```

Historical analysis tetap reference ke version lama yang digunakan saat decision dibuat.

---

# 20. AI Analysis Flow

```text
Case enters AI_ANALYSIS
        │
        ▼
Create analysis execution context
        │
        ▼
Retrieve evidence
        │
        ▼
Retrieve active policy chunks
        │
        ▼
Build prompt/context
        │
        ▼
Gemini Analysis
        │
        ▼
Validate structured JSON
        │
        ▼
Gemini Verifier
        │
        ├── PASS
        ├── PASS_WITH_WARNING
        └── FAIL
```

If:

```text
PASS
PASS_WITH_WARNING
```

then:

```text
Persist Analysis
Persist Policy References
Persist Evidence References
Update current_analysis_id
Transition → CHECKING
```

If:

```text
FAIL
```

analysis does not enter Checker review.

---

# 21. AI Output Boundary

AI boleh menghasilkan:

```text
Summary
Facts
Assumptions
Unknowns
Risk Analysis
Compliance Analysis
Recommendation
Alternatives
Missing Information
Evidence Quality
Uncertainty
Verification Result
```

AI tidak boleh:

```text
Change Case State
Approve
Reject Human Decision
Sign
Execute
Assign Participant
Activate Policy
Close Case
Delete Audit Data
```

---

# 22. Re-analysis Architecture

Trigger:

```text
Checker Reject
Signer Reject
Execution Blocked
Execution Failed
Manual Re-analysis
```

Flow:

```text
New Feedback / Evidence
↓
Transition → AI_ANALYSIS
↓
Build Fresh Context
↓
Retrieve Current ACTIVE Policies
↓
Generate New Analysis Version
↓
Verify
↓
Persist
↓
Update current_analysis_id
↓
CHECKING
```

Previous analysis hanya historical context.

Approval tidak diwariskan ke version baru.

---

# 23. Concurrency Architecture

Critical mutation menggunakan database transaction dan row lock.

Pattern:

```sql
SELECT ...
FROM cases
WHERE id = $1
FOR UPDATE;
```

Digunakan pada:

```text
submit
checker decision
signer decision
execution result
close
analysis completion
```

Tujuan:

```text
Prevent duplicate transition
Prevent concurrent invalid approval
Prevent race on current_analysis_id
```

---

# 24. Stale Analysis Protection

Decision request selalu membawa:

```text
analysis_id
```

Backend check:

```text
request.analysis_id
==
cases.current_analysis_id
```

Jika tidak:

```text
409 STALE_ANALYSIS
```

Frontend:

```text
Refetch Case
Refetch Current Analysis
Require Human Re-review
```

---

# 25. Transaction Architecture

Critical mutation dilakukan dalam short-lived transaction.

Example Checker Reject:

```text
BEGIN

Lock Case
Validate State
Validate Actor
Validate Analysis
Insert Decision
Insert Feedback Evidence
Insert Audit Event
Update Case State

COMMIT
```

Setelah commit:

```text
Trigger AI Re-analysis
```

External AI call tidak dijalankan dalam open DB transaction.

---

# 26. File Storage Architecture

Evidence binary:

```text
Google Cloud Storage
```

Metadata:

```text
PostgreSQL
```

Upload flow:

```text
Frontend
  ↓
Request Signed URL
  ↓
Backend
  ↓
Generate Signed Upload URL
  ↓
Frontend
  ↓
Direct Upload to Cloud Storage
  ↓
Frontend
  ↓
Register Evidence Metadata
  ↓
Backend
  ↓
PostgreSQL
```

Backend tidak menjadi proxy untuk large binary upload.

---

# 27. Evidence Architecture

Evidence sources:

```text
MAKER
CHECKER
SIGNER
EXECUTER
SYSTEM
```

Evidence types:

```text
COMMENT
DOCUMENT
LOG
SCREENSHOT
REFERENCE
EXECUTION_RESULT
```

AI menggunakan evidence sebagai context.

Evidence tidak memiliki policy authority.

---

# 28. Audit Architecture

Audit data disimpan pada:

```text
audit_events
```

Audit sifatnya:

```text
append-only
```

Event example:

```text
CASE_CREATED
CASE_SUBMITTED
AI_ANALYSIS_COMPLETED
CHECKER_REJECTED
SIGNER_APPROVED
EXECUTION_BLOCKED
CASE_DONE
```

Audit dapat merekonstruksi:

```text
Who
Did What
Against Which Analysis
At What Time
With What Result
```

---

# 29. Decision Snapshot

Saat Signer approve, backend menyimpan decision snapshot pada audit metadata.

Snapshot:

```text
Case ID
Analysis ID
Analysis Version
Policy Versions
Evidence References
Checker Approvals
Signer
Recommendation
Timestamp
```

Tujuan:

```text
Decision reproducibility
```

---

# 30. Error Architecture

Backend error contract:

```json
{
  "error": {
    "code": "STALE_ANALYSIS",
    "message": "Analysis version is no longer current.",
    "details": {}
  }
}
```

Error categories:

```text
Validation
Authentication
Authorization
Not Found
Conflict
AI Failure
Internal Failure
```

Frontend menginterpretasikan `error.code`.

---

# 31. Reliability Architecture

MVP reliability principles:

```text
DB commit before external AI call
Short-lived transactions
Retry external AI selectively
Persist AI failure state
Do not lose case on AI failure
Bound re-analysis loop
Preserve audit trail
```

Maximum re-analysis:

```text
MAX_REANALYSIS = 3
```

Jika limit tercapai:

```text
ESCALATION_REQUIRED
```

---

# 32. AI Failure Handling

Failure types:

```text
Vertex AI unavailable
Timeout
Invalid structured output
Verifier failure
Embedding generation failure
Policy indexing failure
```

Case tidak dihapus atau di-reset.

Failure state dicatat.

AI analysis dapat:

```text
retry
or
escalate
```

sesuai configured limit.

---

# 33. Observability Architecture

Application menggunakan structured logging.

Required fields:

```text
timestamp
level
request_id
user_id
case_id
analysis_id
component
message
```

AI log:

```text
case_id
analysis_version
model_name
latency_ms
status
token_usage
```

Evidence content sensitif tidak ditulis ke logs.

---

# 34. Request Correlation

Setiap inbound request memiliki:

```text
request_id
```

Jika client tidak mengirim request ID, backend membuat UUID.

Request ID diteruskan ke:

```text
application logs
AI logs
error logs
```

Audit event tetap menggunakan domain event ID sendiri.

---

# 35. Security Architecture

Security boundary:

```text
Browser
  ↓
Firebase Authentication
  ↓
Backend Authorization
  ↓
Controlled Service Access
```

Frontend tidak menyimpan:

```text
Service Account Credential
Database Credential
Vertex AI Credential
Cloud SQL Credential
```

Backend service account memiliki access minimum yang diperlukan.

---

# 36. Service Permissions

`sentinel-api` membutuhkan permission untuk:

```text
Cloud SQL connect
Cloud Storage signed URL / object access
Vertex AI inference
Vertex AI embedding
Firebase token verification
```

`sentinel-web` tidak membutuhkan direct permission ke:

```text
Cloud SQL
Vertex AI
Backend Storage Credential
```

---

# 37. Data Security

MVP menggunakan synthetic data.

Tidak menggunakan:

```text
real customer data
real account data
real transaction data
production banking secrets
```

Evidence demo juga synthetic.

---

# 38. Prompt Injection Boundary

Attachment dan evidence diperlakukan sebagai:

```text
data
```

bukan:

```text
system instruction
```

AI prompt harus memisahkan:

```text
System Rules
Policy Context
Case Context
Evidence
Reviewer Feedback
```

Policy authority ditentukan backend metadata, bukan isi prompt.

---

# 39. Network Architecture

Logical network flow:

```text
Browser
  ↓ HTTPS
Cloud Run Web
  ↓ HTTPS
Cloud Run API
  ↓
Cloud SQL
Cloud Storage
Vertex AI
```

Public client tidak terhubung langsung ke database.

---

# 40. CI/CD Architecture

## Frontend

```text
Push / Merge
  ↓
GitHub Actions
  ↓
Lint
  ↓
Typecheck
  ↓
Test
  ↓
Next.js Build
  ↓
Docker Build
  ↓
Cloud Run Deploy
```

Service:

```text
sentinel-web
```

---

## Backend

```text
Push / Merge
  ↓
GitHub Actions
  ↓
Go Test
  ↓
Lint
  ↓
Build
  ↓
Docker Build
  ↓
Cloud Run Deploy
```

Service:

```text
sentinel-api
```

---

# 41. Environment Architecture

Environments:

```text
DEV
PROD
```

DEV:

```text
development
integration testing
mock/demo validation
```

PROD:

```text
competition demo
submission
```

Each environment memiliki:

```text
separate configuration
separate database
separate storage path/bucket configuration
separate deployment
```

---

# 42. Configuration Architecture

Backend env:

```text
APP_ENV
APP_PORT
DATABASE_URL

FIREBASE_PROJECT_ID

GCP_PROJECT_ID
GCP_REGION
GCS_BUCKET

VERTEX_AI_LOCATION
VERTEX_AI_MODEL
VERTEX_EMBEDDING_MODEL

MAX_REANALYSIS
POLICY_RETRIEVAL_TOP_K
```

Frontend env:

```text
NEXT_PUBLIC_API_BASE_URL
NEXT_PUBLIC_FIREBASE_API_KEY
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN
NEXT_PUBLIC_FIREBASE_PROJECT_ID
NEXT_PUBLIC_FIREBASE_APP_ID
```

---

# 43. Sequence — Create and Submit Case

```text
Maker
  │
  ▼
Frontend
  │ POST /cases
  ▼
Backend
  │
  ├── Validate Input
  ├── Insert Case
  ├── Insert Maker Participant
  └── Audit CASE_CREATED
  │
  ▼
PostgreSQL

Maker
  │
  ▼
Frontend
  │ POST /cases/{id}/submit
  ▼
Backend
  │
  ├── Validate Participants
  ├── Validate SoD
  ├── Lock Case
  ├── Transition → SUBMITTED
  ├── Audit
  ├── Transition → AI_ANALYSIS
  └── Commit
  │
  ▼
AI Orchestrator
```

---

# 44. Sequence — AI Analysis

```text
Backend
  │
  ▼
Context Builder
  │
  ├── Case
  ├── Evidence
  ├── Reviewer Feedback
  └── Execution Feedback
  │
  ▼
Policy Retriever
  │
  ├── Metadata Filter
  ├── Embedding
  └── pgvector Search
  │
  ▼
Vertex AI Gemini
  │
  ▼
Structured Output
  │
  ▼
Validator
  │
  ▼
Verifier
  │
  ▼
Backend Transaction
  │
  ├── Insert Analysis
  ├── Insert Policy Refs
  ├── Insert Evidence Refs
  ├── Update current_analysis_id
  ├── Transition → CHECKING
  └── Audit
```

---

# 45. Sequence — Checker Reject

```text
Checker
  │
  ▼
Frontend
  │ POST checker decision
  ▼
Backend
  │
  ├── Verify Auth
  ├── Lock Case
  ├── Validate CHECKING
  ├── Validate Checker Assignment
  ├── Validate analysis_id
  ├── Insert REJECT Decision
  ├── Insert Feedback Evidence
  ├── Audit
  ├── Transition → AI_ANALYSIS
  └── Commit
  │
  ▼
AI Re-analysis
```

---

# 46. Sequence — Signer Approve

```text
Signer
  │
  ▼
Frontend
  │ POST signer decision
  ▼
Backend
  │
  ├── Verify Auth
  ├── Lock Case
  ├── Validate SIGNING
  ├── Validate Signer Assignment
  ├── Validate Current Analysis
  ├── Validate All Checker Approvals
  ├── Insert APPROVE Decision
  ├── Store Decision Snapshot
  ├── Audit
  ├── Transition → EXECUTION
  └── Commit
```

---

# 47. Sequence — Execution Blocked

```text
Executer
  │
  ▼
Frontend
  │ POST execution result
  ▼
Backend
  │
  ├── Lock Case
  ├── Validate EXECUTION
  ├── Validate Executer
  ├── Update Execution → BLOCKED
  ├── Insert Execution Evidence
  ├── Audit
  ├── Transition → AI_ANALYSIS
  └── Commit
  │
  ▼
AI Re-analysis
```

---

# 48. Scalability Boundary

MVP menggunakan modular monolith.

Scale path jika dibutuhkan di masa depan:

```text
Frontend remains independent
Backend modules can be extracted later
AI orchestration can become separate service
Policy retrieval can move to dedicated search service
Audit can move to dedicated storage
```

MVP tidak melakukan split tersebut.

---

# 49. Model Provider Boundary

AI orchestration tidak mengikat core workflow langsung ke Gemini API call.

Logical boundary:

```text
Workflow
  ↓
AI Orchestrator
  ↓
Model Adapter
  ↓
Gemini / Future Model
```

MVP implementation:

```text
Gemini via Vertex AI
```

Future provider dapat diganti tanpa mengubah workflow state machine.

---

# 50. Architecture Invariants

Invariant berikut harus selalu benar:

```text
1. Frontend never directly changes workflow state.
2. AI never directly changes workflow state.
3. Backend is the workflow authority.
4. Every decision is tied to an analysis version.
5. Historical analysis is immutable.
6. Historical policy reference is immutable.
7. Only ACTIVE applicable policies are authoritative.
8. External AI calls do not run inside open DB transactions.
9. Critical workflow mutation uses transaction + row locking.
10. File binary is stored in Cloud Storage, metadata in PostgreSQL.
11. Browser never connects directly to PostgreSQL.
12. Browser never calls Vertex AI directly.
13. Audit events are append-only.
14. Authorization is always enforced server-side.
15. Re-analysis creates a new analysis version.
16. New analysis invalidates prior approval authority.
17. DONE only follows successful execution.
18. BLOCKED/FAILED returns workflow to AI_ANALYSIS.
19. AI failure does not delete or reset the case.
20. Docs define the contract; FE and BE implement it.
```

---

# 51. Related Documents

Workflow:

```text
jawir-sentinel-docs/workflow/workflow.md
```

Database:

```text
jawir-sentinel-docs/database/database-design.md
```

API Contract:

```text
jawir-sentinel-docs/api/api-contract.md
```

Backend Implementation:

```text
jawir-sentinel-be/README.md
```

Frontend Implementation:

```text
jawir-sentinel-fe/README.md
```

Dokumen ini menjadi source of truth untuk system architecture JAWIR Sentinel MVP.
