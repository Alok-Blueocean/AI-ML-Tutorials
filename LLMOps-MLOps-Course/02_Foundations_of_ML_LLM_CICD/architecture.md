# Module 02 — Architecture Deep Dive: ML/LLM CI/CD Pipelines

This companion document expands on `tutorial.md` with larger system diagrams, a full
sequence diagram of a pipeline run (happy path and both failure paths), and a decision
tree for choosing which validation/eval tooling to reach for at each gate. Read
`tutorial.md` first — this document assumes its terminology (Gate 1/2/3, fail-fast,
idempotency) without re-explaining it.

---

## 1. End-to-End System Architecture — Full ML CI/CD/CT Platform

This is the "zoomed out" view: not just one pipeline run, but the surrounding platform
— source control, orchestrator, registry, serving, and monitoring — and how the three
gates from `tutorial.md` sit inside it. This is the shape a Netflix/Uber/Spotify-style
internal ML platform (Michelangelo, Metaflow-orchestrated pipelines) plausibly takes.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SOURCE OF TRUTH LAYER                                        │
│                                                                                                  │
│   ┌────────────────┐   ┌───────────────────┐   ┌────────────────────┐   ┌───────────────────┐ │
│   │  Git repo        │   │  Data warehouse /   │   │  Feature store      │   │  Prompt/config    │ │
│   │  (pipeline code,  │   │  data lake          │   │  (offline + online) │   │  registry (LLMOps │ │
│   │  Pandera schemas, │   │  (raw transactional │   │                      │   │  systems only)    │ │
│   │  eval configs)    │   │  data, vendor feeds)│   │                      │   │                   │ │
│   └────────┬─────────┘   └─────────┬─────────┘   └──────────┬─────────┘   └─────────┬─────────┘ │
│            │                       │                         │                        │           │
└────────────┼───────────────────────┼─────────────────────────┼────────────────────────┼───────────┘
             │                       │                         │                        │
             │  triggers             │  read by                │  read by               │  read by
             ▼                       ▼                         ▼                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              ORCHESTRATION / CI-CD-CT LAYER                                     │
│                    (GitHub Actions / Airflow / Metaflow / Vertex Pipelines)                     │
│                                                                                                  │
│   TRIGGERS:  [ code push ]   [ cron schedule ]   [ upstream data-arrival event ]                │
│                     │                │                        │                                │
│                     └────────────────┴────────────────────────┘                                │
│                                       ▼                                                          │
│                         ┌─────────────────────────┐                                             │
│                         │   STAGE: fetch/prepare    │                                            │
│                         └─────────────┬───────────┘                                             │
│                                       ▼                                                          │
│                    ┌──────────────────────────────────┐                                         │
│                    │  GATE 1 — DATA VALIDATION           │  fail ──────────┐                      │
│                    │  schema (Pandera) + row-count +     │                 │                      │
│                    │  drift (KS/PSI vs. baseline)         │                 │                      │
│                    └──────────────┬───────────────────┘                 │                      │
│                                   ▼ pass                                 │                      │
│                    ┌──────────────────────────────────┐                 │                      │
│                    │  STAGE: train candidate model        │                 │                      │
│                    └──────────────┬───────────────────┘                 │                      │
│                                   ▼                                       │                      │
│                    ┌──────────────────────────────────┐                 │                      │
│                    │  GATE 2 — MODEL EVALUATION            │  fail ──────────┤                      │
│                    │  metric >= baseline * factor (MLflow) │                 │                      │
│                    └──────────────┬───────────────────┘                 │                      │
│                                   ▼ pass                                 │                      │
│                    ┌──────────────────────────────────┐                 │                      │
│                    │  GATE 3 — RESPONSE/PROMPT QUALITY     │  fail ──────────┤                      │
│                    │  (LLM systems only — LLM-judge score  │                 │                      │
│                    │   vs. golden eval set)                │                 │                      │
│                    └──────────────┬───────────────────┘                 │                      │
│                                   ▼ pass                                 ▼                      │
│                    ┌──────────────────────────────────┐   ┌───────────────────────────┐         │
│                    │  REGISTER candidate as new version   │   │  BLOCK. Emit structured    │         │
│                    │  (MLflow Model Registry)               │   │  JSON failure log.          │         │
│                    └──────────────┬───────────────────┘   │  Notify (Slack/PagerDuty). │         │
│                                   │                          │  PREVIOUS model/prompt      │         │
│                                   │                          │  version keeps serving.     │         │
│                                   ▼                          └───────────────────────────┘         │
│                    ┌──────────────────────────────────┐                                          │
│                    │  Notify team (Slack): version +      │                                          │
│                    │  metric delta                        │                                          │
│                    └──────────────┬───────────────────┘                                          │
└──────────────────────────────────┼──────────────────────────────────────────────────────────────┘
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     DEPLOYMENT LAYER                                           │
│                                                                                                  │
│    ┌───────────────────┐        ┌───────────────────────┐        ┌───────────────────────┐    │
│    │  Staged rollout      │───────►│  Serving infrastructure │───────►│  Live production traffic │    │
│    │  (canary → % ramp)   │        │  (model server / LLM     │        │                          │    │
│    │                      │        │   gateway)                │        │                          │    │
│    └───────────────────┘        └───────────┬───────────┘        └───────────────────────┘    │
│                                              │                                                   │
└──────────────────────────────────────────────┼───────────────────────────────────────────────────┘
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   MONITORING / CT FEEDBACK LAYER                                │
│                                                                                                  │
│   Same drift/quality metric computations as Gate 1/2/3, run continuously on live traffic:      │
│   feature PSI drift dashboards · prediction quality proxies · business KPI tracking ·           │
│   LLM response-quality sampling. Anomalies here can themselves become a new TRIGGER              │
│   ("data drift detected" → kick off an out-of-schedule retrain) — closing the loop back          │
│   to the ORCHESTRATION LAYER at the top. This is what makes the system Continuous TRAINING,      │
│   not just Continuous Deployment.                                                                │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Key structural point:** the monitoring layer at the bottom computes *the same metrics*
as the gates in the middle. This is deliberate, not incidental — a PSI computation that
blocks a pipeline pre-deploy and a PSI computation that fires a dashboard alert
post-deploy should be the same function, called at two different points in the
artifact's lifecycle. Teams that implement these as two unrelated codebases (one owned
by the ML platform team, one owned by an observability team) tend to end up with drift
definitions that silently diverge over time.

---

## 2. Sequence Diagram — One Full Pipeline Run (All Three Branches)

This ASCII sequence diagram traces a single scheduled run of the churn-retrain pipeline
through the actors involved, showing the happy path and both gate-blocked branches from
`tutorial.md` §3.3 as alternate continuations from the same run.

```
 Cron       GitHub        fetch_data.py   validate_data.py   train.py    evaluate_model.py   MLflow      Slack
 (03:00)    Actions                                                                          Registry
   │           │                │                │               │              │              │           │
   │  trigger  │                │                │               │              │              │           │
   ├──────────►│                │                │               │              │              │           │
   │           │  run job       │                │               │              │              │           │
   │           ├───────────────►│                │               │              │              │           │
   │           │                │  pull last 30d │               │              │              │           │
   │           │                │  transactions  │               │              │              │           │
   │           │                │◄ ─ ─ ─ ─ ─ ─ ─ ┤               │              │              │           │
   │           │  data written  │                │               │              │              │           │
   │           │◄───────────────┤                │               │              │              │           │
   │           │                                  │               │              │              │           │
   │           │  invoke Gate 1                   │               │              │              │           │
   │           ├─────────────────────────────────►│               │              │              │           │
   │           │                                  │  row_count?   │              │              │           │
   │           │                                  │  schema OK?   │              │              │           │
   │           │                                  │  drift OK?    │              │              │           │
   │           │                                  │               │              │              │           │
   │           │           ┌──────────────────────┴───────────────────────────────────────────────────┐    │
   │           │           │ BRANCH A — HAPPY PATH: row_count=12,400 (>=1000), schema OK, drift OK        │    │
   │           │           └──────────────────────┬───────────────────────────────────────────────────┘    │
   │           │  exit 0 (pass, JSON log)          │               │              │              │           │
   │           │◄─────────────────────────────────┤               │              │              │           │
   │           │  invoke train                     │               │              │              │           │
   │           ├───────────────────────────────────────────────────►│              │              │           │
   │           │                                  │               │  train        │              │           │
   │           │                                  │               │  XGBoost;      │              │           │
   │           │                                  │               │  write         │              │           │
   │           │                                  │               │  dataset.hash  │              │           │
   │           │  model + hash written             │               │              │              │           │
   │           │◄───────────────────────────────────────────────────┤              │              │           │
   │           │  invoke Gate 2                                     │              │              │           │
   │           ├────────────────────────────────────────────────────────────────────►│              │           │
   │           │                                  │               │              │  AUC=0.84    │           │
   │           │                                  │               │              │  >= 0.82 ✓   │           │
   │           │  exit 0 (pass, JSON log)                          │              │              │           │
   │           │◄────────────────────────────────────────────────────────────────────┤              │           │
   │           │  register model                                                    │              │           │
   │           ├───────────────────────────────────────────────────────────────────────────────────►│           │
   │           │                                  │               │              │              │  version+1│
   │           │  registered                                                                        │           │
   │           │◄───────────────────────────────────────────────────────────────────────────────────┤           │
   │           │  notify (success)                                                                                │
   │           ├───────────────────────────────────────────────────────────────────────────────────────────────►│
   │           │                                                                                                  │  ✅ posted
   │           │                                  │               │              │              │           │
   │           │           ┌──────────────────────┴───────────────────────────────────────────────────┐    │
   │           │           │ BRANCH B — GATE 1 BLOCKED: ETL fault, only 600 rows (< 1000 minimum)         │    │
   │           │           └──────────────────────┬───────────────────────────────────────────────────┘    │
   │           │  exit 1 (fail, JSON log:          │               │              │              │           │
   │           │  {"check":"row_count",            │               │              │              │           │
   │           │   "found":600,"required":1000})   │               │              │              │           │
   │           │◄─────────────────────────────────┤               │              │              │           │
   │           │  SKIP train/evaluate/register — job marked failed by GitHub Actions                          │
   │           │  (previous model version keeps serving; nothing below this line executes)                    │
   │           │  notify (failure)                                                                              │
   │           ├───────────────────────────────────────────────────────────────────────────────────────────────►│
   │           │                                                                                                  │  ⛔ posted
   │           │                                  │               │              │              │           │
   │           │           ┌──────────────────────┴───────────────────────────────────────────────────┐    │
   │           │           │ BRANCH C — GATE 2 BLOCKED: Gate 1 passed, trained, but AUC=0.79 < 0.82        │    │
   │           │           └──────────────────────┬───────────────────────────────────────────────────┘    │
   │           │  (Gate 1 passed as in Branch A)   │               │              │              │           │
   │           │  invoke train ────────────────────────────────────►│              │              │           │
   │           │  model + hash written ◄───────────────────────────┤              │              │           │
   │           │  invoke Gate 2 ─────────────────────────────────────────────────────►│              │           │
   │           │                                  │               │              │  AUC=0.79    │           │
   │           │                                  │               │              │  < 0.82 ✗    │           │
   │           │  exit 1 (fail, JSON log:                                          │              │           │
   │           │  {"auc":0.79,"baseline":0.82})                                    │              │           │
   │           │◄────────────────────────────────────────────────────────────────────┤              │           │
   │           │  SKIP register — model trained "successfully" but is still rejected                          │
   │           │  (previous model version keeps serving)                                                       │
   │           │  notify (failure)                                                                              │
   │           ├───────────────────────────────────────────────────────────────────────────────────────────────►│
   │           │                                                                                                  │  ⛔ posted
```

**Reading this diagram:** the critical structural property is that **Branch B and
Branch C both terminate before `register model`** — the previous model version in
MLflow's registry is never touched. This is what "gates block promotion, not just log a
warning" means concretely: the registry's "current production model" pointer only ever
moves forward on Branch A.

---

## 3. Decision Tree — Choosing Validation/Evaluation Tooling for Each Gate

Use this when you're standing up a new pipeline and need to decide, concretely, what to
reach for at each gate.

```
                        ┌───────────────────────────────────────────┐
                        │  What are you validating right now?        │
                        └───────────────────┬─────────────────────┘
                                            │
        ┌───────────────────┬───────────────┼───────────────┬────────────────────┐
        ▼                   ▼               ▼               ▼                    ▼
   "Is the data       "Has the data     "Did I get      "Is the newly       "Is the LLM's
    structurally        distribution      enough data?"   trained model       generated output
    what I expect?"     changed?"                          actually good?"     good?"
        │                   │               │               │                    │
        ▼                   ▼               ▼               ▼                    ▼
 ┌─────────────┐   ┌──────────────────┐ ┌───────────┐ ┌──────────────────┐ ┌───────────────────┐
 │ SCHEMA        │   │ DRIFT / DISTRIBUTION│ │ ROW-COUNT / │ │ MODEL EVALUATION   │ │ RESPONSE/PROMPT     │
 │ VALIDATION    │   │ VALIDATION           │ │ COMPLETENESS│ │ GATE                │ │ QUALITY GATE        │
 └──────┬───────┘   └────────┬─────────┘ └─────┬─────┘ └────────┬─────────┘ └──────────┬────────┘
        │                    │                  │                │                       │
        ▼                    ▼                  ▼                ▼                       ▼
  How many features    How many features   Do you already   Do you already have   Do you have (or can
  / how complex are    do you need to        have a           an eval harness /     you afford to build)
  the constraints?      monitor, and how      well-defined     MLflow tracking       a golden eval set +
                        often?                held-out set?    set up?               an LLM-judge?
        │                    │                  │                │                       │
   ┌────┴────┐          ┌────┴─────┐        yes │ no       yes │ no                yes │ no
   ▼         ▼          ▼          ▼            ▼    ▼         ▼    ▼                  ▼    ▼
 Few,     Many,      A handful   Many features, Use    Define  Use     Start        Build a  Start
 simple   complex    of key      need a         mlflow one     mlflow  with a       small,   with a
 checks   nested     features,  standard,       .eval  first   .log_   simple       curated  single
   │      constraints reusable   pre-built      uate() (holdout,       metric()    golden    fixed
   ▼         │        metric      dashboard        │   labels,    with a          eval set  coherence/
 Custom      ▼          │        (PSI/KS/          │   min-size)  manual          (10s-100s groundedness
 assert   Pandera       ▼        Jensen-Shannon)    │      │      threshold        of         score via
 (df has  DataFrameSchema  Roll your own            ▼      ▼      comparison       prompts),  LLM-judge;
 the      with Check()  KS-test via                 Use    Use     until you       run it     expand only
 right     objects;     scipy.stats                 MLflow custom  can invest      every       once you
 columns)  lazy=True    .ks_2samp,                  (this  simple  in a full        pipeline    have real
           to collect   or PSI by                   module's       eval           run         production
           all failures hand for                    default        harness                    incidents
           at once      1-2 features                choice)                                    to mine
                              │
                              ▼
                        Adopt Evidently AI
                        Test Suites instead
                        of hand-rolling —
                        packages PSI/KS/
                        Jensen-Shannon as
                        ready CI pass/fail
                        checks
```

**How to use this tree in practice:** most teams start in the "few, simple checks" /
"1–2 features KS-test" / "MLflow with manual threshold" / "small golden eval set"
leaves — this is intentionally the cheapest possible version of all four gates, matching
the `validate_data.py` and `evaluate_model.py` examples in `tutorial.md`. Move right/down
the tree (Pandera schemas, Evidently Test Suites, full LLM-judge eval harnesses) as the
number of features, the number of models, or the production stakes grow — not before,
per the "don't over-gate exploratory work" guidance in `tutorial.md` §5 and §6.

---

## 4. Gate Placement Within a Single GitHub Actions Job vs. Across Jobs

A secondary architectural decision worth making explicit: should the three gates be
steps within one job, or separate jobs (possibly even separate workflows)?

```
 OPTION A — single job, sequential steps (what tutorial.md's example workflow uses)
 ┌─────────────────────────────────────────────────────────────────┐
 │ job: retrain                                                      │
 │   step: fetch data                                                 │
 │   step: GATE 1                                                     │
 │   step: train                                                      │
 │   step: GATE 2                                                     │
 │   step: register                                                   │
 │   step: notify                                                     │
 └─────────────────────────────────────────────────────────────────┘
   + simplest to read top-to-bottom; steps share the same filesystem/workspace
     (no artifact upload/download needed between steps)
   + a step failing automatically skips all subsequent steps in the job
   - all steps share one timeout budget and one runner; hard to give Gate 3's
     expensive LLM-judge eval a different runner size/timeout than Gate 1's
     cheap schema check

 OPTION B — separate jobs with needs: (better once Gate 3 becomes expensive)
 ┌───────────────┐   needs   ┌───────────────┐   needs   ┌───────────────────┐
 │ job: data_gate  │──────────►│ job: train_eval │──────────►│ job: quality_gate   │
 │ (small runner,  │           │ (GPU/larger      │           │ (separate runner,    │
 │  fast timeout)  │           │  runner, longer   │           │  own timeout, can     │
 │                 │           │  timeout)         │           │  run in parallel with │
 │                 │           │                   │           │  register step below) │
 └───────────────┘           └───────────────┘           └───────────────────┘
   + each job can have its own runner size, timeout, and required secrets
     (principle of least privilege — the schema-check job never needs registry
     write credentials at all)
   + jobs run in parallel where the dependency graph (needs:) allows it
   - artifacts (the trained model, the dataset) must be explicitly passed between
     jobs via actions/upload-artifact + actions/download-artifact, adding a
     little plumbing

 Guidance: start with Option A. Move to Option B once (a) Gate 3's LLM-judge
 evaluation cost/runtime is meaningfully larger than Gates 1-2, or (b) you need
 different credential scopes per gate for security reasons (see tutorial.md §6,
 "Security").
```
