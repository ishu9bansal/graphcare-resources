# 1) One-line project goal

Build a clinician-centric Patient Summary & Diagnosis system that collects structured and unstructured patient data into an extensible canonical profile, produces clinician-reviewable summaries, and maintains a continuously evolving patient state consumed by a hybrid Diagnosis Module (KG + fine-tuned reasoning model) to generate ranked diagnoses, confidence scores, and dynamic interview guidance — all designed as an assistive “shadow interviewer”.

# 2) Design intent & constraints

* High-level directional design (no implementation specifics).
* Separation of concerns: parsing/ingestion, canonical profile, summarizer, parser (speech→structured facts), diagnosis (hybrid), UI and governance are distinct.
* Latency: **near-real-time** (5–20s acceptable); system must stream and catch up when needed without blocking clinician workflow.
* Deployment choices (cloud vs on-prem) are left for later — HLD should not hard-lock these.

# 3) Top-level modules (what we need) — *purpose + high level responsibilities*

1. **Integration / Source Adapters**

   * Purpose: connect to structured sources (EHRs, device APIs, health apps), ingest structured records.
   * Responsibility: source-specific adapters → normalized payloads + provenance.

2. **Unstructured Ingestion & Parsing**

   * Purpose: convert free text, documents, PDFs, images, and reports into structured facts.
   * Responsibility: OCR, document parsing, image-report insights, NER, relation extraction, temporal reasoning; output: data points + confidence + provenance.

3. **Speech Pipeline & Real-time Parser**

   * Purpose: capture clinician–patient conversation and convert to structured facts with temporal context.
   * Responsibility: diarization, ASR, speaker labels, timestamps → parser that extracts assertions (symptoms, durations, temporal relations, responses) and submits `Add/Update DataPoint` events.

4. **Canonical Patient Profile (Profile Store)**

   * Purpose: authoritative patient resource (single source of truth for downstream modules).
   * Responsibility: store canonical data points, per-field metadata (source, confidence, timestamp, verifier), versioned states/history.

5. **Summarisation Module**

   * Purpose: convert canonical profile → concise clinician-facing summary (fact-only, no inference).
   * Responsibility: creating short, prioritized summaries for pre-visit review.

6. **Question-Elicitor (Gap Finder)**

   * Purpose: identify missing or low-confidence fields relevant to diagnosis and propose interview questions.
   * Responsibility: compare profile to diagnostic requirements/guidelines and produce prioritized missing-data list and suggested phrasing.

7. **Diagnosis Module (Hybrid KG + Reasoner)**

   * Purpose: consume profile state and produce ranked diagnoses, confidence scores, evidence traces, and updated question list.
   * Responsibility: hybrid reasoning (KG for explainability + fine-tuned model for ranking/soft reasoning); accept structured inputs only (separation of concern).

8. **Clinician UI / Interaction Layer**

   * Purpose: present profile, summary, live diagnosis panel, question list, edit/verify controls.
   * Responsibility: accept clinician edits/verification; show confidence trends; allow accept/reject of suggestions; send updates back to profile store.

9. **Audit, Governance & Curation Tools**

   * Purpose: curator workflows for KG, evidence management, QA, and audit trails for contributions/edits.
   * Responsibility: KG versioning, evidence attribution, editorial approval flows.

10. **Analytics & Feedback Loop**

    * Purpose: monitor usage and model performance, collect clinician feedback for model improvement.
    * Responsibility: metrics (acceptance rates, latency), dataset collection for retraining, drift detection.

11. **Security & Access Control**

    * Purpose: auth, RBAC, logging and provenance (policy & compliance readiness).
    * Responsibility: identity, consent, audit logs — enforced across modules.

# 4) Data model — starter canonical schema (recommended initial fields)

A finite, extensible list to start with. Each field should carry metadata: `{value, source, confidence, timestamp, verifier, provenanceRef}`.

Core groups:

* Patient identifiers: `{patientId, name, dob, sex, MRN, contact}`
* Encounter context: `{encounterId, encounterType, encounterDatetime, clinicianId}`
* Presenting complaint & symptom set: `{chiefComplaint, symptoms: [{name, onsetDate, duration, severity, progression, modifiers, temporalRelation}]}`
* Vitals & observations: `{BP, HR, RR, Temp, SpO2, BMI, weight, height}`
* Medications: `[{drug, dose, route, frequency, startDate, endDate, adherence}]`
* Allergies: `[{substance, reaction, severity}]`
* Past Medical History (PMHx): `[{condition, onsetDate, status}]`
* Surgical history: `[{procedure, date, outcome}]`
* Family history: `[{relation, condition}]`
* Social history: `{smokingStatus, alcoholUse, occupation, livingSituation}`
* Immunizations: `[{vaccine, date}]`
* Labs & imaging: `[{testName, value, unit, refRange, date, reportSummary}]`
* Devices / wearable data: `[{type, metric, timeframe}]`
* Functional status / PROs: `{scoreType: value, date}`
* Exam findings: `[{bodySystem, finding, laterality, certainty}]`
* Problem list / active diagnoses: `[{condition, certainty, onsetDate}]`
* Notes & attachments: `[{type, title, date, sourceRef}]`
* Metadata: provenance + field level confidence + lastVerifiedBy.

This list is intentionally conservative and extensible.

# 5) High-level data & control flows (sequence)

1. Pre-visit ingest: Integration Adapters → Profile Store → Summariser → Clinician preview.
2. Interview (live):

   * Audio → Speech Pipeline → Real-time Parser → `Add/Update DataPoint` events → Profile Store.
   * On each relevant update: Question-Elicitor & Diagnosis Module re-run (incremental), UI updated (near-real-time).
3. Post-interview: Clinician edits and verifies → Finalized profile + diagnosis suggestions → Audit log + optional EHR write-back.
4. Continuous: Analytics collects acceptance/edits for retraining; KG curation updates propagate to Diagnosis Module.

# 6) Interfaces (module boundaries — high level)

* `AddDataPoint(patientId, dataPoint, metadata)` — canonical write API used by parsers, integrations, UI.
* `GetProfile(patientId, options)` — read API for summariser, diagnosis module, UI.
* `GetSummary(patientId)` — summariser output.
* `GetMissingDataPoints(patientId)` — question-elictor.
* `EvaluateState(patientId, profileSnapshot)` → `[{diagnosis, confidence, evidenceRef}]` — diagnosis module.
* `AcknowledgeAction(actionId, clinicianId, decision)` — for clinician acceptance/feedback.
  All APIs must preserve provenance and versioning.

# 7) Diagnostic strategy choice

* **Hybrid (KG + fine-tuned reasoning model)** — recommended for this stage: KG provides explainability and evidence paths; the fine-tuned model handles soft ranking, pattern recognition, and probabilistic scoring over noisy inputs. Keep KG authoritative for traceability; let model output be ranked suggestions with evidence references from KG when possible.

# 8) Key design decisions & tradeoffs (for leadership to consider)

* **Evolving schema vs upfront completeness:** Start with a minimal canonical schema (above) and keep schema migrations simple and versioned.
* **Real-time catch-up behavior:** Use an append-only event stream for data points so the diagnosis module can re-evaluate incrementally and process backlogs if it falls behind. This avoids blocking the clinician.
* **Explainability vs accuracy tension:** KG helps explainability; a black-box model may be higher-performing but harder to justify in clinic. Hybrid reduces regulatory/ adoption risk.
* **On-prem requirements later:** If some hospitals demand on-prem, design module boundaries and data export/import patterns to make isolated on-prem deployments feasible later.

# 9) Risks & suggested mitigations (top items)

1. **Incorrect extraction from unstructured sources**

   * Mitigation: per-field confidence, provenance UI, quick edit flow, human-in-the-loop validation, capture corrections for model retraining.
2. **Temporal reasoning errors**

   * Mitigation: parser must output normalized timestamps and event relations; provide explicit UI to correct timeline items; test with annotated corpora.
3. **Model drift & poor suggestions**

   * Mitigation: analytics to detect drop in acceptance; continuous retraining cadence and canary evaluations; capture clinician feedback as labeled data.
4. **Clinician workflow friction (latency / too many suggestions)**

   * Mitigation: near-real-time is okay; allow asynchronous updates; surface only top-N suggestions and high-value question prompts.
5. **Regulatory / provenance challenges**

   * Mitigation: full audit trail, evidence links to KG, restrict automated actions (e.g., no auto-ordering), clinician sign-off required.
6. **Scope creep (feature explosion)**

   * Mitigation: scope around minimum viable canonical fields and core flows for the first milestone.

# 10) Suggested milestones & scope cuts (to structure roadmap)

Phase 0 — Discovery & Foundations (2–6 weeks)

* Lock initial canonical schema.
* Define event model for `Add/Update DataPoint`.
* Prototype minimal profile store and summariser (UI mockups).
* Collect sample data (documents, transcripts) for parser training.

Phase 1 — Ingestion + Profile + Summariser (8–12 weeks)

* Build Integration adapters for one structured source (or synthetic sample).
* Implement Unstructured Ingestion PoC (document parsing + OCR).
* Implement Profile store + Summariser + simple clinician review UI.
* Manual pipeline to feed parsed facts into profile (no realtime audio yet).

Phase 2 — Speech Pipeline + Real-time Parser (8–12 weeks)

* ASR + diarization + parser (temporal reasoning).
* Wire speech parser → Profile store (streaming).
* UI: live profile updates and edit flow.

Phase 3 — Diagnosis Module (hybrid) + Question-Elicitor (10–16 weeks)

* Implement KG skeleton + evidence model.
* Integrate fine-tuned reasoning model for ranking.
* Show ranked diagnoses + confidence + evidence trace in UI.
* Basic analytics capture.

Phase 4 — Governance, Curation & Feedback Loop (ongoing)

* KG curation UI, version control.
* Retraining workflows, analytics dashboards, and clinician feedback ingestion.

Phase 5 — Hardening & Integrations (depends on customers)

* EHR read/write, security & compliance, scalability, localization, on-prem options.

Cut options if you must reduce scope initially:

* Skip live ASR — use recorded interviews for Phase 2.
* Keep diagnosis module as model-only (defer KG) initially and add KG later for explainability.
* Start with text/pdf ingestion only (defer image/chart parsing).

# 11) Suggested team structure (small cross-functional pods)

Core team for MVP:

* Product lead / Clinician liaison (1) — domain validation, acceptance criteria.
* Tech lead / System architect (1) — HLD ownership.
* Backend engineers (2) — profile store, APIs, ingestion adapters.
* ML/NLP engineers (2) — document parser + speech parser + temporal reasoning.
* ML Research / Model engineer (1) — diagnosis hybrid model + KG lead.
* Frontend / UX (1) — clinician UI prototypes & flows.
* QA & Data engineer (1) — test data, analytics.
* DevOps & Security (1) — deployment, access control, logging.

As project grows add: KG curators (clinical SMEs), more frontend/backend, regulatory/compliance expert.

# 12) Open questions to lock later (for implementation planning)

* Finalize first target integration (which EHR / device).
* Data retention & residency policies per customer.
* Acceptance criteria / KPI for diagnosis module (precision@k, clinician acceptance rate).
* Auditing & logging granularity (legal/forensic needs).
* Decision on initial model sourcing (open models vs licensed proprietary).
