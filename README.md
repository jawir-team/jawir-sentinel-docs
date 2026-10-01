# JAWIR Sentinel

**Specification Version:** 2.0  
**Status:** MVP Baseline  
**Track:** BFSI — Banking, Financial Services, and Insurance  
**Primary Direction:** Risk Assessment  
**Secondary Direction:** Compliance Automation  

---

## 1. Project Overview

JAWIR Sentinel adalah **AI-assisted governed decision workflow** untuk membantu tim operasional finansial menangani operational exception, incident, dan compliance-sensitive case.

Sentinel menggunakan AI untuk memahami case, mengambil SOP yang relevan, menilai risk dan compliance, serta menghasilkan recommendation. Keputusan tetap dilakukan oleh manusia melalui workflow terkontrol.

Prinsip sistem:

> **AI generates intelligence. Backend enforces control. Humans hold authority. Data preserves accountability.**

---

## 2. Problem

Operational case pada institusi finansial dapat melibatkan banyak informasi, SOP, evidence, unit, serta tahapan approval. Proses manual dapat menjadi lambat, tidak konsisten, sulit diaudit, dan bergantung pada knowledge individu.

JAWIR Sentinel menyediakan:

- structured case analysis;
- SOP/policy grounding;
- operational and compliance risk assessment;
- evidence-backed recommendation;
- multi-role review and authorization;
- iterative re-analysis;
- execution feedback loop;
- complete audit trail.

---

## 3. MVP Scope

### Included

- Unit Management
- User Management
- Case Type Management
- Case Management
- Case Participant Assignment
- SOP / Policy Management
- Evidence Management
- AI Case Analysis
- Policy Retrieval
- Risk & Compliance Assessment
- Recommendation and Alternatives
- Missing Information Detection
- AI Verification
- Analysis Versioning
- Multi-Checker Review
- Signer Authorization
- Execution Tracking
- Re-analysis Loop
- DONE / CLOSED lifecycle
- Audit History
- Web Application
- Google Cloud Deployment

### Out of Scope

- core banking integration;
- real customer or transaction data;
- fraud prediction;
- real-time transaction monitoring;
- autonomous financial execution;
- autonomous approval;
- dynamic workflow builder;
- BPMN engine;
- enterprise organization hierarchy;
- enterprise SSO;
- quorum or delegation engine;
- microservice architecture.

---

## 4. Workflow Roles

Workflow role ditentukan **per case**.

| Role | Responsibility |
|---|---|
| Maker | Membuat dan submit case beserta context dan evidence |
| Checker | Memvalidasi AI analysis, SOP grounding, risk, evidence, dan recommendation |
| Signer | Memberikan authorization terhadap action plan |
| Executer | Menjalankan action plan yang sudah diotorisasi |

Satu case dapat memiliki lebih dari satu Checker.

### Segregation of Duties

```text
Maker ≠ Checker
Maker ≠ Signer
Checker ≠ Signer
Checker ≠ Executer
Signer ≠ Executer
Maker = Executer allowed
```

Semua required Checker harus `APPROVE` sebelum case dapat masuk ke Signer.

---

## 5. Main Workflow

```text
Maker
  ↓
Submit Case
  ↓
AI Analysis
  ↓
Checker(s)
  ↓
Signer
  ↓
Executer
  ↓
DONE
```

### Checker / Signer Reject

```text
Checker / Signer
      ↓ REJECT
Feedback + Evidence
      ↓
AI Re-analysis
      ↓
New Analysis Version
      ↓
Checker(s)
      ↓
Signer
```

### Execution Failure

```text
Executer
   ↓
BLOCKED / FAILED
   ↓
Blocker + Evidence
   ↓
AI Re-analysis
   ↓
Checker(s)
   ↓
Signer
   ↓
Executer
```

---

## 6. Case State Machine

Case status:

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

### State Transition

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

State transition hanya dilakukan oleh backend berdasarkan business rule.

---

## 7. AI Analysis

### Analysis Pipeline

```text
Case
  ↓
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
Structured Analysis
```

### Context

AI menerima context dari:

- current case;
- current evidence;
- active SOP/policy;
- latest reviewer feedback;
- previous execution result;
- current workflow state.

AI harus membedakan:

```text
FACT
ASSUMPTION
UNKNOWN
POLICY
REVIEWER_FEEDBACK
AI_INFERENCE
```

### Policy Status

```text
POLICY_FOUND
POLICY_PARTIAL
NO_POLICY_FOUND
INSUFFICIENT_EVIDENCE
POLICY_CONFLICT
```

### Policy-Based Recommendation

Jika applicable policy ditemukan:

- recommendation harus grounded pada policy;
- policy code dan version dicatat;
- section dicatat;
- supporting evidence dapat ditelusuri.

### Non-Policy Recommendation

Jika tidak ada applicable policy:

```text
NON_POLICY_RECOMMENDATION
```

Output wajib memuat:

- potential benefits;
- potential risks;
- assumptions;
- alternatives;
- missing information;
- uncertainty.

### Verification

Verifier memeriksa:

- unsupported claims;
- hallucinated evidence;
- policy contradiction;
- evidence mismatch;
- missing critical information;
- conflicting policies;
- recommendation-policy mismatch.

Verification result:

```text
PASS
PASS_WITH_WARNING
FAIL
```

Analysis dengan status `FAIL` tidak dapat masuk ke Checker.

---

## 8. Analysis Versioning

Setiap re-analysis menghasilkan version baru.

```text
Analysis v1
Analysis v2
Analysis v3
```

Analysis lama tidak dihapus atau ditimpa.

Setiap analysis version menyimpan:

- case snapshot;
- policy references and versions;
- evidence references;
- AI model;
- prompt version;
- analysis output;
- verification result;
- created timestamp.

Setiap approval terikat pada `analysis_id`.

Ketika analysis version baru dibuat, approval pada analysis sebelumnya tidak berlaku pada version baru.

---

## 9. SOP / Policy Management

### Policy Lifecycle

```text
DRAFT
ACTIVE
SUPERSEDED
```

Hanya policy dengan status `ACTIVE` yang digunakan sebagai authoritative source oleh AI.

### Policy Data

Policy menyimpan:

- code;
- title;
- domain;
- case type;
- version;
- content;
- status;
- created by;
- approved by;
- effective from;
- effective until.

Satu policy hanya memiliki satu active version pada satu waktu.

Saat version baru diaktifkan:

- active version lama menjadi `SUPERSEDED`;
- version baru menjadi `ACTIVE`.

---

## 10. Unit and User Management

Struktur organisasi MVP menggunakan unit sederhana.

Contoh:

```text
Operations
Risk
Application Development
Management
```

Setiap user terhubung ke satu unit.

Untuk MVP, satu unit dapat direpresentasikan oleh satu user.

Workflow role tidak disimpan sebagai global user role. Role Maker, Checker, Signer, dan Executer ditentukan melalui assignment pada setiap case.

---

## 11. System Architecture

```text
┌────────────────────────────┐
│      Next.js Web App       │
│        Cloud Run           │
└──────────────┬─────────────┘
               │ HTTPS
               ▼
┌────────────────────────────┐
│       Go Backend API       │
│         Cloud Run          │
│                            │
│  - Case Service            │
│  - Workflow Engine         │
│  - Policy Service          │
│  - Evidence Service        │
│  - Review Service          │
│  - Execution Service       │
│  - Audit Service           │
│  - AI Orchestrator         │
└──────┬─────────┬───────────┘
       │         │
       │         └────────────────────┐
       ▼                              ▼
┌──────────────────┐         ┌──────────────────────┐
│ Cloud SQL        │         │ Vertex AI Gemini    │
│ PostgreSQL       │         │ Analysis + Verifier │
│ + pgvector       │         └──────────────────────┘
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Cloud Storage    │
│ Evidence / Files │
└──────────────────┘
```

Authentication menggunakan Firebase Authentication.

---

## 12. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js + TypeScript |
| Backend | Golang |
| HTTP Router | Chi |
| Database | PostgreSQL |
| Database Service | Google Cloud SQL |
| SQL Access | sqlc |
| Vector Search | pgvector |
| AI Model | Gemini via Vertex AI |
| File Storage | Google Cloud Storage |
| Authentication | Firebase Authentication |
| Runtime | Google Cloud Run |
| CI/CD | GitHub Actions + Docker |

MVP menggunakan satu backend service dan satu frontend service.

---

## 13. GitHub Organization and Repository Structure

Seluruh repository berada di dalam satu GitHub Organization:

```text
JAWIR
├── jawir-sentinel-be
├── jawir-sentinel-fe
└── jawir-sentinel-docs
```

### 13.1 `jawir-sentinel-be`

Repository backend dan core business logic.

```text
jawir-sentinel-be/
├── cmd/
│   └── api/
├── internal/
│   ├── unit/
│   ├── user/
│   ├── casetype/
│   ├── case/
│   ├── workflow/
│   ├── policy/
│   ├── evidence/
│   ├── analysis/
│   ├── review/
│   ├── execution/
│   ├── audit/
│   └── ai/
├── db/
│   ├── migrations/
│   └── queries/
├── ai/
│   ├── prompts/
│   ├── schemas/
│   └── evaluation/
├── testdata/
│   ├── cases/
│   └── policies/
├── Dockerfile
├── sqlc.yaml
├── go.mod
├── go.sum
└── README.md
```

Backend repository memiliki ownership terhadap:

- database schema and migrations;
- business rules;
- workflow state machine;
- AI orchestration;
- policy retrieval;
- SOP ingestion;
- analysis versioning;
- review and signing logic;
- execution logic;
- audit trail;
- backend API;
- Google Cloud backend deployment.

Workflow transition dikelola pada:

```text
internal/workflow/
├── states.go
├── events.go
├── guards.go
└── transition.go
```

### 13.2 `jawir-sentinel-fe`

Repository frontend web application.

```text
jawir-sentinel-fe/
├── src/
│   ├── app/
│   ├── components/
│   ├── features/
│   ├── services/
│   ├── hooks/
│   └── types/
├── public/
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md
```

Frontend repository memiliki ownership terhadap:

- dashboard;
- case list;
- case creation;
- case detail;
- AI analysis presentation;
- SOP and evidence presentation;
- Checker review UI;
- Signer authorization UI;
- execution UI;
- audit/history timeline;
- Unit Management UI;
- User Management UI;
- Case Type Management UI;
- SOP Management UI;
- frontend authentication flow;
- API client integration;
- frontend Cloud Run deployment.

### 13.3 `jawir-sentinel-docs`

Repository dokumentasi dan **single source of truth** untuk keputusan hasil grooming.

```text
jawir-sentinel-docs/
├── README.md
├── product/
│   └── product-spec.md
├── architecture/
│   ├── system-architecture.md
│   ├── backend-architecture.md
│   └── ai-architecture.md
├── workflow/
│   ├── business-flow.md
│   └── state-machine.md
├── database/
│   └── erd.md
├── api/
│   └── api-contract.md
└── diagrams/
```

`jawir-sentinel-docs` menjadi authoritative reference untuk:

- product scope;
- workflow;
- business rules;
- system architecture;
- AI architecture;
- ERD;
- state machine;
- API contract.

Perubahan contract dilakukan pada repository docs terlebih dahulu dan kemudian diimplementasikan pada FE dan BE.

---

## 14. FE–BE Contract

Boundary antar repository:

```text
jawir-sentinel-docs
        ↓
  API Contract
        ↓
BE ←────────────→ FE
```

`jawir-sentinel-docs/api/api-contract.md` menjadi contract antara FE dan BE.

Frontend dapat bekerja menggunakan mock response sesuai API contract tanpa menunggu implementasi backend selesai.

Backend wajib memenuhi request/response contract yang telah didefinisikan pada docs.

---

## 15. GitHub Project

Project management menggunakan GitHub Project pada level Organization.

Board:

```text
Backlog
Ready
In Progress
Review
Done
```

Satu board dapat berisi issue dari:

```text
jawir-sentinel-be
jawir-sentinel-fe
jawir-sentinel-docs
```

Issue tetap dibuat pada repository yang memiliki ownership terhadap pekerjaan tersebut.

---

## 16. Core Data Model

Core tables:

```text
units
users

case_types
cases
case_participants

policies
policy_versions
policy_chunks

case_evidences

ai_analyses
analysis_policy_refs
analysis_evidence_refs

decisions
executions

audit_events
```

### units

```text
id
code
name
description
created_at
updated_at
```

### users

```text
id
unit_id
name
email
status
is_admin
created_at
updated_at
```

Status:

```text
ACTIVE
INACTIVE
```

### case_types

```text
id
code
name
description
created_at
updated_at
```

### cases

```text
id
case_number
case_type_id
title
description
urgency
status
created_by
owner_id
current_analysis_id
closed_by
close_reason
closed_at
created_at
updated_at
```

Urgency:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

### case_participants

```text
id
case_id
user_id
role
required
status
assigned_by
assigned_at
unassigned_at
```

Role:

```text
MAKER
CHECKER
SIGNER
EXECUTER
```

Constraint:

```text
UNIQUE(case_id, user_id, role)
```

### policies

```text
id
code
title
domain
case_type_id
description
created_at
updated_at
```

### policy_versions

```text
id
policy_id
version
status
content
file_path
effective_from
effective_until
created_by
approved_by
created_at
approved_at
```

Constraint:

```text
UNIQUE(policy_id, version)
```

### policy_chunks

```text
id
policy_version_id
section
content
embedding
```

`embedding` menggunakan `pgvector`.

### case_evidences

```text
id
case_id
source_type
source_user_id
evidence_type
title
content
file_path
created_at
```

Source type:

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

### ai_analyses

```text
id
case_id
version
status

summary
facts
assumptions
unknowns

risk_analysis
compliance_analysis
recommendation
alternatives
missing_information

policy_status
evidence_quality
uncertainty

verification_status
verification_notes

model_name
prompt_version
created_at
```

Structured AI fields menggunakan PostgreSQL `JSONB`.

Constraint:

```text
UNIQUE(case_id, version)
```

### analysis_policy_refs

```text
id
analysis_id
policy_version_id
section
excerpt
relevance_score
```

### analysis_evidence_refs

```text
id
analysis_id
evidence_id
usage_type
```

### decisions

```text
id
case_id
analysis_id
actor_id
actor_role
decision
reason
comment
created_at
```

Actor role:

```text
CHECKER
SIGNER
```

Decision:

```text
APPROVE
REJECT
```

Constraint:

```text
UNIQUE(analysis_id, actor_id, actor_role)
```

### executions

```text
id
case_id
analysis_id
executer_id
status
action_taken
result
blocker
started_at
completed_at
created_at
```

Status:

```text
IN_PROGRESS
SUCCESS
BLOCKED
FAILED
```

### audit_events

```text
id
case_id
event_type
actor_id
actor_role
analysis_id
metadata
created_at
```

Audit event bersifat append-only pada application layer.

---

## 17. Policy Retrieval

SOP active version dipecah menjadi chunk dan disimpan pada `policy_chunks`.

Retrieval flow:

```text
Case Context
  ↓
Metadata Filter
  ↓
ACTIVE Policy Filter
  ↓
Case Type / Domain Filter
  ↓
Vector Similarity Search
  ↓
Relevant Policy Chunks
```

Relevant chunk diberikan ke Gemini sebagai policy context.

---

## 18. API Contract

Base path:

```http
/api/v1
```

### Current User

```http
GET /me
```

### Units

```http
GET  /units
POST /units
```

### Users

```http
GET   /users
POST  /users
PATCH /users/{user_id}
```

### Case Types

```http
GET  /case-types
POST /case-types
```

### Cases

```http
POST  /cases
GET   /cases
GET   /cases/{case_id}
PATCH /cases/{case_id}

POST /cases/{case_id}/submit
POST /cases/{case_id}/close
```

### Participants

```http
POST   /cases/{case_id}/participants
DELETE /cases/{case_id}/participants/{participant_id}
```

### Evidence

```http
GET  /cases/{case_id}/evidences
POST /cases/{case_id}/evidences

POST /cases/{case_id}/evidences/upload-url
POST /cases/{case_id}/evidences/file
```

### Analysis

```http
GET  /cases/{case_id}/analyses
GET  /cases/{case_id}/analyses/current
GET  /cases/{case_id}/analyses/{analysis_id}
POST /cases/{case_id}/reanalyze
```

### Checker

```http
POST /cases/{case_id}/checker-decisions
GET  /cases/{case_id}/checker-status
```

### Signer

```http
POST /cases/{case_id}/signer-decision
```

### Execution

```http
POST /cases/{case_id}/executions
POST /cases/{case_id}/executions/{execution_id}/result
```

### Policies

```http
GET  /policies
POST /policies

POST /policies/{policy_id}/versions
GET  /policies/{policy_id}/versions/{version_id}
POST /policies/{policy_id}/versions/{version_id}/activate
```

### Audit

```http
GET /cases/{case_id}/history
```

### Dashboard

```http
GET /dashboard/summary
```

State berubah hanya melalui business action. Tidak tersedia endpoint generic `change-status`.

---

## 19. API Response

Success:

```json
{
  "data": {}
}
```

Error:

```json
{
  "error": {
    "code": "INVALID_STATE_TRANSITION",
    "message": "Case must be in CHECKING state",
    "details": {}
  }
}
```

Pagination:

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

Stale analysis:

```http
409 Conflict
```

```json
{
  "error": {
    "code": "STALE_ANALYSIS",
    "message": "Analysis version is no longer current",
    "details": {}
  }
}
```

---

## 20. Transaction Rules

Critical workflow mutation dijalankan dalam database transaction.

Contoh Checker reject:

```text
BEGIN

Insert Decision
Insert Review Feedback Evidence
Insert Audit Event
Update Case Status → AI_ANALYSIS

COMMIT
```

AI call dijalankan setelah database transaction selesai.

---

## 21. Audit Rules

Critical events:

```text
CASE_CREATED
CASE_SUBMITTED

AI_ANALYSIS_STARTED
AI_ANALYSIS_COMPLETED
AI_ANALYSIS_FAILED

CHECKER_APPROVED
CHECKER_REJECTED

SIGNER_APPROVED
SIGNER_REJECTED

EXECUTION_STARTED
EXECUTION_BLOCKED
EXECUTION_FAILED
EXECUTION_SUCCESS

CASE_DONE
CASE_CLOSED

POLICY_CREATED
POLICY_ACTIVATED
POLICY_SUPERSEDED

EVIDENCE_ADDED
```

Audit event tidak dapat di-update atau di-delete melalui application API.

---

## 22. Authentication and Authorization

Authentication menggunakan Firebase Authentication.

Backend menerima Firebase ID Token:

```http
Authorization: Bearer <firebase-id-token>
```

Backend memvalidasi token dan memetakan identity ke `users`.

Authorization dilakukan server-side berdasarkan:

- authenticated user;
- case participant assignment;
- current case state;
- segregation-of-duties rule.

---

## 23. File Storage

Attachment disimpan di Google Cloud Storage.

Upload flow:

```text
Frontend
  ↓
Request Signed Upload URL
  ↓
Cloud Storage
  ↓
Register Evidence Metadata
```

---

## 24. AI Runtime

MVP menggunakan:

```text
Gemini via Vertex AI
```

AI orchestration berada di backend.

AI tidak dapat:

- mengubah workflow state;
- approve case;
- sign case;
- execute action;
- mengaktifkan SOP.

Re-analysis limit:

```text
max_reanalysis = 3
```

Jika limit tercapai:

```text
ESCALATION_REQUIRED
```

---

## 25. Deployment

Frontend:

```text
jawir-sentinel-fe
  ↓ GitHub Actions
Docker Build
  ↓
sentinel-web
  ↓
Cloud Run
```

Backend:

```text
jawir-sentinel-be
  ↓ GitHub Actions
Docker Build
  ↓
sentinel-api
  ↓
Cloud Run
```

Infrastructure:

```text
PostgreSQL → Cloud SQL
Files      → Cloud Storage
AI         → Vertex AI
Auth       → Firebase Authentication
```

Environment:

```text
DEV
PROD
```

---

## 26. MVP Acceptance Flow

MVP dinyatakan berhasil ketika workflow berikut dapat dijalankan dari web application:

```text
Maker creates case
↓
Maker submits case
↓
AI Analysis v1 generated
↓
Checker rejects with feedback
↓
AI Analysis v2 generated
↓
All Checkers approve
↓
Signer approves
↓
Executer marks execution BLOCKED
↓
AI Analysis v3 generated
↓
All Checkers approve
↓
Signer approves
↓
Executer marks SUCCESS
↓
Case becomes DONE
```

Seluruh event harus terlihat pada history dan setiap analysis version harus tetap dapat dibuka.

---

## 27. Team

**Team:** JAWIR  
**Project:** JAWIR Sentinel  
**Tagline:** *Know the risk before you make the call.*
