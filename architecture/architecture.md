# Arsitektur JAWIR Sentinel

**Versi Spesifikasi:** 1.0  
**Status:** MVP Baseline  
**Path Repository:** `jawir-sentinel-docs/architecture/architecture.md`

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

# 1. Prinsip Arsitektur

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

# 2. Konteks Sistem

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
    ├── Firebase Authentication
    └── Transactional Outbox
             │
             ▼
          RabbitMQ
             │
             ▼
      Sentinel AI Worker
             │
             └── Vertex AI Gemini
```

Runtime/external services:

```text
Firebase Authentication
Google Cloud Run Service      → sentinel-api
Google Cloud Run Worker Pool  → sentinel-worker
Google Cloud SQL
Google Cloud Storage
Vertex AI
RabbitMQ broker
```

PostgreSQL tetap menjadi workflow truth. RabbitMQ hanya membawa pekerjaan dan tidak pernah menjadi workflow authority.

---

# 3. Arsitektur Tingkat Tinggi

```text
┌──────────────────────────────────────────────────────────────┐
│ HUMAN USERS                                                  │
│ Maker       Checker       Signer       Executer       Admin  │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
                    ┌────────────────────┐
                    │ sentinel-web       │
                    │ Next.js / Cloud Run│
                    └─────────┬──────────┘
                              │ HTTPS
                              ▼
                    ┌────────────────────┐
                    │ sentinel-api       │
                    │ Go / Cloud Run     │
                    │                    │
                    │ Workflow authority │
                    │ DB transactions    │
                    │ Outbox writer      │
                    └───┬────────────┬───┘
                        │            │
                        ▼            ▼
              ┌────────────────┐  ┌────────────────┐
              │ Cloud SQL      │  │ Cloud Storage  │
              │ PostgreSQL     │  │ Evidence Files │
              │ + pgvector     │  └────────────────┘
              └───────┬────────┘
                      │
             outbox_events (durable)
                      │
                      ▼
             ┌──────────────────┐
             │ sentinel-worker  │
             │ Worker Pool      │
             │ Outbox Dispatcher│
             │ Rabbit Consumer  │
             └────┬────────┬────┘
                  │        │
           publish│        │consume
                  ▼        │
             ┌──────────┐  │
             │ RabbitMQ │◄─┘
             │ quorum   │
             │ queue    │
             └────┬─────┘
                  │
                  ▼
             AI job consumer
                  │
                  ├── Context Builder
                  ├── Policy Retrieval
                  ├── Gemini Analysis
                  ├── Structured Validation
                  └── Gemini Verification
                  │
                  ▼
             Vertex AI Gemini
```

Reliability boundary:

```text
business transaction
→ workflow mutation + GENERATING analysis + audit + outbox row
→ COMMIT
→ dispatcher publishes with publisher confirms
→ RabbitMQ durable quorum queue
→ worker consumes with manual acknowledgement
→ finalization transaction
→ ACK only after durable finalization
```

Delivery RabbitMQ bersifat at-least-once. Karena itu logic consumer/finalization harus idempotent.

---

# 4. Arsitektur Repository

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

Docs repository menjadi single sumber kebenaran untuk contract lintas repository.

---

# 5. Arsitektur Deployment

```text
Internet
   │
   ▼
sentinel-web
Cloud Run Service
   │ HTTPS
   ▼
sentinel-api
Cloud Run Service
   │
   ├──────────────► Cloud SQL PostgreSQL + pgvector
   ├──────────────► Cloud Storage
   └──────────────► Firebase token verification

Cloud SQL outbox_events
   │
   ▼
sentinel-worker
Cloud Run Worker Pool
   │
   ├── Outbox Dispatcher ──► RabbitMQ
   └── AI Consumer ◄──────── RabbitMQ
                              │
                              ▼
                        Vertex AI Gemini
```

Backend repository menghasilkan satu container image dengan dua runtime mode:

```text
server  → sentinel-api
worker  → sentinel-worker
```

Ini bukan pemisahan microservice atas business ownership. Kedua mode menggunakan domain, repository, workflow, dan kontrak database yang sama.

Topology RabbitMQ production wajib memakai durable queue dan broker/cluster yang fault-tolerant. Aplikasi menerima koneksi broker melalui `RABBITMQ_URL`; hosting broker mengikuti environment.

Development lokal dapat menjalankan RabbitMQ melalui Docker Compose.

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

# 6. Arsitektur Frontend

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

Tanggung jawab frontend:

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

# 7. Arsitektur Backend

Backend menggunakan satu modular-monolith codebase dengan dua runtime modes.

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
├── Outbox
├── Messaging
└── AI
```

Deployables:

```text
sentinel-api
→ HTTP/API
→ workflow/business transactions
→ writes outbox_events
→ never performs long AI calls in request transaction

sentinel-worker
→ continuous background runtime
→ dispatches outbox to RabbitMQ
→ consumes AI jobs
→ calls Vertex AI
→ finalizes analysis
```

Business authority tetap berada pada shared application/domain service dan PostgreSQL.

---

# 8. Layering Backend

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

# 9. Batas Package Backend

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
├── outbox
├── messaging
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

# 10. Arsitektur Workflow

Workflow state machine berada di backend.

```text
internal/workflow/
├── states.go
├── events.go
├── guards.go
└── transition.go
```

Alur:

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

## 10.1 Snapshot Governance Case

Case menggunakan logical governance snapshot pada saat submission.

Tidak ada table snapshot participant terpisah pada MVP. Snapshot dijaga dengan rule immutability:

```text
DRAFT
→ case core data editable
→ Checker / Signer / Executer assignment editable

SUBMIT
→ validate cardinality
→ validate SoD
→ freeze case core data
→ freeze participant set

SUBMITTED and later
→ participant mutation forbidden
→ case core mutation forbidden
```

Locked cardinality:

```text
Maker     exactly 1, creator, immutable
Checker   1..N, at least 1 required
Signer    exactly 1
Executer  exactly 1
```

Data yang tetap berkembang selama workflow:

```text
evidence
AI analysis versions
human decisions
execution attempts
audit events
```

Jika participant pada case berjalan harus diganti, backend tidak melakukan role transfer. Existing case di-close dengan reason, lalu Maker membuat case baru dengan participant context yang baru.

---

# 11. Arsitektur Authentication

Identity provider:

```text
Firebase Authentication
```

Alur:

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

# 12. Arsitektur Authorization

Sentinel memisahkan **system role** dari **case workflow role**.

System role:

```text
USER
ADMIN
```

Case role:

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

`ADMIN` is not a case-participant role and does not bypass workflow SoD.

Authorization matrix:

```text
Read units / case types / safe user directory
→ any authenticated ACTIVE user

Create/update units, users, case types
→ ADMIN

Create/version/activate policy
→ ADMIN

Create case
→ any authenticated ACTIVE user
→ creator becomes Maker + owner

Read a case
→ active case participant OR ADMIN

Mutate case workflow
→ exact assigned case role + state/analysis guards
→ ADMIN alone gives no workflow authority
```

Backend authorization evaluates:

```text
Authenticated User
users.system_role
Case Participant Assignment
Participant Role
Current Case State
Current Analysis Version
Segregation of Duties
```

Contoh:

```text
Checker Approve Request
↓
authenticated ACTIVE user?
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

# 13. Arsitektur Data

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
Operational Outbox Data
```

---

# 14. Data State Saat Ini

State saat ini:

```text
cases.status
cases.current_analysis_id
case_participants.status
policy_versions.status
executions.status
```

`cases.current_analysis_id` **bukan** pointer untuk attempt terbaru. Artinya:

```text
latest COMPLETED analysis
with PASS / PASS_WITH_WARNING verification
that became eligible for human review
```

Analysis FAILED yang lebih baru dapat ada sementara `current_analysis_id` masih menunjuk analysis berhasil sebelumnya, atau tetap NULL jika belum pernah ada analysis yang selesai dengan sukses.

Attempt terbaru ditentukan dari `ai_analyses.version` tertinggi yang tersimpan.

Data ini dapat berubah sesuai workflow.

---

# 15. Data Historis

Data historis:

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

Data historis tidak di-overwrite untuk menggantikan decision context lama.

---

# 16. Arsitektur AI

AI subsystem dijalankan oleh `sentinel-worker`.

```text
RabbitMQ Consumer
│
├── Analysis Job Guard
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
Vertex AI
model                  = gemini-embedding-001
output dimensionality  = 768
document task type     = RETRIEVAL_DOCUMENT
query task type        = RETRIEVAL_QUERY
distance               = cosine
index                   = HNSW
```

Queue correctness:

- identitas message menggunakan `outbox_events.id` yang sudah dipersist;
- payload berisi exact `case_id` dan `analysis_id`;
- duplicate/redelivered message diperbolehkan;
- worker memeriksa persisted analysis status sebelum bekerja;
- finalization row-lock/state guards ensure at most one durable outcome;
- duplicate consumer yang menemukan analysis sudah COMPLETED/FAILED melakukan ACK tanpa state mutation.

Kontrak embedding tetap independen dari transport RabbitMQ.

---

# 17. AI Context Builder

Context Builder membentuk decision context yang tepat untuk satu analysis version yang tersimpan.

Inputs:

```text
Current Case Snapshot
Current Evidence
Current ACTIVE + READY Policy Chunks
Latest Reviewer Feedback
Latest Execution Feedback
Previous Analysis Summary
Current Workflow State
```

Evidence handling:

```text
text evidence
→ prompt text/data part

supported file evidence
→ Gemini fileData using GCS URI + MIME type
```

MVP multimodal allowlist:

```text
application/pdf
image/jpeg
image/png
```

Tidak diperlukan custom OCR/extraction pipeline untuk file evidence yang didukung. MIME yang tidak didukung ditolak oleh file-evidence flow, bukan diam-diam dihilangkan dari AI context.

Seluruh content evidence/file diperlakukan sebagai **data** yang tidak dipercaya, bukan system instruction.

Context Builder mempertahankan provenance ID agar facts/recommendations yang dihasilkan dapat merujuk source policy/evidence yang tepat.

---

# 18. Arsitektur Policy Retrieval

Policy retrieval menggunakan:

```text
PostgreSQL
+
pgvector VECTOR(768)
+
HNSW cosine index
```

Embedding configuration:

```text
policy chunk → gemini-embedding-001 / RETRIEVAL_DOCUMENT / 768
query        → gemini-embedding-001 / RETRIEVAL_QUERY    / 768
```

Alur:

```text
Case / Retrieval Context
  ↓
Receive case_type_id + retrieval_text
  ↓
Filter ACTIVE + READY + Effective Policy
  ↓
Filter matching case_type_id OR generic policy
  ↓
Generate Query Embedding
  ↓
HNSW Cosine Similarity Search
  ↓
Top 8 Chunks
  ↓
Return Chunks + Provenance + Score
```

MVP candidate policy rule:

```text
policy_versions.status = ACTIVE

AND
policy_versions.index_status = READY

AND
effective_from <= now() OR effective_from IS NULL

AND
effective_until > now() OR effective_until IS NULL

AND
(
  policies.case_type_id = cases.case_type_id
  OR policies.case_type_id IS NULL
)
```

`policies.domain` tetap disimpan sebagai metadata, tetapi **bukan hard filter** pada MVP karena case tidak memiliki domain field.

Retrieval boundary:

- Context Builder bertanggung jawab membentuk `retrieval_text`.
- Policy Retriever menerima `case_type_id` + `retrieval_text`.
- Retriever tidak merakit business context sendiri.
- Query embedding menggunakan `gemini-embedding-001`, task `RETRIEVAL_QUERY`, dimension 768.
- Search menggunakan HNSW + cosine distance.
- `POLICY_RETRIEVAL_TOP_K=8`.
- MVP tidak menggunakan hard similarity threshold.
- MVP tidak menggunakan reranker kedua.
- Retriever mengembalikan chunk + complete provenance + distance/relevance score.
- Retriever tidak menentukan `POLICY_FOUND`, `NO_POLICY_FOUND`, `INSUFFICIENT_EVIDENCE`, atau `POLICY_CONFLICT`; status tersebut ditentukan pada analysis/verifier layer.

Policy source of authority for retrieval:

```text
policy_versions.status = ACTIVE
AND policy_versions.index_status = READY
```

Reviewer feedback bukan policy authority.

---

# 19. Arsitektur Policy Ingestion

Pembuatan dan aktivasi policy:

```text
Create Policy
  ↓
Create DRAFT Version
index_status = NOT_STARTED
  ↓
Activation Requested
  ↓
validate target effective NOW
validate effective_until > effective_from when both exist
  ↓
claim index attempt
index_status = PROCESSING
index_attempt_id = new UUID
index_started_at = now()
  ↓
Normalize Content
  ↓
Detect Section / Heading
  ↓
Split by Section and Paragraph
  ↓
Build Chunks
  ↓
Generate All Embeddings
  ↓
Persist Complete policy_chunks
  ↓
index_status = READY
  ↓
Atomic Policy Activation
  ↓
old ACTIVE → SUPERSEDED
target DRAFT → ACTIVE
  ↓
Ready for Retrieval
```

Jika chunking atau embedding gagal pada attempt yang sedang memegang claim:

```text
target.status       = DRAFT
target.index_status = FAILED
target.index_error  = safe diagnostic summary
target.indexed_at   = NULL

current ACTIVE version remains ACTIVE
```

PROCESSING recovery:

```text
POLICY_INDEX_LEASE_SECONDS controls stale threshold.

PROCESSING + lease not stale
→ reject duplicate activation

PROCESSING + lease stale
→ claim a new index_attempt_id
→ old attempt loses write authority
→ re-index
```

Setiap write READY/FAILED wajib membandingkan expected `index_attempt_id`. Worker terlambat dari attempt lama tidak boleh menimpa claim yang lebih baru.

Aktivasi final memvalidasi ulang bahwa target masih `DRAFT + READY` dan effective **saat transaction berjalan** sebelum men-supercede ACTIVE version lama.

Version yang effective di masa depan atau sudah expired tidak dapat diaktifkan pada MVP. Tidak ada scheduled activation service.

External embedding request tidak pernah dijalankan di dalam final policy activation transaction.

MVP chunking contract:

```text
strategy      = section-aware + paragraph-aware
max_chunk     = 600 tokens
overlap       = 100 tokens
chunk_index   = global sequential per policy version
reprocessing  = replace derived chunks for exact policy version
```

Rules:

- section yang lebih kecil dari batas tidak dipaksa mencapai 600 tokens;
- section besar dipecah pada paragraph boundary selama memungkinkan;
- overlap 100 tokens hanya digunakan ketika satu section harus dipecah menjadi lebih dari satu chunk;
- heading detector mendukung Markdown heading dan numbered heading seperti `4.2 Approval Requirement`;
- jika heading tidak terdeteksi, paragraph boundary menjadi fallback;
- `section` menyimpan heading terdekat bila tersedia;
- `chunk_index` tidak reset per section;
- chunking harus deterministic untuk content dan configuration yang sama;
- `policy_versions.content` tetap source content; `policy_chunks` adalah derived retrieval data.

Jika exact policy version perlu di-index ulang:

```text
Delete derived chunks for policy_version_id
↓
Regenerate chunks using current locked chunking contract
↓
Generate embeddings
↓
Insert replacement chunks
```

Version lama historis:

```text
ACTIVE
→ SUPERSEDED
```

Analysis historis tetap reference ke version lama yang digunakan saat decision dibuat.

---

# 20. Alur Analisis AI

Queueing phase:

```text
business transaction
↓
case → AI_ANALYSIS
↓
allocate ai_analyses row = GENERATING
↓
audit AI_ANALYSIS_STARTED
↓
insert outbox_events.AI_ANALYSIS_REQUESTED
↓
COMMIT
```

Delivery phase:

```text
Outbox Dispatcher
↓ publisher confirm
RabbitMQ quorum queue
↓ manual delivery
sentinel-worker
```

Worker phase:

```text
Load + lock analysis by analysis_id
↓
if status != GENERATING → ACK / no-op
↓
claim worker_attempt_id + worker_started_at

fresh duplicate with active non-stale claim
→ ACK duplicate / no-op

redelivered message OR stale worker lease
→ rotate worker_attempt_id
→ previous worker loses finalization authority
↓
Build current context
↓
Retrieve ACTIVE + READY policy chunks
↓
Attach supported GCS evidence files directly to Gemini
↓
Gemini Analysis
↓
Validate structured JSON
↓
Gemini Verifier
```

PASS / PASS_WITH_WARNING:

```text
finalization transaction
→ persist complete analysis + provenance
→ update current_analysis_id
→ ANALYSIS_SUCCESS
→ CHECKING
→ COMMIT
→ RabbitMQ ACK
```

Verifier FAIL:

```text
finalization transaction
→ persist FAILED analysis + valid candidate + provenance
→ audit AI_ANALYSIS_FAILED / VERIFIER_FAIL
→ ANALYSIS_FAILED
→ ESCALATION_REQUIRED
→ COMMIT
→ RabbitMQ ACK
```

Technical retry exhaustion:

```text
finalization transaction
→ analysis FAILED
→ audit AI_ANALYSIS_FAILED / TECHNICAL_RETRY_EXHAUSTED
→ ANALYSIS_FAILED
→ ESCALATION_REQUIRED
→ COMMIT
→ RabbitMQ ACK
```

Jika finalisasi gagal sebelum commit, worker tidak melakukan ACK; RabbitMQ dapat melakukan redelivery.

Setiap DB mutation milik worker wajib memvalidasi `worker_attempt_id` saat ini, termasuk increment `technical_retry_count` dan finalisasi. Superseded worker yang terlambat boleh menyelesaikan external model call, tetapi tidak boleh mengonsumsi retry budget, menulis hasil analysis, atau mengubah workflow state.

`AI_WORKER_LEASE_SECONDS` controls stale worker-claim recovery. `technical_retry_count` remains persisted on the analysis and is never reset by redelivery/reclaim.

---

# 21. Batas Output AI

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

# 22. Arsitektur Re-analysis

Trigger:

```text
Checker Reject
Signer Reject
Execution Blocked
Execution Failed
```

MVP hanya mengenal empat governed trigger di atas. Tidak ada direct/generic re-analysis command. Evidence baru saja tidak otomatis membuat analysis version baru.

Alur ketika quota tersedia:

```text
Governed reject/block/fail business transaction
↓
Transition → AI_ANALYSIS
↓
Allocate next GENERATING analysis
↓
Audit AI_ANALYSIS_STARTED
↓
Insert PENDING transactional outbox event
↓
COMMIT
↓
Outbox Dispatcher → RabbitMQ
↓
Worker claims exact analysis_id
↓
Build Fresh Context
↓
Retrieve Current ACTIVE + READY Policies
↓
Generate / Validate / Verify
↓
Finalize exact analysis
↓
CHECKING on success
```

Jika quota habis, business action pemicu tetap dipersist dan case masuk `ESCALATION_REQUIRED` tanpa membuat analysis/outbox row.

MVP re-analysis semantics:

```text
MAX_REANALYSIS = 3

v1 = initial analysis, not counted
v2 = re-analysis #1
v3 = re-analysis #2
v4 = re-analysis #3

request for next business re-analysis
→ REANALYSIS_LIMIT_REACHED
→ ESCALATION_REQUIRED
```

Formula:

```text
reanalysis_count = latest_analysis_version - 1
```

Technical model/provider retry tidak membuat analysis version baru dan tidak mengonsumsi re-analysis quota. Retry teknis tetap berada pada business analysis cycle/version yang sama.

Analysis sebelumnya hanya menjadi konteks historis.

Approval tidak diwariskan ke version baru.

---

# 23. Arsitektur Concurrency

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
Prevent participant-assignment vs user-deactivation race
```

Participant assignment dan mutation user ACTIVE→INACTIVE wajib terserialisasi pada target user row (atau transaction lock setara) agar user INACTIVE tidak pernah menjadi active case participant.

---

# 24. Proteksi Stale Analysis

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

# 25. Arsitektur Transaction

Mutation workflow kritis dan pembuatan AI job bersifat atomic melalui transactional outbox.

Initial submit example:

```text
BEGIN

Lock Case
Validate DRAFT + participants + SoD
Freeze governance context
DRAFT → SUBMITTED → AI_ANALYSIS
Create Analysis v1 = GENERATING
Audit CASE_SUBMITTED
Audit AI_ANALYSIS_STARTED
Insert outbox AI_ANALYSIS_REQUESTED(case_id, analysis_id)

COMMIT
```

Governed re-analysis example:

```text
BEGIN

Lock Case
Persist reject/block/fail business record
Apply business event → AI_ANALYSIS
Check MAX_REANALYSIS

if quota available:
  allocate next GENERATING analysis
  audit AI_ANALYSIS_STARTED
  insert outbox AI_ANALYSIS_REQUESTED

if quota exhausted:
  audit REANALYSIS_LIMIT_REACHED
  AI_ANALYSIS → ESCALATION_REQUIRED
  no analysis row
  no outbox row

COMMIT
```

HTTP request tidak menjalankan Vertex call setelah transaction. Durable outbox memastikan intent tetap ada meskipun API/process gagal.

Delivery outbox bersifat at-least-once. Duplicate delivery RabbitMQ aman karena worker finalization memvalidasi exact `analysis_id`, melakukan row lock pada case, dan hanya memfinalisasi analysis GENERATING satu kali.

---

# 26. Arsitektur File Storage

Evidence file flow:

```text
Client
  ↓
Request signed upload URL
  ↓
Backend validates actor/state/MIME
  ↓
Backend returns signed URL + case-scoped file key
  ↓
Client uploads directly to Cloud Storage
  ↓
Client registers evidence
  ↓
Backend revalidates state/actor
  ↓
Backend verifies object exists + case prefix + MIME
  ↓
Persist evidence metadata
```

MVP file MIME allowlist:

```text
application/pdf
image/jpeg
image/png
```

Evidence yang tersimpan mencakup `mime_type`.

Saat membentuk AI context, backend mengubah case-scoped GCS object menjadi URI `gs://...` dan mengirimkannya ke Gemini sebagai file data. Main API tidak pernah mem-proxy binary dan tidak memerlukan custom OCR pipeline untuk allowlist ini.

MIME yang tidak didukung ditolak, bukan disimpan sebagai evidence yang tidak terlihat oleh AI.

---

# 27. Arsitektur Evidence

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

Otorisasi user evidence mengikuti state:

```text
DRAFT               → Maker only
CHECKING             → any active participant
SIGNING              → any active participant
EXECUTION            → any active participant
ESCALATION_REQUIRED  → any active participant

SUBMITTED / AI_ANALYSIS / DONE / CLOSED
→ no user evidence mutation
```

Request user tidak perlu membawa role yang dapat dipercaya dari client. Backend memvalidasi active case assignment authenticated user lalu melakukan derivasi source role:

```text
source_user_id
source_type
```

`SYSTEM` evidence is internal-only and cannot be selected by the client.

Write evidence bersifat append-only pada MVP dan tidak langsung mengubah workflow state.

Evidence yang ditambahkan saat CHECKING/SIGNING/EXECUTION dapat memengaruhi governed action berikutnya, tetapi hanya Checker REJECT, Signer REJECT, Execution BLOCKED, atau Execution FAILED yang dapat memicu analysis version baru.

AI menggunakan evidence sebagai context.

Evidence tidak memiliki policy authority.

---

# 28. Arsitektur Audit

Audit data disimpan pada satu table:

```text
audit_events
```

Audit sifatnya:

```text
append-only
scoped
historically traceable
```

Supported scope:

```text
CASE
POLICY
```

CASE scope digunakan untuk workflow event dan dapat mereferensikan:

```text
case_id
analysis_id
actor_id
actor_role
```

POLICY scope digunakan untuk policy lifecycle event dan dapat mereferensikan:

```text
policy_id
policy_version_id
actor_id
```

Policy event tidak menggunakan fake `case_id`. Pada POLICY scope, `actor_role` adalah NULL; authorization admin berasal dari `users.system_role = ADMIN`, bukan role workflow.

Alasan kegagalan AI pada case disimpan di audit metadata, bukan di column baru pada case:

```text
AI_ANALYSIS_FAILED
metadata.failure_type =
  VERIFIER_FAIL
  | TECHNICAL_RETRY_EXHAUSTED

REANALYSIS_LIMIT_REACHED
metadata includes:
  latest_analysis_version
  max_reanalysis
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

POLICY_CREATED
POLICY_VERSION_CREATED
POLICY_ACTIVATED
POLICY_SUPERSEDED
```

Case audit dapat merekonstruksi:

```text
Who
Did What
Against Which Analysis
At What Time
With What Result
```

Policy audit dapat merekonstruksi:

```text
Who
Changed Which Policy / Version
At What Time
With What Lifecycle Result
```

`GET /cases/{case_id}/history` hanya membaca `scope_type = CASE`.

Policy history endpoint tidak wajib untuk MVP; policy audit tetap dipersist untuk accountability.

---

# 29. Snapshot Decision

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

# 30. Arsitektur Error

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

# 31. Arsitektur Reliability

MVP reliability principles:

```text
PostgreSQL is workflow truth
Business state + AI enqueue intent commit atomically
RabbitMQ delivery is at-least-once
Consumer/finalization is idempotent
External AI calls never run in open DB transactions
Retry technical AI failures selectively
Preserve audit/history
Bound technical retry loop
Bound business re-analysis loop
```

Transactional outbox:

```text
API transaction
→ outbox_events.status = PENDING
→ COMMIT

sentinel-worker dispatcher
→ publish persistent message
→ wait for RabbitMQ publisher confirm
→ mark outbox PUBLISHED
```

Jika dispatcher crash sebelum publish, PENDING tetap dapat di-retry.
Jika publish berhasil tetapi process gagal sebelum update PUBLISHED, duplicate publish diperbolehkan; consumer harus state-idempotent.

RabbitMQ contract:

```text
durable quorum queue
persistent messages
publisher confirms
manual consumer acknowledgements
message_id = outbox_event.id
```

Consumer melakukan ACK hanya setelah finalization transaction commit. Kegagalan connection/process sebelum ACK dapat menyebabkan redelivery.

Technical AI retry configuration:

```text
AI_TECHNICAL_MAX_RETRIES=<non-negative integer>
```

`total attempts = 1 initial + configured retries` within the same analysis version.

Setelah retry habis:

```text
analysis FAILED
→ AI_ANALYSIS_FAILED / TECHNICAL_RETRY_EXHAUSTED
→ ESCALATION_REQUIRED
```

Verifier FAIL adalah hasil semantic, bukan technical retry:

```text
analysis FAILED
→ AI_ANALYSIS_FAILED / VERIFIER_FAIL
→ ESCALATION_REQUIRED
```

Business re-analysis tetap dibatasi secara terpisah oleh `MAX_REANALYSIS`.

---

# 32. Penanganan Kegagalan AI

Penyebab terminal escalation untuk case-analysis sengaja dibatasi menjadi:

```text
VERIFIER_FAIL
TECHNICAL_RETRY_EXHAUSTED
REANALYSIS_LIMIT_REACHED
```

Kegagalan teknis seperti Vertex unavailable, timeout, atau retryable invalid structured output lebih dulu mengonsumsi `AI_TECHNICAL_MAX_RETRIES`. Hanya setelah budget habis status menjadi `TECHNICAL_RETRY_EXHAUSTED`.

`POLICY_CONFLICT` is analysis/verifier information, not a direct workflow transition. It only escalates when verification resolves to `FAIL`.

Kegagalan embedding/policy-indexing termasuk lifecycle aktivasi policy dan tidak langsung mengubah state case.

Case tidak dihapus atau di-reset.

`ESCALATION_REQUIRED` is a stopped state in MVP:

```text
allowed:
- view
- history
- analysis history
- add evidence
- close

not available:
- resume
- direct/generic re-analysis trigger
- approve/reject/sign/execute
- participant mutation
- core case mutation
```

---

# 33. Arsitektur Observability

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

# 34. Korelasi Request

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

# 35. Arsitektur Security

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

# 36. Permission Service

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

# 37. Keamanan Data

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

# 38. Batas Prompt Injection

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

# 39. Arsitektur Network

Logical network flow:

```text
Browser
  ↓ HTTPS
Cloud Run Web
  ↓ HTTPS
Cloud Run API
  ├── Cloud SQL
  └── Cloud Storage signed-upload control

Cloud Run Worker Pool
  ├── Cloud SQL / transactional outbox
  ├── RabbitMQ durable queue
  ├── Cloud Storage evidence read
  └── Vertex AI

Cloud Run API
  └── Vertex AI embedding for policy indexing
```

Public client tidak terhubung langsung ke database.

---

# 40. Arsitektur CI/CD

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

# 41. Arsitektur Environment

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

# 42. Arsitektur Konfigurasi

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
AI_TECHNICAL_MAX_RETRIES
POLICY_RETRIEVAL_TOP_K
POLICY_INDEX_LEASE_SECONDS
AI_WORKER_LEASE_SECONDS
RABBITMQ_URL
RABBITMQ_AI_QUEUE
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

# 43. Sequence — Create dan Submit Case

```text
Maker
  │
  ▼
Frontend
  │ POST /cases
  ▼
Backend transaction
  ├── Insert Case(owner = creator)
  ├── Insert immutable Maker participant
  └── Audit CASE_CREATED
  │
  ▼
COMMIT

Maker
  │
  ▼
Frontend
  │ POST /cases/{id}/submit
  ▼
Backend transaction
  ├── Lock Case
  ├── Validate Participant Cardinality + strict SoD
  ├── Freeze Case Core + Participant Context
  ├── DRAFT → SUBMITTED → AI_ANALYSIS
  ├── Allocate Analysis v1 = GENERATING
  ├── Audit CASE_SUBMITTED + AI_ANALYSIS_STARTED
  ├── Insert PENDING outbox AI_ANALYSIS_REQUESTED
  └── COMMIT
  │
  ▼
HTTP response = AI_ANALYSIS
```

Tidak ada RabbitMQ/Vertex call di dalam request transaction.

---

# 44. Sequence — Analisis AI

```text
sentinel-worker Outbox Dispatcher
  │
  ├── Read PENDING outbox
  ├── Publish persistent RabbitMQ message
  ├── Wait publisher confirm
  └── Mark PUBLISHED
  │
  ▼
RabbitMQ quorum queue
  │
  ▼
AI Consumer
  │
  ├── Load + lock exact GENERATING analysis
  ├── Claim worker_attempt_id
  └── Commit claim
  │
  ▼
Context Builder
  ├── Case
  ├── Evidence text / supported GCS fileData
  ├── Reviewer Feedback
  └── Execution Feedback
  │
  ▼
Policy Retriever → ACTIVE + READY policy chunks
  │
  ▼
Vertex AI Gemini → Validator → Verifier
  │
  ▼
Finalization Transaction
  ├── Validate current worker_attempt_id
  ├── Persist COMPLETED/FAILED result + provenance
  ├── Update current_analysis_id only on success
  ├── Transition CHECKING or ESCALATION_REQUIRED
  └── Audit
  │
  ▼
COMMIT → RabbitMQ ACK
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
  ├── Audit CHECKER_REJECTED
  ├── Transition → AI_ANALYSIS
  ├── Check MAX_REANALYSIS
  ├── If allowed: allocate GENERATING + outbox
  ├── If exhausted: ESCALATION_REQUIRED
  └── Commit
  │
  ▼
RabbitMQ worker path only when queued
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
  ├── Audit EXECUTION_BLOCKED
  ├── Transition → AI_ANALYSIS
  ├── Check MAX_REANALYSIS
  ├── If allowed: allocate GENERATING + outbox
  ├── If exhausted: ESCALATION_REQUIRED
  └── Commit
  │
  ▼
RabbitMQ worker path only when queued
```

---

# 48. Batas Skalabilitas

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

# 49. Batas Model Provider

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

# 50. Invariant Arsitektur

Invariant berikut harus selalu benar:

```text
1. Frontend never directly changes workflow state.
2. AI never directly changes workflow state.
3. Backend is the workflow authority.
4. Every decision is tied to an analysis version.
5. Historical analysis is immutable.
6. Historical policy reference is immutable.
7. Only ACTIVE + READY applicable policies are authoritative.
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
20. Analysis result NULL means not produced; valid empty collections are preserved as empty JSON.
21. Invalid/unvalidated model output is never persisted as structured analysis data.
22. Case core data and participant set are immutable after submission.
23. In-flight participant replacement is not supported; close + new case is required.
24. Maker is the case creator and its assignment is immutable.
25. Manual/generic re-analysis is not exposed in MVP; re-analysis requires a governed business trigger.
26. Evidence actor/source is validated server-side; client cannot self-assert SYSTEM evidence.
27. User evidence writes never transition workflow state directly.
28. current_analysis_id points only to the latest reviewable COMPLETED analysis; FAILED attempts do not replace it.
29. AI terminal failure causes are VERIFIER_FAIL or TECHNICAL_RETRY_EXHAUSTED; quota exhaustion is REANALYSIS_LIMIT_REACHED.
30. ESCALATION_REQUIRED has no resume path in MVP.
31. System role USER/ADMIN is separate from case workflow roles; ADMIN alone never grants case-action authority.
32. Maker, Checker, Signer, and Executer must be distinct active users for a case.
33. Case owner is the immutable Maker/creator in MVP.
34. AI job intent is persisted through transactional outbox in the same transaction as AI_ANALYSIS/GENERATING state.
35. RabbitMQ delivery is at-least-once; worker_attempt_id/worker_started_at claim semantics prevent stale consumers from finalizing over a newer claim.
36. Close is rejected while an analysis is GENERATING or an execution is IN_PROGRESS.
37. Policy activation requires target READY and currently effective; future/expired versions are not activatable.
38. Stale policy indexing attempts cannot finalize after a newer index_attempt_id is claimed.
39. Supported PDF/JPEG/PNG evidence is passed directly from GCS to Gemini.
40. technical_retry_count survives worker restart/redelivery and is not reset by transport recovery.
41. At least one ACTIVE ADMIN must remain; the last ACTIVE ADMIN cannot be demoted or inactivated.
42. An ACTIVE participant on a non-terminal case cannot be made INACTIVE; finish or safely close the affected case first.
43. Docs define the contract; FE and BE implement it.
```

---

# 51. Dokumen Terkait

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

Dokumen ini menjadi sumber kebenaran untuk system architecture JAWIR Sentinel MVP.
