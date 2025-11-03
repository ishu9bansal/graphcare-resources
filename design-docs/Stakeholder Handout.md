# Visual Addendum — Stakeholder Handout

## 1. Project Timeline & Milestones (Visual)

```mermaid
gantt
title Project Timeline — Patient Summary & Diagnosis System
dateFormat  YYYY-MM-DD
section Discovery & Planning
Phase 0; Define Schema & Data Flow      :a1, 2026-01-01, 30d
section Core Foundation
Phase 1; Ingestion + Profile + Summariser :a2, after a1, 60d
Phase 2; Speech Pipeline + Parser          :a3, after a2, 60d
section Intelligence Layer
Phase 3; Diagnosis Module + Question-Elicitor :a4, after a3, 90d
section Governance & Feedback
Phase 4; Governance + Analytics Loop       :a5, after a4, 60d
```

**Notes:**

* Each phase delivers an independent, testable system increment.
* Governance & feedback work continues in parallel after Phase 3.

---

## 2. Team Ownership Map

```mermaid
graph TD

A[Product & Clinical Liaison] -->|Defines Use Cases| B[System Architect]
B -->|Module Boundaries & Data Contracts| C[Backend Team]
B -->|ML Interface Specs| D[ML / NLP Team]
B -->|UX Flows & Integration| E[Frontend Team]

C -->|APIs & Profile Store| F[Canonical Patient Profile]
C -->|Data Pipelines| G[Integration & Ingestion]

D -->|Document + Speech Parsing| H[Parser Modules]
D -->|Diagnosis Model| I[Diagnosis Module]
D -->|Question Generation| J[Question Elicitor]

E -->|Clinician Experience| K[Clinician UI]
E -->|Data Visualizations| L[Analytics Dashboards]

M[Governance & Security] -->|Compliance, Audit, Access| N[Security Layer]
M -->|Knowledge Graph Curation| O[KG Curation Tools]

subgraph Teams
C
D
E
M
end

subgraph Modules
F
G
H
I
J
K
L
N
O
end
```

**Ownership summary:**

* **Backend team:** Profile store, ingestion pipelines, API layer.
* **ML/NLP team:** Parsing, diagnosis reasoning, question elicitation.
* **Frontend team:** Clinician dashboard, feedback tools, analytics.
* **Governance team:** KG curation, security, compliance.
* **Product & Clinical liaison:** requirements validation, clinician feedback, prioritization.

---

## 3. Module Complexity & Risk Heatmap (Indicative)

| Module               | Complexity | Risk Level | Key Risk                          |
| -------------------- | ---------- | ---------- | --------------------------------- |
| Profile Store        | Medium     | Low        | Schema evolution & consistency    |
| Unstructured Parser  | High       | High       | NLP accuracy & temporal reasoning |
| Speech Parser        | High       | Medium     | Latency & noise handling          |
| Summariser           | Low        | Low        | Quality of summarization          |
| Question Elicitor    | Medium     | Medium     | Diagnostic relevance of questions |
| Diagnosis Module     | High       | High       | Explainability & validation       |
| KG Curation          | Medium     | Medium     | Evidence sourcing & versioning    |
| UI Layer             | Medium     | Low        | Clinician usability               |
| Analytics & Feedback | Low        | Low        | Data governance                   |

---

## 4. Visual Summary of System Layers

```mermaid
graph LR
A[Input Sources] --> B[Ingestion & Parsing Layer]
B --> C[Canonical Patient Profile]
C --> D[Summarisation Layer]
C --> E[Question-Elicitor]
C --> F[Diagnosis Engine]
E --> G[Clinician UI]
F --> G
G -->|Feedback| C
C --> H[Analytics & Governance]
```

**Layer descriptions:**

* **Ingestion & Parsing Layer:** Responsible for converting heterogeneous input into structured facts.
* **Profile Layer:** Maintains authoritative patient data.
* **Reasoning Layer:** Includes summarisation, question elicitor, and diagnosis engine.
* **Presentation Layer:** Clinician interface with live updates and controls.
* **Governance Layer:** Ensures auditability, compliance, and feedback loops.

---

**Purpose of this Visual Addendum:** To complement the main HLD document with an executive-level overview of project phases, ownership, risks, and dependencies — supporting leadership alignment and planning discussions.
