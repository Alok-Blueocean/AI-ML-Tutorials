# Module 01 — Example Projects

Module 01 has no tools to build with yet — its deliverable is a correct mental model and a set of planning artifacts you will reuse for the rest of the course. Accordingly, these three projects are planning/reasoning projects rather than code projects. Each still scales the way the rest of this course's projects do: Mini is a single self-contained artifact, Medium is a small multi-part deliverable you could show a manager, and Production is the full end-to-end version you could hand to a real engineering organization and have them act on it.

---

## Mini Project — Personal Readiness Dossier

**Scope:** Produce a single one-to-two-page personal document combining:
1. Your completed Skills Self-Assessment Checklist (Section 1 of `tutorial.md`) with an honest score and a remediation plan for any gaps (mirrors Exercise 1).
2. Your from-memory explanations of the eight core terms from the Key Terminology table, self-graded (mirrors Exercise 2).
3. Your personalized week-by-week pacing plan for all 26 modules (mirrors Exercise 7), adjusted for your actual background and available hours.

**What it demonstrates:** That you can honestly self-assess, that you've internalized the core vocabulary well enough to explain it unaided, and that you've converted a generic syllabus into a realistic personal commitment. This is the same "start small, single artifact" pattern as tracking one Iris experiment in the MLflow module — it's a warm-up, not a stress test, but skipping it means starting Module 02 without a real plan.

---

## Medium Project — Multi-System MLOps/LLMOps Audit

**Scope:** Choose 4–6 real systems you have access to (your own side projects, systems at your current job, or well-documented open-source projects if you have neither) that are meaningfully varied in nature — include at least one plain-software system, one classical-ML system, and one LLM-backed system. For each system, produce:

1. A completed walk of the `architecture.md` Section 3 decision tree, with your answer to every branch point and your reasoning.
2. A final classification (DevOps only / + MLOps / + LLMOps) with a concrete list of practices that should apply, referencing the specific module(s) that teach each practice (e.g., "needs a model registry — Module 03/13").
3. For each system, one paragraph on the single biggest gap between what the system *currently* does and what this module says it *should* do — i.e., a mini gap-analysis, not just a classification.
4. A one-page summary memo, written as if for an engineering manager, prioritizing which system's gap is most urgent to close and why (blast radius / iteration frequency, per Section 5/6 of the tutorial).

**What it demonstrates:** That you can apply the decision framework to real, messy systems rather than clean textbook examples, that you can distinguish "needs full rigor" from "would be over-engineering," and that you can communicate the result the way a working engineer actually needs to — as a prioritized, actionable memo, not just a classification table. This is the equivalent step-up from the Mini project that "track an XGBoost pipeline" is to "track an Iris experiment" in the MLflow module: same underlying skill, applied to something with real edges and real trade-offs.

---

## Production-Grade Project — Organizational MLOps/LLMOps Platform Charter

**Scope:** Write a full platform charter document for a (real or realistically hypothetical) organization, structured to mirror the Section 1 reference architecture in `architecture.md`. This is the planning-and-governance equivalent of "full experiment tracking server + model registry + CI/CD + deployment" from later, tool-heavy modules — here the deliverable is the architecture and rollout plan a platform team would actually write before touching infrastructure. It must include:

1. **Current-state map.** For each of the 8 bands in the Section 1 reference architecture (source of truth, CI pipeline, registry layer, CD/orchestration, serving layer, application layer, gateway, observability/feedback loop), document what the organization currently has (even if "nothing") and what specific gap exists.
2. **Target-state architecture diagram.** Your own ASCII or drawn version of the full reference architecture, annotated with the specific tools/practices this organization would adopt for each band (you may reference tools taught later in the course — MLflow, Kubernetes/KServe, vLLM/Triton, a specific vector DB, OpenTelemetry, Langfuse/Phoenix — as forward-looking placeholders, since the org doesn't need to have chosen them yet).
3. **Phased rollout plan** that deliberately mirrors this course's own build-order argument (Section 3.2): release engineering foundation first, then application-layer concerns, then evaluation, then observability, then flagship architecture (RAG/agents if applicable), then infra-at-scale, then governance/cost — but explicitly flag which cross-cutting practices (secrets management, structured logging, a minimal eval set) must NOT be deferred to their "teaching-order" phase, per the tutorial's explicit warning that build order and teaching order diverge for security/cost.
4. **Governance-gap remediation section.** A concrete plan for how every production request will be traceable — which prompt version, which retrieved documents/index snapshot, and which model version produced it — from day one, referencing the specific span-tagging pattern from `architecture.md` Section 2.
5. **Ownership sketch.** A lightweight RACI (or equivalent) naming who owns each band day-to-day, and where the natural handoff points and failure points are likely to be (e.g., "who gets paged when the drift-monitoring loop fires?").
6. **Risk register.** At least five concrete production failure scenarios (extend your Exercise 8 pre-mortem) mapped to the specific architecture band and module whose absence would most plausibly cause them, with a mitigation already scheduled in the rollout plan.

**What it demonstrates:** The ability to think and communicate at the level a senior MLOps/LLMOps engineer or platform lead actually operates at — translating the abstract reference architecture and roadmap rationale from this module into a document a real organization could execute against, complete with sequencing, ownership, and governance built in from the start rather than bolted on after an incident. Every later module in this course effectively fills in one section of the charter you wrote here — by Module 26 (the capstone), you should be able to point at your own running system and check off every band this charter described.
