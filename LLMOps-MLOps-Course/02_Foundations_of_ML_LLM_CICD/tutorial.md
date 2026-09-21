# Module 02 — Foundations of ML and LLM CI/CD

> Series: *Foundations of CI/CD for ML and LLM Systems*
> This module answers one question in three parts: **what breaks when you point a normal
> DevOps pipeline at a machine learning system, why it breaks, and exactly what you add
> to a `.github/workflows/*.yml` file to stop it from breaking.**

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain, precisely and defensibly (e.g. in an interview), *why* ML and LLM systems
   need CI/CD gates that plain DevOps pipelines do not — and name each gate.
2. Draw and explain the difference between DevOps, MLOps, and LLMOps release units,
   versioning scope, and testing scope.
3. Design a trigger-driven, gate-based ML pipeline (scheduled retraining, fail-fast,
   idempotent) and implement it in GitHub Actions.
4. Implement all three "extra" gates in code: schema validation (Pandera), distribution
   / drift validation (SciPy KS-test and PSI), row-count / completeness validation, and
   a model-evaluation gate (MLflow) — wired together with `sys.exit(1)` semantics so a
   failure actually blocks the pipeline instead of just logging a warning.
5. Reason about how a company like Netflix, Uber, or a frontier LLM lab would plausibly
   structure these gates at production scale, and articulate the tradeoffs of doing so.
6. Recognize the common ways teams silently defeat these gates (the failure modes this
   module exists to prevent) and correct them.

### Prerequisites

- Comfortable with Python (functions, exceptions, `sys.exit`, basic OOP).
- Basic ML literacy: what training/validation/test splits are, what accuracy/AUC/NDCG
  roughly measure, what "a model" is as an artifact (weights + config, not code).
- Basic Git/GitHub familiarity: commits, branches, pull requests.
- Helpful but not required: prior exposure to any CI/CD tool (Jenkins, GitLab CI,
  CircleCI, GitHub Actions) — this module uses GitHub Actions as the running example
  because it is free for public repos, ubiquitous, and maps cleanly onto the concepts.

### Key Terminology

| Term | Definition |
|---|---|
| **CI (Continuous Integration)** | Automatically building and testing every code change before it merges. |
| **CD (Continuous Delivery/Deployment)** | Automatically packaging and releasing validated changes into an environment. |
| **CT (Continuous Training)** | The ML-specific addition: automatically retraining a model when new data arrives or performance drifts, without a human manually kicking off a training run each time. |
| **Quality gate** | A pass/fail checkpoint in a pipeline that must succeed before execution is allowed to continue; implemented as a step that exits non-zero on failure. |
| **Schema validation** | Checking that a dataset's columns, types, and value constraints match an expected contract before it is used. |
| **Data drift** | A change in the statistical distribution of input features between the data a model was trained on and the data it now sees in production or in a new training batch. |
| **PSI (Population Stability Index)** | A single scalar summarizing how much a feature's distribution has shifted between two samples (baseline vs. current); the standard "traffic light" drift metric in production ML. |
| **KS-test (Kolmogorov–Smirnov)** | A statistical test comparing two samples' cumulative distributions; used to detect distribution drift for continuous features. |
| **Model evaluation gate** | A CI/CD step that blocks promotion of a newly trained model unless it beats (or nearly matches) a baseline metric — e.g. AUC, accuracy, NDCG. |
| **Idempotency (in a pipeline)** | Re-running the pipeline with the same inputs produces the same output (same model hash) — critical for safe retries after a transient failure. |
| **Silent failure** | A production incident where all infrastructure health checks are green (service is "up", deploy succeeded) but the model's actual predictive/business quality has degraded — the defining risk this module addresses. |
| **Model registry** | A versioned, queryable store of trained model artifacts plus their metadata (metrics, lineage, stage/alias) — e.g. MLflow Model Registry. |
| **Prompt/response quality gate (LLMOps)** | The LLM-specific analogue of a model evaluation gate: automated checks (coherence, groundedness, toxicity, hallucination rate) run against a fixed eval set before a new prompt/model/RAG config is promoted. |

---

## 2. Why This Topic Matters, and Where It Fits in the Lifecycle

Every ML or LLM system you will ever operate lives on a spectrum of *how much can
silently go wrong without anyone noticing.* A typical web service either works
(responds 200, correct payload) or it doesn't (500s, timeouts, alerts fire). An ML
service can respond 200, with a well-formed payload, on every single request — and
still be **wrong**, because a model's *correctness* is a statistical property of its
outputs over time, not a property you can assert on any single request the way you
assert an HTTP status code. That is the single sentence that explains everything in
this module. Once you internalize it, every gate described below stops being an
arbitrary checklist item and becomes an obvious answer to "how would I actually know if
this silently broke?"

Where this sits in the larger MLOps/LLMOps lifecycle:

```
 Data Collection → Feature Engineering → Training → Validation → Registry → Deployment → Serving → Monitoring
                                            ^^^^^^                                          ^^^^^^^^^^
                                     THIS MODULE lives                                 (later module —
                                     here: the automated                                observability,
                                     gates that decide                                   drift monitoring
                                     whether an artifact                                 in production —
                                     is allowed to move                                  builds on the
                                     to the next stage.                                  same drift metrics
                                                                                          introduced here)
```

This module is the connective tissue between "we trained a model" and "we trust this
model enough to let it touch a customer." Skipping it doesn't mean you have no gates —
it means you have exactly one gate (`unit tests pass`), which, as the case studies below
show, is precisely the gate that a corrupted-data incident sails straight through.

It is mid-2026, and the industry consensus reflected in current guidance (MLflow's own
2026 best-practices article on pipeline automation, Google Cloud's MLOps whitepaper, and
the CD4ML framework from ThoughtWorks) has converged hard on this exact framing: CI is
no longer "test the code," it is "test the code, the data, and the model," and CD is no
longer "deploy the artifact," it is "deploy a pipeline that can itself be re-run to
reproduce the artifact." If you are interviewing for a senior MLOps/LLMOps role in 2026,
this is table-stakes framing — expect to be probed on it directly.

---

## 3. Main Concepts

### 3.1 Concept: Why ML/LLM CI/CD Needs Three Extra Gates

#### Theory

Traditional DevOps CI/CD is **code-centric**. Its implicit assumption is: *if the code
is correct and the tests pass, the behavior is correct,* because in a deterministic
software system, behavior is a pure function of code (given the same inputs). CI/CD
therefore only needs to validate one thing that changes over time: the code.

ML and LLM systems break that assumption in a specific, mechanical way: **behavior is a
function of code AND data AND model artifact AND (for LLMs) prompt/retrieval
configuration** — and of these four, only one (code) is what traditional CI/CD checks.
The other three can each independently change without a single line of code changing:

- A vendor upstream changes a column's format or introduces nulls → **data changed**.
- A nightly retrain job runs on that data → **model artifact changed**.
- Someone tweaks a system prompt or the retrieval `top_k` → **prompt/config changed**.

None of these trigger a code review by default. None of them fail a unit test. All of
them can degrade production quality. This is why the pipeline needs three extra gates
layered on top of the standard build/unit-test/integration-test/deploy gates:

| # | Gate | What it catches | Analogous DevOps gate it extends |
|---|---|---|---|
| 1 | **Data validation** | Schema drift, missing/null values, incomplete loads (partial ETL) | Static analysis / linting (validates structure, not just syntax) |
| 2 | **Model evaluation** | A newly trained model that is objectively worse than the one in production | Integration tests (validates behavior against a known-good baseline) |
| 3 | **Response/prompt quality** (LLM-specific) | Hallucination, incoherent explanations, unsafe or off-policy outputs from a new prompt/model/RAG version | End-to-end / acceptance tests (validates the actual user-facing output) |

**Problems this solves:** it converts "unknown unknowns" (a vendor silently changed a
field) into "known knowns" that fail loudly and early, at the cheapest possible point in
the pipeline, instead of being discovered days later via a dashboard showing a metric
quietly bleeding out.

**Tradeoffs:** every gate adds latency and (real) compute/engineering cost to the
pipeline. A schema check on a 50-column table is cheap; a full LLM-judge-based response
quality evaluation over a large eval set is not free — it costs inference-time compute
and wall-clock minutes per run. The senior engineering judgment call is *where* to place
each gate (cheapest, most-likely-to-fail checks first — this is "fail fast," covered in
3.3) and *how strict* each threshold should be (too strict → constant false-positive
blocks and alert fatigue; too loose → the gate is theater).

**When to use:** any ML/LLM system with a recurring training or prompt-update cadence,
or any system where inputs are supplied by an external/upstream party you do not fully
control (a different team's ETL job, a third-party data vendor, user-generated content).
**When NOT to bother with the full three-gate machinery:** a one-off notebook model that
will never be retrained and never touches production traffic doesn't need a CI/CD
pipeline at all — gates are an investment that pays off across *repeated* runs. Don't
add cron-triggered CI/CD ceremony to a model that will be trained exactly once.

#### Architecture

```
                         ┌───────────────────────────────────────────────────────┐
                         │                 TRADITIONAL DEVOPS CI/CD                │
                         │                                                         │
   code change ────────► │  build → unit tests → integration tests → deploy       │
                         └───────────────────────────────────────────────────────┘
                                     assumes: code correct ⇒ behavior correct


                         ┌────────────────────────────────────────────────────────────────────┐
                         │                          ML / LLM CI/CD                              │
                         │                                                                      │
  code change   ────┐    │                                                                      │
  data change  ────┼───► │  build → unit tests → integration tests →                            │
  model retrain ───┘    │                                                                        │
                         │        ┌──────────────────┐   ┌───────────────────┐   ┌─────────────┐│
                         │  ─────►│ GATE 1: data       │──►│ GATE 2: model      │──►│ GATE 3: prompt/││
                         │        │ validation         │   │ evaluation         │   │ response quality││
                         │        │ (schema, drift,    │   │ (metric ≥ baseline)│   │ (LLM systems   ││
                         │        │  row-count)        │   │                    │   │  only)         ││
                         │        └──────────────────┘   └───────────────────┘   └─────────────┘│
                         │              │ fail              │ fail                  │ fail        │
                         │              ▼                   ▼                       ▼             │
                         │        BLOCK — keep prior model serving, alert team, stop here          │
                         │                                                                         │
                         └────────────────────────────────────────────────────────────────────────┘
```

#### Examples

- **Beginner:** A weekly Jupyter-notebook-turned-script retrains a small scikit-learn
  model on a CSV someone manually exports. Adding *one* gate — "does this CSV have the
  same columns as last week?" — before training is already a massive reliability win
  over "just run the notebook and hope."
- **Intermediate:** The churn-prediction pipeline from the transcripts: a scheduled
  GitHub Actions job pulls 30 days of transactional data, runs a Pandera schema check
  and row-count check (Gate 1), trains, and compares accuracy against
  `baseline * 0.99` (Gate 2) before registering to MLflow.
- **Production-grade:** A recommendation system (Netflix-style) with Gate 1 (schema +
  null-rate check on the vendor genre feed), Gate 2 (NDCG@10 ≥ baseline × 0.99 against a
  held-out evaluation slice, computed via MLflow's evaluation API), and — because the
  system also generates natural-language explanations ("Because you watched…") — Gate 3
  (an LLM-judge-scored coherence check ≥ 0.85 on a fixed set of explanation prompts)
  before any of it reaches a production traffic percentage.

#### Code — the "three gates" as a single orchestrating script

This is the shape every concrete implementation in this module converges on. Each gate
is its own function; each function either returns cleanly or calls `sys.exit(1)`. This
single pattern is what turns "validation" from a Jupyter cell into an enforceable
CI/CD control.

```python
# gates/run_all_gates.py
"""
Orchestrates the three ML/LLM CI/CD gates. Designed to be called as a single
GitHub Actions step so the whole gate sequence produces one clear pass/fail signal,
while each gate still logs structured (JSON) output for debugging and audit.
"""
import sys
import json
import logging

from gates.data_gate import validate_data
from gates.model_gate import evaluate_model
from gates.quality_gate import evaluate_response_quality  # LLM systems only

logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger("cicd.gates")


def emit(gate: str, status: str, **details):
    """Structured, machine-parseable log line — the contract every gate must honor."""
    log.info(json.dumps({"gate": gate, "status": status, **details}))


def main():
    # GATE 1 — data validation (schema + row-count + null checks)
    try:
        stats = validate_data("data/latest_batch.parquet")
        emit("data_validation", "pass", **stats)
    except Exception as e:
        emit("data_validation", "fail", error=str(e))
        sys.exit(1)  # hard stop — never spend compute training on bad data

    # GATE 2 — model evaluation against baseline
    try:
        metrics = evaluate_model(data_path="data/latest_batch.parquet")
        emit("model_evaluation", "pass", **metrics)
    except Exception as e:
        emit("model_evaluation", "fail", error=str(e))
        sys.exit(1)  # hard stop — never register/promote an underperforming model

    # GATE 3 — response/prompt quality (only present in LLM-serving pipelines)
    if evaluate_response_quality is not None:
        try:
            quality = evaluate_response_quality()
            emit("response_quality", "pass", **quality)
        except Exception as e:
            emit("response_quality", "fail", error=str(e))
            sys.exit(1)

    emit("pipeline", "all_gates_passed")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

---

### 3.2 Concept: DevOps vs. MLOps vs. LLMOps — Scope Comparison

#### Theory

The three "-Ops" disciplines are not competing methodologies — they are **nested,
expanding scopes**, each a superset of the previous one's concerns applied to a wider
release unit. DevOps manages software delivery. MLOps manages software *plus data and
model* delivery. LLMOps manages software plus model *plus prompt, retrieval, and context*
delivery. A team practicing LLMOps still needs every DevOps discipline (code review, CI
builds, container security scanning) — it just isn't sufficient on its own.

| Dimension | DevOps | MLOps | LLMOps |
|---|---|---|---|
| **What gets versioned** | Application source code | Code + datasets + trained model artifacts + feature-pipeline definitions | Code + model config/weights (if fine-tuned) + prompts + retrieval index/config + eval sets |
| **What gets tested** | Unit tests, integration tests, static analysis | Unit/integration tests **+ data validation + model evaluation against baseline** | All of MLOps **+ response/prompt quality, hallucination rate, safety/toxicity, retrieval relevance** |
| **Deployment artifact** | Application binary / container image | Container image **+ a specific model version pinned by ID/hash** | Container image **+ model/API version + prompt template version + retrieval/index version** — the full "release bundle" |
| **What can cause a regression with zero code change** | Nothing (behavior is a pure function of code) | Upstream data drift, stale training data, retraining on corrupted data | All MLOps causes **+ an upstream foundation-model version bump, a vector-index rebuild, a silent prompt edit by a non-engineer** |
| **Primary release cadence driver** | Feature/bug-fix velocity | Data freshness / drift schedule (e.g. daily retrain) | Data freshness **+** foundation model release cycle **+** prompt iteration velocity (often the fastest-changing artifact of all) |
| **"Green" pipeline guarantees...** | The code behaves as tested | The code behaves as tested, on data resembling what it was validated against | The code behaves as tested, on data resembling what it was validated against, **and the generated responses meet a quality bar sampled at eval time** — never a 100% guarantee for open-ended generation |

**Why this matters as a mental model:** in an interview or an architecture review, when
someone says "we already have CI/CD, why do we need more infrastructure for the ML
team," this table is your answer in one sentence: **the release unit got wider, so the
test surface has to get wider with it.**

#### Architecture

```
 DevOps  release unit:   [ code ]
 MLOps   release unit:   [ code | data | model artifact ]
 LLMOps  release unit:   [ code | model/API version | prompt template | retrieval/index config | eval-set version ]

              wider release unit  ─────────────────────────────────────►

 Each additional bracket compartment is a NEW axis of independent change,
 and therefore a NEW axis that must be versioned, tested, and gated.
```

#### Examples

- **Beginner:** A Flask app with a `requirements.txt` pin — pure DevOps.
- **Intermediate:** A scikit-learn churn model retrained nightly, versioned in MLflow,
  gated on AUC — MLOps.
- **Production-grade:** A customer-support RAG assistant where the underlying LLM API
  version, the embedding model, the vector index snapshot, and the system prompt are
  *each independently versioned and each independently able to trigger a re-run of the
  evaluation suite* — full LLMOps.

#### Code — expressing the release bundle as an explicit, versioned manifest

A practical technique used by teams that have matured past "the prompt lives in a
Python string in a random file": pin the entire LLMOps release unit as one artifact.

```python
# release_manifest.py — the LLMOps "release unit" made explicit and diffable
from dataclasses import dataclass, asdict
import hashlib
import json


@dataclass(frozen=True)
class ReleaseManifest:
    code_commit_sha: str
    model_provider: str          # e.g. "anthropic"
    model_id: str                # e.g. "claude-sonnet-5-20260115"
    prompt_template_version: str # e.g. "support_agent_v14"
    retrieval_index_version: str # e.g. "kb_index_2026_07_29"
    eval_set_version: str        # e.g. "golden_eval_v6"

    def content_hash(self) -> str:
        payload = json.dumps(asdict(self), sort_keys=True).encode()
        return hashlib.sha256(payload).hexdigest()[:12]


# Every CI run diffs the new manifest against the currently-deployed one.
# ANY field changing re-triggers the full response-quality gate (Gate 3) —
# not just code changes. This is the concrete mechanism behind the theory
# point "these systems can regress even when no code changes are made."
```

---

### 3.3 Concept: Trigger-Driven, Gate-Based, Fail-Fast Pipeline Automation

#### Theory

Four automation principles turn a script into a production-grade pipeline:

1. **Trigger-driven** — pipelines run on an event (code push), a schedule (cron), or a
   pre-deployment hook — not "whenever someone remembers to run it." Continuous
   Training (CT) specifically depends on the *schedule* trigger, since retraining is
   often decoupled from any code change at all.
2. **Gates at each stage** — every stage produces a pass/fail signal *before* the next
   stage is allowed to run, rather than one big script that runs end-to-end and then
   someone manually eyeballs the output.
3. **Fail fast** — order gates from cheapest/most-likely-to-catch-a-problem to most
   expensive, and stop at the first failure. Checking that a data file has >1000 rows
   costs milliseconds; training an XGBoost model costs minutes to hours. Never spend
   the expensive step's compute validating something the cheap step could have caught.
4. **Treat the output as a promotable artifact** — a trained model is not "done" when
   training finishes; it is a *candidate* that earns promotion only by passing every
   gate, exactly like a build artifact in traditional CD earns promotion from
   staging to production only after passing its test suite.

**Problems this solves:** wasted compute (training on data that was already known-bad),
wasted human attention (someone has to notice a bad model got promoted), and — the big
one — **non-determinism management**. Two training runs on "the same" code can produce
different weights due to random seeds, data ordering, or hardware nondeterminism in GPU
kernels. Because ML runs are not guaranteed bit-for-bit reproducible the way a compiled
binary is, the pipeline's *job* is to make the observable outcome (does it pass the
gates? does it hash the same, given the same inputs?) deterministic even when the
underlying training process is not perfectly so.

**Tradeoffs:** more gates and more automation means more pipeline code to maintain, more
CI minutes billed, and more alert channels to tune. A pipeline with 15 finicky gates
that pages someone every night for a threshold miss of 0.1% will get its alerts muted
within a month — gate design is itself an engineering discipline (see Common Mistakes,
§5).

**When to use:** any recurring training/deployment cadence. **When not to over-engineer
this:** a research/exploration phase, before a model has a real production consumer,
benefits from loose, fast iteration — bolt on the full gate machinery once the model is
being seriously considered for production, not before (over-gating an experimental
notebook just slows down exploration for no safety benefit).

#### Architecture — the churn-model daily retrain pipeline (from the transcript, expanded)

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│  TRIGGER: cron "0 3 * * *"  (03:00 daily)                                          │
└───────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 1 — Pull last 30 days of      │
                 │ transactional data                 │
                 └──────────────────────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 2 — GATE: Data validation      │   fail ──► BLOCK. Keep prior
                 │  • Pandera schema check             │            model serving.
                 │  • row_count >= 1000                │            Alert. STOP. (no
                 └──────────────────────────────────┘            training compute spent)
                                    │ pass
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 3 — Train + evaluate           │
                 │  candidate model                    │
                 └──────────────────────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 4 — GATE: Model evaluation      │  fail ──► BLOCK. Do not
                 │  accuracy >= baseline * 0.99         │            register. Alert.
                 └──────────────────────────────────┘            Prior model stays live.
                                    │ pass
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 5 — Register to MLflow          │
                 │  (run_id, dataset hash, timestamp)   │
                 └──────────────────────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────┐
                 │ STEP 6 — Notify (Slack webhook):     │
                 │  new version + accuracy delta        │
                 └──────────────────────────────────┘
                                    │
                                    ▼
                              REPEAT tomorrow at 03:00
```

#### Examples

- **Beginner:** A GitHub Actions workflow with a single `schedule:` trigger that just
  runs `python train.py` — no gates. This is the *starting point*, not the goal; it
  demonstrates the trigger principle alone.
- **Intermediate:** The churn model above — trigger + 2 gates (data, model) +
  registry + notification. This is the minimum viable production pattern for a
  classical ML retrain loop.
- **Production-grade:** An e-commerce churn model using XGBoost, gated on
  AUC ≥ 0.82, registered to MLflow, with three concrete tested outcomes:
  - **Happy path:** 12,400 rows pulled, schema passes, AUC = 0.84 ≥ 0.82 → registered,
    team notified.
  - **Gate 1 blocked:** an ETL fault delivers only 600 rows (< minimum) → training is
    skipped entirely, previous stable model stays live — fail-fast in action.
  - **Gate 2 blocked:** training runs fine, but AUC = 0.79 < 0.82 → model is *not*
    registered, an alert fires — the model trained successfully but is still rejected,
    which is exactly why "the training job succeeded" must never be conflated with "the
    model is good."

#### Code — the full GitHub Actions workflow

This is the centerpiece implementation artifact for this module: a complete, runnable
CI/CD workflow for the daily churn retrain, implementing trigger, both gates,
registration, and notification, with fail-fast ordering and idempotency via a dataset
hash.

```yaml
# .github/workflows/churn-retrain.yml
name: Daily Churn Model Retrain

on:
  schedule:
    - cron: "0 3 * * *"   # 03:00 UTC daily
  workflow_dispatch: {}    # allow manual trigger for debugging/backfills

concurrency:
  group: churn-retrain     # prevents overlapping runs if a prior run is still going
  cancel-in-progress: false

jobs:
  retrain:
    runs-on: ubuntu-latest
    timeout-minutes: 45

    steps:
      - name: Checkout pipeline code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      # --- STEP 1: pull fresh data -------------------------------------------------
      - name: Pull last 30 days of transactional data
        run: python pipeline/fetch_data.py --window-days 30 --out data/latest.parquet

      # --- GATE 1: data validation (fail-fast, cheapest check first) --------------
      - name: "GATE 1 — Validate data (schema, row count, nulls)"
        id: data_gate
        run: python pipeline/validate_data.py --input data/latest.parquet
        # validate_data.py calls sys.exit(1) on any failure — GitHub Actions
        # automatically marks this step (and the job) as failed, and every
        # step below is skipped. No training compute is spent on bad data.

      # --- STEP 2: train candidate model -------------------------------------------
      - name: Train candidate model (XGBoost)
        run: |
          python pipeline/train.py \
            --input data/latest.parquet \
            --output artifacts/model_candidate.json \
            --dataset-hash-out artifacts/dataset.hash

      # --- GATE 2: model evaluation against baseline -------------------------------
      - name: "GATE 2 — Evaluate model against baseline (AUC >= 0.82)"
        id: model_gate
        run: |
          python pipeline/evaluate_model.py \
            --model artifacts/model_candidate.json \
            --input data/latest.parquet \
            --baseline-auc 0.82

      # --- STEP 3: register only if both gates passed ------------------------------
      - name: Register model to MLflow
        if: success()   # unreachable if either gate above failed
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: |
          python pipeline/register_model.py \
            --model artifacts/model_candidate.json \
            --dataset-hash artifacts/dataset.hash \
            --run-timestamp "$(date -u +%Y-%m-%dT%H:%M:%SZ)"

      # --- STEP 4: notify regardless of outcome ------------------------------------
      - name: Notify Slack (success)
        if: success()
        run: |
          curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"✅ churn model retrained: new version registered.\"}" \
            "${{ secrets.SLACK_WEBHOOK_URL }}"

      - name: Notify Slack (failure — gate blocked)
        if: failure()
        run: |
          curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"⛔ churn retrain blocked at a quality gate — prior model still serving. Check run logs.\"}" \
            "${{ secrets.SLACK_WEBHOOK_URL }}"
```

**Idempotency note:** `pipeline/train.py` writes `dataset.hash` — a SHA-256 hash of the
input dataset — alongside the model. `register_model.py` uses this hash as an MLflow
tag. If the workflow is re-run (say, after a transient network failure) against the
*same* 30-day window, the resulting model hash is identical, and registration becomes a
safe no-op rather than creating a confusing duplicate "new" version. This is the
concrete mechanism behind "idempotency" from the transcript's fourth pipeline outcome.

---

### 3.4 Concept: Data and Model Validation Gates — Implementation Detail

#### Theory

This concept is the "how" behind Gates 1 and 2. Four validation types, each catching a
distinct failure mode that the others cannot:

| Validation type | What it checks | Catches | Tool | Fails via |
|---|---|---|---|---|
| **Schema validation** | Required columns exist; types correct; value ranges/constraints hold | Vendor format changes, unexpected categorical values (e.g. a churn label of `2` or `-1` instead of `{0,1}`), a numeric column silently becoming a string | **Pandera** | Raises `SchemaError` |
| **Distribution / drift validation** | Does the new batch's feature distribution still resemble the training/baseline distribution? | Gradual behavioral drift, seasonal effects, upstream sampling changes — none of which break the *schema*, only the *statistics* | **SciPy (`ks_2samp`)** or **PSI** | Custom threshold check |
| **Row-count / completeness validation** | Is the dataset large enough / complete enough to trust? | Partial ETL failures, upstream database outages, truncated exports | Custom (`assert len(df) >= MIN_ROWS`) | `AssertionError` / custom exception |
| **Model evaluation** | Does the newly trained model meet a minimum performance bar? | An objectively worse model that nonetheless "trained successfully" | **MLflow** (`mlflow.evaluate`) | Custom threshold check |

A critical nuance: **schema validation and distribution validation are not
redundant.** A batch of data can pass schema validation perfectly (all columns present,
all types correct, all values within declared ranges) while still representing a
*meaningfully different population* than what the model was trained on — e.g., a
sudden shift in customer geography mix after a marketing campaign. Schema validation
answers "is this data structurally the same kind of thing?" Distribution validation
answers "is this data statistically the same population?" You need both.

**PSI interpretation** (the standard convention used across the industry, including in
tools like Evidently AI):

| PSI value | Interpretation |
|---|---|
| < 0.10 | Stable — no significant population shift |
| 0.10 – 0.25 | Slight drift — worth watching, not necessarily blocking |
| > 0.25 | Significant drift — investigate, likely block promotion |

**Validation operating rules** (from the transcript, and standard practice):

1. Every validation script must produce **structured (JSON) logs** — machine-readable,
   so dashboards/alerting tools can parse *what* failed, not just *that* something
   failed.
2. Every failed validation must **exit with a non-zero code** (conventionally `1`).
   CI/CD systems make their continue/stop decision purely from the exit code of the
   last command run — this is the entire mechanism by which "validation" becomes an
   *enforced* gate rather than a report nobody reads.

**When to use each check:** run schema + row-count checks on *every* pipeline run — they
are cheap and catch the most common real-world failure (upstream data problems). Run
full distribution/PSI checks on every run too if compute allows; if not, consider
sampling or running the expensive drift check on every Nth run while keeping cheap
checks on every run. **When NOT to over-invest:** for a small, stable, internally
generated dataset with a controlled schema (e.g., data generated by your own upstream
service, not a third-party vendor), a lightweight schema+row-count check may be
sufficient without a heavyweight PSI/KS pipeline — match validation depth to how much
you actually control (and therefore trust) the upstream source.

#### Architecture

```
                incoming batch (data/latest.parquet)
                              │
                              ▼
        ┌───────────────────────────────────────┐
        │  validate_data.py                       │
        │                                          │
        │  1. row_count = len(df)                  │
        │     assert row_count >= MIN_ROWS  ───────┼──► fail → JSON log + sys.exit(1)
        │                                          │
        │  2. ChurnSchema.validate(df, lazy=True)  │
        │     (Pandera: columns, dtypes, ranges,   │
        │      null constraints all in one pass)   │──► fail → JSON log + sys.exit(1)
        │                                          │
        │  3. drift = ks_2samp(baseline[col],       │
        │             current[col])                │
        │     assert drift.pvalue > 0.05  ─────────┼──► fail → JSON log + sys.exit(1)
        │     (or: PSI(baseline, current) < 0.25)   │
        │                                          │
        │  ALL PASS → structured "pass" log,        │
        │  exit 0 → pipeline proceeds to training    │
        └───────────────────────────────────────┘
```

#### Examples

- **Beginner:** A single `assert df.shape[0] > 0` before training — catches the
  degenerate "empty file" case, nothing more.
- **Intermediate:** The churn model's `validate_data.py` (below) — Pandera schema +
  minimum row count (10,000) + JSON error output + `sys.exit(1)`.
- **Production-grade:** A layered validation service used by every model team in an
  org (Uber's Michelangelo-style internal platform is the plausible real-world
  analogue) that runs schema, PSI-per-feature, and row-count checks as a shared library,
  emitting a single structured "data quality report" artifact consumed both by the
  blocking CI/CD gate *and* by a long-running dashboard that tracks drift trends over
  weeks — the same drift computation feeding both a hard gate and a soft monitoring
  signal.

#### Code — `validate_data.py` (schema + row-count + drift, fully worked)

```python
# pipeline/validate_data.py
"""
Hard gate before training. Any failure here must stop the workflow —
it is far cheaper to reject bad data than to train, evaluate, and
possibly promote a model built on it.
"""
import sys
import json
import argparse

import pandas as pd
import pandera.pandas as pa
from pandera.pandas import Column, DataFrameSchema, Check
from scipy.stats import ks_2samp

MIN_ROWS = 10_000

# --- Gate 1a: schema ---------------------------------------------------------
ChurnSchema = DataFrameSchema(
    {
        "customer_id": Column(str, nullable=False, unique=True),
        "transaction_count_30d": Column(int, Check.ge(0)),
        "avg_spend_usd": Column(float, Check.ge(0.0)),
        "churn_label": Column(int, Check.isin([0, 1])),
    },
    strict=False,   # allow extra columns; only enforce the ones we declare
)


def emit(status: str, **details):
    print(json.dumps({"gate": "data_validation", "status": status, **details}))


def validate_row_count(df: pd.DataFrame) -> None:
    if len(df) < MIN_ROWS:
        emit("fail", check="row_count", found=len(df), required=MIN_ROWS)
        sys.exit(1)


def validate_schema(df: pd.DataFrame) -> None:
    try:
        ChurnSchema.validate(df, lazy=True)  # lazy=True: collect ALL failures, not just the first
    except pa.errors.SchemaErrors as err:
        emit("fail", check="schema", errors=err.failure_cases.to_dict(orient="records"))
        sys.exit(1)


def validate_drift(df: pd.DataFrame, baseline_path: str, column: str = "avg_spend_usd") -> None:
    baseline = pd.read_parquet(baseline_path)
    stat, p_value = ks_2samp(baseline[column], df[column])
    if p_value < 0.05:  # distributions likely different -> drift
        emit("fail", check="distribution_drift", column=column, ks_stat=stat, p_value=p_value)
        sys.exit(1)
    return {"ks_stat": stat, "p_value": p_value}


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    parser.add_argument("--baseline", default="data/baseline.parquet")
    args = parser.parse_args()

    df = pd.read_parquet(args.input)

    validate_row_count(df)
    validate_schema(df)
    drift_stats = validate_drift(df, args.baseline)

    emit("pass", row_count=len(df), drift=drift_stats)
    sys.exit(0)


if __name__ == "__main__":
    main()
```

#### Code — model evaluation gate with MLflow

```python
# pipeline/evaluate_model.py
"""
Gate 2: block promotion of a candidate model that doesn't meet the baseline bar.
"""
import sys
import json
import argparse

import mlflow
from sklearn.metrics import roc_auc_score
import joblib
import pandas as pd


def emit(status: str, **details):
    print(json.dumps({"gate": "model_evaluation", "status": status, **details}))


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", required=True)
    parser.add_argument("--input", required=True)
    parser.add_argument("--baseline-auc", type=float, required=True)
    args = parser.parse_args()

    model = joblib.load(args.model)
    df = pd.read_parquet(args.input)
    X, y = df.drop(columns=["churn_label"]), df["churn_label"]

    preds = model.predict_proba(X)[:, 1]
    auc = roc_auc_score(y, preds)

    with mlflow.start_run(run_name="churn_eval"):
        mlflow.log_metric("auc", auc)
        mlflow.log_param("baseline_auc", args.baseline_auc)

        if auc < args.baseline_auc:
            emit("fail", auc=auc, baseline=args.baseline_auc)
            sys.exit(1)  # do NOT register — prior model keeps serving

        emit("pass", auc=auc, baseline=args.baseline_auc)


if __name__ == "__main__":
    main()
```

---

## 4. Real-World Case Studies (Reasoned Inference)

> These describe how systems at these companies would **plausibly** be architected,
> based on publicly documented engineering patterns (Michelangelo, Metaflow, published
> MLOps/LLMOps whitepapers) — not confirmed internal implementation details.

**Netflix (recommendations).** Netflix's engineering blog has publicly described
Metaflow, its human-centric ML workflow framework, precisely because ordinary
software-engineering friction (versioning, dependency chains, retriable steps) was
identified as data scientists' biggest obstacle to shipping — exactly the gap this
module's three gates close. A recommendation pipeline structured this way would
plausibly express each of Gate 1 (schema/null-rate check on the vendor genre/metadata
feed), Gate 2 (NDCG@k against a held-out slice), and Gate 3 (an LLM-judge coherence
score on generated "because you watched" explanations) as independently retriable
Metaflow steps, each emitting metrics that both gate the pipeline and feed a long-lived
drift dashboard.

**Uber (Michelangelo).** Uber's own published description of Michelangelo shows an
internal ML-as-a-service platform spanning manage-data → train → evaluate → deploy →
predict → monitor as one integrated system. The plausible reasoning: at Uber's scale
(many product teams each shipping models — ETA prediction, fraud detection, pricing),
building the three gates once as *shared platform infrastructure* — a company-wide
schema-validation library, a shared model-evaluation harness, a shared registry — avoids
every team reinventing (and inevitably under-implementing) the same gates independently.

**OpenAI / Anthropic (frontier LLM providers).** A frontier lab's LLMOps pipeline for
its own hosted models would plausibly extend Gate 3 far beyond a single coherence score:
a large, continuously maintained eval suite (covering capability regressions, safety
behaviors, refusal calibration, and known jailbreak patterns) would need to pass before
any new model checkpoint, prompt/system-prompt default, or serving-infrastructure change
reaches production traffic — with staged rollout (canary → percentage ramp) functioning
as an additional gate layered *after* the offline eval suite, since offline evals alone
cannot fully capture real-world usage-distribution shift.

**Spotify (recommendations / personalization).** Given Spotify's publicly known
investment in experimentation infrastructure, a personalization pipeline there would
plausibly treat Gate 2 not as a single static threshold but as a multi-metric gate (e.g.
engagement proxy metrics plus fairness/diversity metrics) evaluated first offline
(the Gate 2 pattern in this module) and then online via a bandit/A-B rollout — offline
gates functioning as a cheap pre-filter before the more expensive, slower online
experiment gate.

**Databricks / NVIDIA (platform vendors).** Both companies build and sell the tooling
that *implements* these gates for other companies (Databricks: MLflow, Delta Live
Tables expectations; NVIDIA: NeMo Guardrails-style output validation for LLM pipelines).
It is reasonable to infer that their own internal model-shipping pipelines (for
Databricks' own foundation-model offerings, for NVIDIA's own NIM microservices) dogfood
these exact gate categories, since the vendors' public documentation frames them as
best practice for *any* team shipping models at scale — it would be inconsistent for
their own pipelines to skip the gates their tooling exists to enforce.

---

## 5. Common Mistakes

1. **Treating "unit tests pass" as "the model is fine."** The single biggest failure
   mode this whole module exists to prevent. Unit tests validate code paths, not data
   quality or model quality — a pipeline can have 100% unit test coverage and still
   train a materially worse model on corrupted data (the transcript's Netflix
   recommendation-quality scenario).
2. **Validating schema but not distribution (or vice versa).** As discussed in §3.4,
   these catch different failure classes. Schema-only validation misses gradual
   population drift; distribution-only validation (without a schema check) can be
   fooled by a type change that happens to still parse numerically.
3. **Logging validation failures as warnings instead of exiting non-zero.** A
   validation script that prints `"WARNING: schema mismatch"` and continues anyway is
   not a gate — it's a comment. The exit code is the entire enforcement mechanism.
4. **Setting the model-evaluation threshold as a static absolute value that's never
   revisited.** A baseline like `AUC >= 0.82` needs an owner and a review cadence;
   otherwise it either rots (too easy, letting mediocre models through as the problem
   space naturally gets harder) or blocks every legitimate retrain forever (too strict,
   set once during an unusually good training run).
5. **No fail-fast ordering.** Running the full (expensive) training job before checking
   whether the input data even has the right columns wastes compute and delays the
   failure signal by however long training takes.
6. **Ignoring idempotency.** Retrying a failed pipeline run without a dataset hash or
   equivalent can silently create a *different* "same-day" model version, making
   incident postmortems and rollback confusing.
7. **Conflating a green deployment with business success.** A model can deploy cleanly
   (infrastructure gate green) while quality silently degrades (business gate never
   checked) — this is the "silent failure" concept from §1, and it is the most
   expensive mistake on this list because it is invisible until someone notices the
   business metric moved.
8. **Over-gating exploratory/research pipelines.** The opposite failure: bolting the
   full three-gate, alert-heavy production machinery onto a notebook that three people
   are still iterating on daily, generating noise and slowing iteration for no
   corresponding safety benefit (see Theory "when not to use" in §3.1 and §3.3).
9. **For LLM systems: treating "the API call succeeded" as "the response quality was
   fine."** LLM APIs return 200 with well-formed JSON for both a brilliant answer and a
   confident hallucination — the response quality gate must be a *separate* check, not
   inferred from HTTP status.

---

## 6. Best Practices and Production Tips

**When to use full three-gate CI/CD:** any model or prompt configuration with a
recurring update cadence and a real production consumer. **When not to:** one-off
research models with no production path (skip gate ceremony; use lightweight sanity
checks instead) — introduce the full pipeline when the model is seriously being
considered for production, not before.

**Alternatives to hand-rolling each gate:**

| Gate | Hand-rolled | Managed/library alternative |
|---|---|---|
| Schema validation | Custom `assert` statements | **Pandera** (this module's choice), or Great Expectations |
| Distribution/drift | Raw `scipy.stats.ks_2samp` calls | **Evidently AI** Test Suites — packages PSI/KS/Jensen-Shannon as ready-made CI-friendly pass/fail checks, worth adopting once you have more than a couple of features to monitor |
| Model evaluation | Custom threshold comparison script | **MLflow** `mlflow.evaluate()` plus Model Registry stage/alias transitions gated by CI |
| Prompt/response quality | Custom LLM-judge prompt | Structured eval frameworks (an LLM-as-judge harness with a fixed golden eval set, versioned like any other test suite) |

**Cost and scaling:** cheap gates (schema, row-count) should run on *every* pipeline
execution regardless of scale. Expensive gates (full distribution checks across every
feature, large-eval-set LLM-judge scoring) may need sampling or reduced-frequency
execution as data volume or eval-set size grows — measure the marginal compute cost per
gate and place the expensive ones last, per fail-fast ordering.

**Monitoring:** the same drift/quality metrics computed for a hard gate at training time
should also feed a continuous *post-deployment* monitoring dashboard — drift doesn't
stop the moment a model is promoted; it's exactly as likely to develop gradually
afterward. Treat the pre-deploy gate and the post-deploy monitor as the same metric
computation, viewed at two different points in time, not two unrelated systems.

**Security:** validation scripts and evaluation harnesses often need read access to
production-adjacent data and write access to a model registry — scope CI/CD credentials
(e.g., `MLFLOW_TRACKING_URI` auth, cloud storage read tokens) as narrowly as possible,
and never let a validation/eval script have write access to the *serving* environment
directly — promotion to serving should be a separate, more tightly controlled step than
"the eval script said pass."

**Performance tradeoffs:** `lazy=True` in Pandera (collect all schema errors in one pass)
costs a bit more compute than stopping at the first failure, but is almost always worth
it — a single validation run that reports all five things wrong with a batch saves
four subsequent round trips compared to a validator that reports one error at a time.

**Rollback:** every gate failure should leave the *previous* model/prompt version still
serving — gates block *promotion*, they should never be implemented in a way that leaves
nothing serving. Confirm this explicitly in pipeline design and in tests of the pipeline
itself (test the failure path, not just the happy path).

---

## 7. Interview Questions

1. **"Why isn't a standard DevOps CI/CD pipeline sufficient for an ML system?"**
   *Model answer:* DevOps CI/CD assumes behavior is a pure function of code; testing
   the code is therefore sufficient. ML/LLM systems' behavior is also a function of
   data, the trained model artifact, and (for LLMs) prompt/retrieval configuration —
   any of which can change independently of code and degrade quality. So the pipeline
   needs additional gates: data validation, model evaluation, and (for LLMs)
   response/prompt quality — on top of, not instead of, standard code gates.

2. **"What's the difference between a schema validation check and a distribution/drift
   check, and why do you need both?"**
   *Model answer:* Schema validation checks structural correctness (columns present,
   correct types, values within declared constraints); it answers "is this the same
   kind of data?" Distribution/drift validation checks statistical similarity to a
   baseline (via KS-test or PSI); it answers "is this the same population of data?"
   Data can pass schema validation while representing a meaningfully shifted
   population (e.g., a marketing campaign changes customer geography mix) — so
   schema-only validation misses gradual drift, which is why both are needed.

3. **"How would you design a retraining pipeline to fail fast?"**
   *Model answer:* Order gates from cheapest/most-likely to catch a problem to most
   expensive: row-count and schema checks first (milliseconds, catch the most common
   real failure — partial/corrupted upstream data), then training (expensive), then
   model evaluation against baseline. Any gate failure calls `sys.exit(1)` (or
   equivalent), which a CI/CD system reads as "stop the workflow here" — so no compute
   is wasted training on data already known to be bad, and no bad model is ever
   registered.

4. **"What does idempotency mean in the context of an ML retraining pipeline, and why
   does it matter?"**
   *Model answer:* Idempotency means re-running the pipeline with the same input data
   produces the same output (ideally verifiable via a content hash of the resulting
   model). It matters because production pipelines get retried after transient
   failures (network errors, spot-instance preemption); without idempotency, a retry
   could produce a subtly different "new" model version, complicating incident
   postmortems, rollback, and trust in the registry's version history.

5. **"A production model's serving infrastructure shows 100% healthy status, but
   business metrics have degraded over the last week. What ML-specific failure mode is
   this, and how would you have caught it earlier?"**
   *Model answer:* This is a "silent failure" — infrastructure health (uptime,
   latency, error rate) is orthogonal to model quality, since a model can serve
   perfectly well-formed, low-latency predictions that are simply *wrong*. It should
   have been caught by continuous post-deployment monitoring using the same
   drift/quality metrics computed at the training-time gates (e.g., feature PSI vs.
   baseline, prediction-quality proxy metrics), rather than relying solely on
   infrastructure health checks.

6. **"How would AUC-baseline-comparison model evaluation gates need to change for an
   LLM-based system versus a classical ML classifier?"**
   *Model answer:* A classical model's evaluation gate compares a single scalar metric
   (AUC, accuracy) against a fixed baseline on a held-out set — cheap and largely
   deterministic. An LLM's evaluation gate must additionally assess open-ended,
   free-text output quality (coherence, factual grounding, absence of hallucination,
   safety) typically via an LLM-as-judge scored against a fixed golden eval set, which
   is more expensive, noisier (judge scores have variance), and needs its own
   validation (the judge itself can drift or be gamed) — so LLM eval gates are
   qualitatively, not just quantitatively, harder than classical ML eval gates.

7. **"Where would you place the model-evaluation gate relative to the model registry in
   an MLflow-based pipeline, and why?"**
   *Model answer:* Evaluation must happen *before* registration/promotion — evaluate the
   candidate model against the baseline first; only register to MLflow (and only
   transition it to the "production"/serving alias) if it passes. Registering first and
   evaluating after risks a race where something downstream picks up the
   not-yet-validated version before the evaluation gate has run.

8. **"What's a concrete way a team can accidentally defeat their own CI/CD data
   validation gate without realizing it?"**
   *Model answer:* Logging a schema/validation failure as a warning and letting the
   script continue with a `return` instead of `sys.exit(1)` — the check still "runs"
   and even "detects" the problem, but since the CI/CD system only reacts to the
   process's exit code, a non-zero-less failure is invisible to the pipeline and the
   bad data proceeds straight into training. This is why "produces structured logs" and
   "exits non-zero on failure" are treated as two separate, both-mandatory operating
   rules, not one.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

Traditional DevOps CI/CD validates code because, in traditional software, code is the
only thing that changes behavior. ML and LLM systems break that assumption: data, the
trained model artifact, and (for LLMs) prompt/retrieval configuration can each change
behavior independently of code, and none of that is caught by unit tests. This module
covered the three extra gates this requires (data validation, model evaluation,
response/prompt quality), the automation principles that make a retraining pipeline
production-grade (trigger-driven, staged gates, fail-fast, promotable artifacts), and
the concrete implementation of each validation type (Pandera schema, SciPy/PSI drift,
row-count, MLflow evaluation) enforced via structured JSON logs and `sys.exit(1)`.

### Key Takeaways

- **A green pipeline is not the same as a good model.** Infrastructure health and model
  quality are orthogonal; both need explicit checks.
- **The release unit gets wider as you move DevOps → MLOps → LLMOps** — code, then
  +data +model, then +prompt +retrieval +context — and the test surface must widen with
  it.
- **Order gates cheapest-first** and make every failure a hard, non-zero exit — a
  validation check that only logs a warning is not a gate.
- **Idempotency and structured logging** are what make a pipeline safely retriable and
  debuggable in production, not just correct on the happy path.
- **Match gate rigor to production stakes** — don't over-gate exploratory work, and
  don't under-gate anything with a real production consumer.

### Production Checklist

- [ ] Pipeline is trigger-driven (scheduled cron and/or code-change trigger), not
      manually invoked.
- [ ] Gate 1 (data validation) runs before any training compute is spent: row-count
      check + schema check (Pandera or equivalent) + null checks.
- [ ] Gate 1b (distribution/drift check — KS-test or PSI) runs against a versioned
      baseline, with a documented, owned threshold.
- [ ] Gate 2 (model evaluation) compares the candidate against a versioned baseline
      metric before registry promotion, not after.
- [ ] Gate 3 (response/prompt quality), for LLM systems, runs against a fixed, versioned
      golden eval set before any prompt/model/retrieval-config change is promoted.
- [ ] Every gate emits structured (JSON) logs and exits non-zero on failure — verified
      by an explicit test of the failure path, not just the happy path.
- [ ] A gate failure leaves the previous model/prompt version still serving — verified,
      not assumed.
- [ ] Pipeline runs are idempotent (dataset/content hash checked) so retries after
      transient failures are safe.
- [ ] Post-deployment monitoring reuses the same drift/quality metric computations as
      the pre-deploy gates, rather than being a separate, unrelated system.
- [ ] Thresholds (baseline AUC, PSI cutoff, coherence score minimum) have a named owner
      and a review cadence — they are not "set once and forgotten."
- [ ] CI/CD credentials used by validation/eval scripts are scoped narrowly (read-only
      where possible); promotion to serving is a distinct, more tightly controlled step.

---

## 9. Further Reading

Full citations, official documentation links, papers, and engineering blog posts backing
every claim in this module are in this same folder:

- **`references.md`** — official docs (Pandera, MLflow, GitHub Actions, SciPy, Evidently
  AI, Google Cloud MLOps whitepaper), papers (CD4ML), and engineering blog posts (Uber
  Michelangelo, Netflix Metaflow) with what each teaches and difficulty/reading-time
  estimates.
- **`videos.md`** — supplementary video resources for this module's topics.
- **`books.md`** — book chapters/recommendations for deeper theoretical grounding.
- **`github.md`** — hands-on repositories (`GokuMohandas/mlops-course`, `iterative/cml`,
  `unionai-oss/pandera`, `evidentlyai/evidently`, `mlflow/mlflow`, and others) to clone
  and run the patterns from this module yourself.
