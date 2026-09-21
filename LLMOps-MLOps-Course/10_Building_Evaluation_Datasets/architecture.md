# Module 10 — Architecture Deep Dive: Building Evaluation Datasets

This file complements `tutorial.md` with a larger end-to-end system diagram, a full sequence diagram of building and freezing one evaluation dataset version, and a decision tree for choosing between the tools and approaches covered in this module. Read `tutorial.md` first — this file assumes that context.

---

## 1. End-to-End System Architecture

This is the full picture: how production logs become a stratified, gold-labeled, bias-audited, versioned evaluation dataset that then feeds every downstream evaluation and promotion gate in this course.

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    PRODUCTION SYSTEM (live traffic)                                  │
│                                                                                                       │
│   users ──► LLM app / RAG pipeline / agent ──► response                                             │
│                          │                                                                            │
│                          ▼                                                                            │
│              ┌───────────────────────┐                                                               │
│              │  Structured logging      │  every request: prompt, response, retrieved context,        │
│              │  (OTel spans / DB table) │  retry_count, judge_score, latency, user feedback           │
│              └───────────┬───────────┘                                                               │
│                          ▼                                                                            │
│              ┌───────────────────────┐                                                               │
│              │  production_reviews      │  human/automated review adds: issue_type, resolved,          │
│              │  table (source of truth) │  label, updated_at                                          │
│              └───────────┬───────────┘                                                               │
└──────────────────────────┼────────────────────────────────────────────────────────────────────────┘
                            │  gold_set SQL query (Section 3.1): resolved=TRUE, issue_type NOT NULL,
                            │  ORDER BY updated_at DESC LIMIT 500
                            ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              OFFLINE DATASET-BUILDING PIPELINE                                       │
│                                                                                                       │
│  ┌─────────────────┐    ┌────────────────────┐    ┌──────────────────────────┐                     │
│  │ Category           │    │ Stratified proportional│    │ Edge-case miner (sampler.py)│                     │
│  │ distribution        │───►│ sample: 160 normal      │◄───│ failure_driven / escalation/ │                     │
│  │ computation         │    │ (80%) matched to        │    │ statistical / adversarial /  │                     │
│  │ (billing 26%, ...)  │    │ category proportions    │    │ synthetic (40 = 20%)         │                     │
│  └─────────────────┘    └───────────┬────────────┘    └──────────────┬───────────┘                     │
│                                      └───────────────┬────────────────┘                                  │
│                                                       ▼                                                   │
│                                       ┌──────────────────────────────┐                                   │
│                                       │  Candidate gold set (n=200)     │                                   │
│                                       └──────────────┬───────────────┘                                   │
│                                                       ▼                                                   │
│                          ┌────────────────────────────────────────────────┐                             │
│                          │            LABELING STAGE                       │                             │
│                          │  ┌────────────────┐   ┌────────────────┐        │                             │
│                          │  │ Expert Labeler A│   │ Expert Labeler B│        │  independent, blind         │
│                          │  └────────┬───────┘   └────────┬───────┘        │  to each other's labels      │
│                          │           └────────┬───────────┘                │                             │
│                          │                    ▼                            │                             │
│                          │       Cohen's kappa computation                 │                             │
│                          │       (overall + per-category)                  │                             │
│                          └──────────────────┬─────────────────────────────┘                             │
│                                    κ < 0.7    │    κ >= 0.7                                                │
│                              ┌────────────────┴────────────┐                                              │
│                              ▼                             ▼                                              │
│              ┌───────────────────────────┐   ┌───────────────────────────────┐                          │
│              │ Reconcile disagreements     │   │      5-POINT BIAS AUDIT           │                          │
│              │ (3rd senior reviewer tie-   │   │  1. category balance >= 5%        │                          │
│              │  break); re-label batch     │   │  2. length distribution spread    │                          │
│              └──────────────┬────────────┘   │  3. kappa >= 0.7 (re-check)       │                          │
│                              │                │  4. label difficulty < 80% "correct"│                          │
│                              └───────────────►│  5. temporal spread >= 4 weeks     │                          │
│                                               └────────────────┬───────────────┘                          │
│                                                        pass │      │ fail (any check)                      │
│                                                              ▼      ▼                                       │
│                                              freeze + version   flag findings, fix or                       │
│                                              (below)            document, re-run audit                      │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                              │
                                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                             VERSIONING & DISTRIBUTION LAYER (DVC-style)                               │
│                                                                                                       │
│   gold_set_v3.2.0.jsonl  ──dvc add──►  content-addressed remote (S3/GCS)                             │
│   gold_set_v3.2.0.jsonl.dvc  ──git commit + git tag eval-v3.2.0──►  Git history (small, auditable)   │
│   CHANGELOG.md  ──documents what changed and why──►  Git history                                     │
│                                                                                                       │
│   Schema adapters materialize BOTH consumer formats from the one frozen file:                        │
│         gold_set_v3.2.0.jsonl ──► to_deepeval_golden() ──► DeepEval EvaluationDataset                │
│                                ──► to_ragas_row()       ──► Ragas testset (Dataset)                  │
└───────────────────────────────────────────┬───────────────────────────────────────────────────────┘
                                              │  pinned by tag, never "latest"
                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                        CONSUMERS (this is where Module 03's promotion gate lives)                    │
│                                                                                                       │
│   CI eval job (GitHub Actions) ──checkout eval-v3.2.0──► run_eval.py ──► promotion gate ──► registry │
│   Ad hoc notebook experiments  ──checkout eval-v3.2.0──► same frozen data, reproducible comparisons  │
│   Quarterly refresh job        ──creates eval-v3.3.0──► new tag, changelog entry, NOT a silent edit  │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Sequence Diagram — Building and Freezing One Dataset Version

This traces a single concrete run of the pipeline end to end: an engineer kicking off a quarterly refresh of the evaluation dataset.

```
Engineer      Log Puller     Sampler        Labeler A    Labeler B    Kappa Calc   Bias Auditor   DVC/Git      CI Gate
   │               │              │              │             │            │             │            │            │
   │ trigger        │              │              │             │            │             │            │            │
   │ quarterly       │              │              │             │            │             │            │            │
   │ refresh job     │              │              │             │            │             │            │            │
   ├──────────────►│              │              │             │            │             │            │            │
   │               │ run gold_set  │              │             │            │             │            │            │
   │               │ SQL query     │              │             │            │             │            │            │
   │               │ (resolved,    │              │             │            │             │            │            │
   │               │  reviewed,    │              │             │            │             │            │            │
   │               │  last 30d)    │              │             │            │             │            │            │
   │               │──────────────►│              │             │            │             │            │            │
   │               │  500-row pool │              │             │            │             │            │            │
   │               │◄──────────────┤              │             │            │             │            │            │
   │               │               │ compute       │             │            │             │            │            │
   │               │               │ category dist │             │            │             │            │            │
   │               │               │ + stratified  │             │            │             │            │            │
   │               │               │ 80/20 sample  │             │            │             │            │            │
   │               │               │ (160+40=200)  │             │            │             │            │            │
   │               │               │──────────────────────────────────────────────────────────────────────────────►  │
   │               │               │  candidate gold_set.jsonl dispatched for labeling                                │
   │               │               │              │             │            │             │            │            │
   │               │               │              │ label        │            │             │            │            │
   │               │               │              │ (independent,│            │             │            │            │
   │               │               │              │  blind)      │            │             │            │            │
   │               │               │              ├────────────►│            │             │            │            │
   │               │               │              │              │ label       │             │            │            │
   │               │               │              │              │ (independent,│           │            │            │
   │               │               │              │              │  blind)      │           │            │            │
   │               │               │              │              ├────────────►│            │            │            │
   │               │               │              │  labels_A                  │  labels_B    │            │            │
   │               │               │              ├─────────────────────────────────────────►│            │            │
   │               │               │                                                          │ compute κ   │            │
   │               │               │                                                          │ overall +   │            │
   │               │               │                                                          │ per-category│            │
   │               │               │                                                          │◄────────────┤            │
   │               │               │                                                          │             │            │
   │               │               │                          κ < 0.7 for a subset ──────────►│ reconcile   │            │
   │               │               │                          (loop back to Labeler A/B for   │ (3rd reviewer│           │
   │               │               │                           disputed examples)              │ tie-break)  │            │
   │               │               │                                                          │             │            │
   │               │               │                          κ >= 0.7 overall ───────────────►│ pass to     │            │
   │               │               │                                                          │ bias auditor│            │
   │               │               │                                                          ├────────────►│            │
   │               │               │                                                                        │ run 5-point │
   │               │               │                                                                        │ checklist   │
   │               │               │                                                                        │◄────────────┤
   │               │               │                                                                        │ PASS        │
   │               │               │                                                                        ├────────────►│
   │               │               │                                                                                     │ dvc add,
   │               │               │                                                                                     │ git commit,
   │               │               │                                                                                     │ git tag
   │               │               │                                                                                     │ eval-v3.3.0,
   │               │               │                                                                                     │ dvc push
   │               │               │                                                                                     │◄─────┤
   │ notified:      │              │              │             │            │             │            │            │
   │ eval-v3.3.0     │              │              │             │            │             │            │            │
   │ frozen, PR open │              │              │             │            │             │            │            │
   │◄─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
   │               │              │              │             │            │             │            │  next PR merge
   │               │              │              │             │            │             │            │  triggers CI
   │               │              │              │             │            │             │            │  eval job that
   │               │              │              │             │            │             │            │  checks out
   │               │              │              │             │            │             │            │  eval-v3.3.0
   │               │              │              │             │            │             │            │──────────────►
```

**Key properties this sequence guarantees:**
- Labeling is **blind** — Labeler B never sees Labeler A's labels before submitting their own, which is what makes the kappa computation meaningful rather than anchored/contaminated agreement.
- The **bias audit runs after kappa passes, before freeze** — never after the dataset is already tagged and in use.
- The freeze step (`dvc add` + `git tag`) is the **only** point at which the dataset becomes citable by version; everything before it is a mutable working draft.
- A **new tag** (`eval-v3.3.0`), never a mutation of `eval-v3.2.0`, records the refresh — preserving every historical comparison's validity.

---

## 3. Decision Tree — Choosing Your Approach and Tooling

```
START: I need to build/maintain an evaluation dataset for an LLM/ML system.
│
├─ Q1: Does the system have live production traffic yet?
│   │
│   ├─ NO (pre-launch / cold start)
│   │   └─► Use synthetic generation to bootstrap a v0 set.
│   │       ├─ RAG-style system (has a document corpus / knowledge base)?
│   │       │   └─► Use Ragas testset generation (evolutionary/Evol-Instruct-style
│   │       │       question synthesis from your documents).
│   │       └─ General LLM app / agent (no natural document corpus)?
│   │           └─► Use DeepEval's Golden Synthesizer, seeded with hand-authored
│   │               examples of expected inputs/outputs.
│   │       └─► MANDATORY: schedule replacement with a production-sourced set
│   │           within weeks of launch. Do not let v0 become permanent.
│   │
│   └─ YES (live traffic exists)
│       └─► Proceed to Q2.
│
├─ Q2: How much traffic volume / how many categories?
│   │
│   ├─ Low volume, 1-2 categories, low stakes (internal tool, no compliance exposure)
│   │   └─► Lightweight path: single reviewer + documented rationale is acceptable
│   │       short-term; still stratify by category; still version with DVC.
│   │       Scale up rigor (2nd labeler, kappa) as stakes/usage grow.
│   │
│   └─ Meaningful volume, multiple categories, and/or user-facing or regulated
│       │
│       └─► Full pipeline: production-log SQL pull → stratified 80/20 sample →
│           two independent expert labelers → Cohen's kappa >= 0.7 → 5-point
│           bias audit → freeze + version. (This module's default path.)
│
├─ Q3: Which eval framework(s) will consume this dataset?
│   │
│   ├─ RAG-pipeline-specific metrics needed (faithfulness, context precision/recall)?
│   │   └─► Design schema to satisfy Ragas conventions (user_input /
│   │       retrieved_contexts / reference) — and add DeepEval-compatible fields
│   │       too if any general-purpose assertions are also needed.
│   │
│   ├─ General LLM app / agent / tool-use assertions, pytest-style CI needed?
│   │   └─► Design schema to satisfy DeepEval's Golden schema (input /
│   │       expected_output / context / expected_tools) — add Ragas fields too
│   │       if retrieval metrics may be needed later.
│   │
│   └─ Both, or unsure which you'll need long-term?
│       └─► Build the canonical SUPERSET schema (Section 3.2 of tutorial.md) once;
│           write thin adapters to both formats. This is the recommended default
│           for any team past the prototype stage.
│
├─ Q4: How should labeling be executed?
│   │
│   ├─ Small team, domain experts are internal engineers/PMs
│   │   └─► In-house labeling via a spreadsheet or lightweight internal tool is
│   │       acceptable IF blind independence + kappa computation are still enforced.
│   │
│   ├─ Need scale, audit trail, or multiple concurrent labelers
│   │   └─► Use a dedicated labeling platform (e.g. Label Studio) — gives you
│   │       task queues, blind assignment, and built-in agreement tooling.
│   │
│   └─ Need managed dataset + annotation-queue-from-traces workflow with less
│      custom pipeline code
│       └─► Consider a managed platform (e.g. LangSmith datasets + annotation
│           queues) — trades some control/cost for less infrastructure to own.
│
├─ Q5: How should the dataset be versioned?
│   │
│   ├─ Small-to-mid team, not already on a lakehouse/feature-store platform
│   │   └─► DVC (add → commit → tag → checkout). Default choice as of mid-2026.
│   │
│   ├─ Already standardized on a lakehouse (Delta Lake / Iceberg) with native
│   │   time-travel/versioning
│   │   └─► Use the lakehouse table's native versioning; keep the same discipline
│   │       (immutable tagged versions, changelog, pinned CI checkout) regardless
│   │       of storage substrate.
│   │
│   └─ Using a managed eval platform (LangSmith / Confident AI) that already
│      versions datasets server-side
│       └─► Use its native dataset versioning, but still mirror the version
│           identifier into your release manifest (Module 03 pattern) so CI gates
│           can pin to it explicitly rather than an implicit "latest."
│
└─ Q6: Do I need adversarial/red-team coverage?
    │
    ├─ System has any user-facing surface or handles any sensitive data
    │   └─► YES, mandatory. Use a maintained payload library (built into DeepEval,
    │       Promptfoo, or Giskard) rather than hand-writing from scratch, and
    │       supplement with 5-10 hand-crafted, domain-specific attempts.
    │
    └─ Fully internal, no user input surface, no sensitive data path
        └─► Lower priority, but still include a minimal adversarial slice — internal
            tools are not immune to misuse, and this category is cheap to include.
```

---

## 4. How This Connects to Neighboring Modules

```
 Module 03                    Module 10 (THIS MODULE)                 Later evaluation modules
 (Versioning/Registries)      (Building Evaluation Datasets)          (LLM-as-judge, CI eval gates)
 ┌─────────────────┐          ┌──────────────────────────┐            ┌───────────────────────┐
 │ Deployment triple │◄────────│ Supplies the THIRD leg:    │           │ Consume the frozen,     │
 │ = model_version +  │  dataset│ dataset_version, produced  │──────────►│ versioned dataset as    │
 │   prompt_version +  │ _version│ by the pipeline in this    │  frozen  │ the fixed input to      │
 │   dataset_version   │        │ module, with the same       │  version │ every scoring run,       │
 └─────────────────┘          │ freeze/tag discipline as     │           │ promotion gate, and      │
                               │ models and prompts.          │           │ regression comparison.   │
                               └──────────────────────────┘            └───────────────────────┘
```
