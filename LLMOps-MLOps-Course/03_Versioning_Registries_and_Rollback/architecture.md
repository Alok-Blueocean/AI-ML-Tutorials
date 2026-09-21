# Module 03 — Architecture Deep Dive: Versioning, Registries, and Rollback

This file complements `tutorial.md` with larger system diagrams, a full sequence diagram of an end-to-end release, and a decision tree for choosing between the tools and patterns covered in this module. Read `tutorial.md` first — this file assumes that context.

---

## 1. End-to-End System Architecture

This is the full picture: how versioning, the registry, promotion gates, serving, and rollback fit together as one system, from training run to production traffic to incident recovery.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    OFFLINE / CI SIDE                                          │
│                                                                                                │
│  ┌────────────┐    ┌─────────────┐    ┌───────────────┐    ┌────────────────────────────┐   │
│  │  Data /     │    │  Prompt      │    │  Training      │    │  Evaluation Harness         │   │
│  │  Feature     │──► │  Design /    │──► │  Job           │──► │  - offline metrics           │   │
│  │  Store       │    │  Templates   │    │  (Airflow /    │    │  - LLM-as-judge eval sets    │   │
│  │  (DVC /      │    │              │    │   K8s Job)     │    │  - safety / red-team suite    │   │
│  │  Lakehouse)  │    │              │    │                │    │  - latency benchmark          │   │
│  └─────┬──────┘    └──────┬──────┘    └───────┬───────┘    └───────────────┬────────────┘   │
│        │  version:5.0.0    │  version:1.3.2     │ MLflow run: run_id=abc123   │                    │
│        │                   │                    │                            │                    │
│        └───────────────────┴────────────────────┴────────────────────────────┘                    │
│                                             │                                                       │
│                                             ▼                                                       │
│                              ┌───────────────────────────────┐                                     │
│                              │       REGISTRY LAYER           │                                     │
│                              │  ┌─────────────────────────┐  │                                     │
│                              │  │ MLflow Model Registry    │  │  version=N, tags={semver, dataset_ver,│
│                              │  │  registered_model:       │  │  run_id, eval_scores}                │
│                              │  │  fraud-detector          │  │                                     │
│                              │  └─────────────────────────┘  │                                     │
│                              │  ┌─────────────────────────┐  │                                     │
│                              │  │ MLflow 3 GenAI Prompt    │  │  version=M, commit_message,          │
│                              │  │  Registry: reviewer-     │  │  diff-vs-previous                    │
│                              │  │  prompt                  │  │                                     │
│                              │  └─────────────────────────┘  │                                     │
│                              │  ┌─────────────────────────┐  │                                     │
│                              │  │ Dataset version registry │  │  version, schema hash, row count      │
│                              │  │  (DVC tag / lakehouse    │  │                                     │
│                              │  │  table version)          │  │                                     │
│                              │  └─────────────────────────┘  │                                     │
│                              └───────────────┬───────────────┘                                     │
│                                              │  alias: @challenger                                  │
│                                              ▼                                                       │
│                              ┌───────────────────────────────┐                                     │
│                              │     PROMOTION GATE (CI job)    │                                     │
│                              │  absolute_accuracy >= 0.88 ?    │                                     │
│                              │  relative_delta >= -1% ?         │                                     │
│                              │  p95_latency < 2000ms ?          │                                     │
│                              │  safety_eval_score >= threshold? │                                     │
│                              │  compatibility rules satisfied?  │  (release.yaml)                   │
│                              └───────────────┬───────────────┘                                     │
│                                     pass │       │ fail                                             │
│                                          ▼       ▼                                                    │
│                              alias → @champion   remain @challenger, notify owner                    │
└──────────────────────────────────────────┬───────────────────────────────────────────────────────┘
                                            │  release.yaml committed + tagged in git
                                            ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    ONLINE / SERVING SIDE                                       │
│                                                                                                │
│   ┌────────────────────────────────────────────────────────────────────────────────────┐      │
│   │                          TRAFFIC ROUTER / CANARY CONTROLLER                          │      │
│   │                                                                                        │      │
│   │   95% ──────────────────────────► @champion (model 2.0.1 + prompt 1.4.2 + ds 5.0.0)   │      │
│   │    5% ──────────────────────────► @challenger (model 2.1.0 + prompt 1.4.5 + ds 5.1.0) │      │
│   │                                                                                        │      │
│   └───────────────────────────┬────────────────────────────────────────────────────────┘      │
│                                │  every inference                                                │
│                                ▼                                                                  │
│                  ┌──────────────────────────────┐                                                │
│                  │  Inference Logging             │  logs deployment TRIPLE per request:          │
│                  │  (structured logs / feature     │  {model_version, prompt_version,              │
│                  │   store / OTel spans)           │   dataset_version, input_hash, output, latency}│
│                  └───────────────┬──────────────┘                                                │
│                                  ▼                                                                │
│                  ┌──────────────────────────────┐        ┌────────────────────────────┐          │
│                  │  Monitoring / Observability     │───►  │  Alerting (Prometheus /      │          │
│                  │  - business metrics (accuracy,   │      │  Grafana / PagerDuty)         │          │
│                  │    hallucination rate)            │      │  business-metric alerts,      │          │
│                  │  - infra metrics (latency, errors)│      │  NOT just infra health        │          │
│                  └───────────────┬──────────────┘      └───────────────┬────────────┘          │
│                                  │  threshold breach                                    │          │
│                                  ▼                                                       ▼          │
│                  ┌──────────────────────────────────────────────────────────────┐              │
│                  │                      LINEAGE STORE                            │              │
│                  │  release_id, model_version, prompt_version, dataset_version,   │              │
│                  │  status (GOOD/BAD), timestamp — queryable history              │              │
│                  └───────────────────────────┬──────────────────────────────────┘              │
│                                               │  query: last GOOD release before current           │
│                                               ▼                                                     │
│                  ┌──────────────────────────────────────────────────────────────┐              │
│                  │                    ROLLBACK ENGINE                            │              │
│                  │  1. resolve last-known-good tuple                             │              │
│                  │  2. repoint model alias   3. repoint prompt alias             │              │
│                  │  4. repoint dataset pointer  5. write audit event             │              │
│                  └───────────────────────────┬──────────────────────────────────┘              │
│                                               │  atomic alias repoint (no redeploy)                │
│                                               ▼                                                     │
│                              back to TRAFFIC ROUTER — 100% restored to LKG tuple                    │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Key architectural properties to notice:**

- The registry layer sits between offline experimentation and the promotion gate — it is not a passive artifact store, it is the control point.
- Serving never talks to a hardcoded version; it resolves `@champion` / `@challenger` aliases, which is exactly what makes both promotion *and* rollback an atomic, no-redeploy operation.
- The lineage store is written to on every promotion and every rollback — it is the audit ledger the rollback engine queries, not a side effect.
- Monitoring feeds both the promotion gate (pre-production) and the rollback trigger (post-production) from the *same* metrics pipeline — this is deliberate, per the "Common Mistakes" section of `tutorial.md`.

---

## 2. Sequence Diagram: Full Release Lifecycle (Train → Promote → Canary → Full Rollout → Incident → Rollback)

```
Data/ML       Training      Evaluation     MLflow          CI Promotion    Traffic        Monitoring     Rollback
Engineer      Job           Harness        Registry        Gate            Router         /Alerting      Engine
   │              │              │              │                │              │              │              │
   ├─ trigger ───►│              │              │                │              │              │              │
   │              ├─ train ──────┤              │                │              │              │              │
   │              ├─ log run, params, metrics ──►│                │              │              │              │
   │              │              │              │                │              │              │              │
   │              ├─ request eval ──────────────►│                │              │              │              │
   │              │              ├─ compute accuracy, latency, safety score      │              │              │
   │              │              │              │                │              │              │              │
   │              │              ├─ write metrics to run ───────►│                │              │              │
   │              │              │              │                │              │              │              │
   │              ├─ register model version N ──►│                │              │              │              │
   │              │              │              ├─ tag semver, dataset_version,   │              │              │
   │              │              │              │   run_id                        │              │              │
   │              │              │              ├─ alias @challenger → N          │              │              │
   │              │              │              │                │              │              │              │
   │              │              │              │◄── evaluate promotion criteria ┤              │              │
   │              │              │              │    (accuracy>=0.88, delta>=-1%, │              │              │
   │              │              │              │     p95<2000ms, safety gate)    │              │              │
   │              │              │              │                │              │              │              │
   │              │              │              │   ── PASS ────►│                │              │              │
   │              │              │              │◄─ alias @champion → N ─────────┤                │              │
   │              │              │              │                │                │              │              │
   │              │              │              │                ├─ commit release.yaml (git) ───┤              │
   │              │              │              │                │                │              │              │
   │              │              │              │                ├─ start canary: 5% → @challenger│              │
   │              │              │              │                │                ├─ route 5% ───►│              │
   │              │              │              │                │                │                ├─ observe   │
   │              │              │              │                │                │                │  metrics   │
   │              │              │              │                │                │                │  healthy   │
   │              │              │              │                ├─ ramp: 25% → 100% ─────────────►│              │
   │              │              │              │                │                │  full rollout   │              │
   │              │              │              │                │                │                │              │
   │              │              │              │                │                │  ... time passes, new prompt v1.4.5│
   │              │              │              │                │                │  and dataset v5.1.0 promoted similarly│
   │              │              │              │                │                │                │              │
   │              │              │              │                │                │  02:47 accuracy 0.89→0.81 ────►│
   │              │              │              │                │                │                │◄─ ALERT FIRES│
   │              │              │              │                │                │                │              │
   │              │              │              │                │                │                ├─ page on-call│
   │              │              │              │                │                │                │              │
   │              │              │              │                │                │                ├─ query current tuple ─►│
   │              │              │              │                │                │                │◄── r118 (bad)──────────┤
   │              │              │              │                │                │                ├─ query LKG before r118►│
   │              │              │              │                │                │                │◄── r117 (good)─────────┤
   │              │              │              │                │                │                ├─ authorize rollback ──►│
   │              │              │              │                │                │                │              ├─ repoint model alias
   │              │              │              │                │                │                │              ├─ repoint prompt alias
   │              │              │              │                │                │                │              ├─ repoint dataset ptr
   │              │              │              │                │                │◄── traffic restored to r117 ──┤
   │              │              │              │                │                │                │◄─ write audit event
   │              │              │              │                │                │  accuracy recovers to 0.89     │              │
```

**Reading this diagram**: notice there are exactly two places a *human* is structurally required to act: (1) authorizing the rollback (a deliberate control point — speed with control, not full automation), and (2) the original decision to merge/ship the training job in the first place. Everything between "registered" and "served," and everything between "alert fires" and "traffic restored," is mechanical and code-driven. That is the entire point of this module: **the dangerous, error-prone step (manual eyeballing of "is this good enough") has been engineered out of both promotion and rollback.**

---

## 3. Decision Tree: Choosing a Registry and a Rollback Pattern

### 3a. Which registry should we use?

```
                         ┌─────────────────────────────────────┐
                         │  Do you need to version PROMPTS      │
                         │  (LLM/GenAI system) as well as        │
                         │  model weights?                       │
                         └───────────────────┬───────────────────┘
                                 YES ◄────────┴────────► NO
                                  │                        │
                                  ▼                        ▼
                ┌───────────────────────────┐   ┌───────────────────────────────┐
                │ MLflow 3 (native Model     │   │  Are you already deeply        │
                │ Registry + GenAI Prompt    │   │  invested in W&B or Comet for  │
                │ Registry in one platform)  │   │  experiment tracking?           │
                │ is the current strongest   │   └───────────────┬───────────────┘
                │ default — closes the       │              YES  │  NO
                │ model+prompt gap that      │        ┌──────────┘  └───────────┐
                │ previously needed          │        ▼                          ▼
                │ PromptLayer/LangSmith      │  ┌─────────────┐          ┌─────────────────┐
                │ bolted on.                 │  │ Use that     │          │ Default to MLflow │
                └───────────────┬────────────┘  │ vendor's     │          │ (open-source,      │
                                 │               │ registry     │          │ widest adoption,    │
                                 │               │ (W&B / Comet)│          │ no vendor lock-in)  │
                                 │               │ — reduces    │          └─────────────────┘
                                 │               │ integration  │
                                 │               │ friction     │
                                 │               └─────────────┘
                                 ▼
             ┌───────────────────────────────────────────┐
             │  Does your scale / compliance need exceed   │
             │  what any off-the-shelf tool provides?       │
             │  (e.g. thousands of concurrently served      │
             │  models, bespoke internal approval flows,    │
             │  deep coupling to a proprietary deploy        │
             │  platform — Uber Michelangelo-class scale)    │
             └───────────────────┬───────────────────────┘
                        YES ◄─────┴─────► NO
                         │                  │
                         ▼                  ▼
             ┌───────────────────┐   ┌─────────────────────────────┐
             │ Build a custom      │   │  MLflow (self-hosted or       │
             │ registry — but      │   │  Databricks-managed with       │
             │ budget real          │   │  Unity Catalog for lakehouse-  │
             │ engineering time     │   │  native governance) is the     │
             │ for lineage,         │   │  right default choice.          │
             │ audit, and access    │   └─────────────────────────────┘
             │ control — these are  │
             │ easy to get subtly   │
             │ wrong.                │
             └───────────────────┘
```

### 3b. What rollback pattern should we use for this release?

```
                     ┌───────────────────────────────────────────┐
                     │  Did this release change MORE THAN ONE      │
                     │  leg of the deployment triple (model,        │
                     │  prompt, dataset) at once?                    │
                     └───────────────────┬───────────────────────┘
                            YES ◄──────────┴──────────► NO (single leg changed)
                             │                              │
                             ▼                              ▼
             ┌───────────────────────────┐      ┌───────────────────────────────┐
             │ MANDATORY: full-tuple       │      │  Is there still meaningful      │
             │ rollback. Do not attempt    │      │  risk of interaction effects     │
             │ to isolate which leg        │      │  with unrelated concurrent       │
             │ caused the regression        │      │  changes (e.g. a feature flag,   │
             │ during the incident itself   │      │  infra change, upstream data      │
             │ — restore the whole LKG      │      │  pipeline change)?                │
             │ tuple first, root-cause      │      └───────────────┬───────────────┘
             │ afterward in the postmortem. │                YES ◄─┴─► NO
             └───────────────────────────┘                  │           │
                                                              ▼           ▼
                                              ┌─────────────────┐  ┌──────────────────┐
                                              │ Still prefer full │  │ Single-leg rollback│
                                              │ tuple rollback —  │  │ is acceptable (e.g. │
                                              │ cost of restoring  │  │ revert only the     │
                                              │ 3 legs is low,     │  │ prompt version) —   │
                                              │ cost of a wrong    │  │ still LOG it as a   │
                                              │ guess under        │  │ rollback event with  │
                                              │ incident pressure   │  │ full tuple recorded  │
                                              │ is high.            │  │ for audit purposes.  │
                                              └─────────────────┘  └──────────────────┘

                     ┌───────────────────────────────────────────┐
                     │  Is this a NEW release about to ship        │
                     │  (not yet an incident)?                       │
                     └───────────────────┬───────────────────────┘
                            YES ◄──────────┴──────────► N/A (already an incident — see above)
                             │
                             ▼
             ┌───────────────────────────────────────────┐
             │  Use a CANARY ramp (5% → 25% → 100%) with    │
             │  automatic rollback-to-zero on any threshold  │
             │  breach, rather than a direct 100% cutover —  │
             │  regardless of how confident offline eval      │
             │  results looked. Offline eval cannot fully     │
             │  simulate real production input distribution.  │
             └───────────────────────────────────────────┘
```

### 3c. Should this promotion be automatic or require a human sign-off?

```
                     ┌───────────────────────────────────────────┐
                     │  Does the domain carry regulatory,           │
                     │  safety-critical, or high-reputational-       │
                     │  risk weight (credit decisions, healthcare,   │
                     │  new jurisdiction launch, safety policy       │
                     │  changes)?                                     │
                     └───────────────────┬───────────────────────┘
                            YES ◄──────────┴──────────► NO
                             │                              │
                             ▼                              ▼
             ┌───────────────────────────┐      ┌───────────────────────────────┐
             │ Automated gate is the FLOOR │      │  Fully automated promotion is   │
             │ (must pass objective        │      │  appropriate: code-driven        │
             │ criteria), PLUS a required   │      │  gates only, no manual step —    │
             │ human sign-off before        │      │  this is the default and the     │
             │ promotion — never a human    │      │  module's central "never          │
             │ gate WITHOUT the automated    │      │  promote manually" principle      │
             │ floor underneath it.          │      │  applies directly.                │
             └───────────────────────────┘      └───────────────────────────────┘
```

---

## 4. How the Three Diagrams Relate

- Section 1 (system architecture) is the **static** picture: what components exist and how data flows between them at rest.
- Section 2 (sequence diagram) is the **dynamic/temporal** picture: the same components, but ordered through time across one full lifecycle including a real incident.
- Section 3 (decision trees) is the **decision-time** artifact: what a senior engineer should reach for *before* section 1 is even built, and what an on-call engineer should reach for *during* the incident depicted in section 2.

Together they answer the three questions posed at the top of `tutorial.md`: what's running (Section 1), how did it get there (Section 2, top half), and how do we get back (Section 2, bottom half, guided by Section 3b).

---

For citations, official documentation links, and source repositories referenced in these diagrams, see `references.md` and `github.md` in this same folder.
