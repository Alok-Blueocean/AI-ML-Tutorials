# Module 13 — Experiment Tracking and MLflow Deep Dive

> Module family: LLMOps/MLOps production engineering. Builds directly on Module 02 (CI/CD foundations), Module 03 (versioning, registries, rollback), Module 04 (reproducibility/environments), Module 05 (Docker), Module 06 (Kubernetes), and Modules 07–12 (prompt engineering, latency/cost, evaluation datasets, LLM-as-judge, deployment quality gates). This module assumes you can already build a container image, stand up a k8s deployment, and design a CI pipeline — we will not re-teach those.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain why "just running experiments" is insufficient in production AI, and what a disciplined experiment-tracking practice adds.
2. Design privacy-safe telemetry for LLM inference — what to log, what never to log, and how to keep debuggability without leaking PII.
3. Use MLflow's four components — **Tracking**, **Projects**, **Models**, **Model Registry** — as an integrated system rather than four disconnected features.
4. Write production-grade Python code that logs parameters, metrics, and artifacts; compares runs programmatically; and promotes a model through the registry using the modern **alias**-based pattern (not the deprecated Staging/Production stage model).
5. Build a tagging strategy (model, prompt version, dataset version, evaluator) that makes hundreds of experiments searchable and reproducible.
6. Apply Pareto-style, multi-metric comparison (quality vs. latency vs. cost) instead of chasing a single top score, using both MLflow's query API and Weights & Biases' parallel-coordinates view.
7. Track live inference metrics (TTFT, P95/P99 latency, cost per query, error rate, cache-hit rate) and wire them to alerting thresholds that drive real operational action.
8. Compare MLflow, Weights & Biases, and Comet across purpose, cost model, and production fit, and make a defensible tool choice for a given org.

### Prerequisites

- Comfortable with Python, virtual environments, and reading/writing YAML and JSON.
- Understands what a "model registry" and "artifact" are in the general versioning sense (Module 03).
- Has built and pushed a Docker image and can read a Kubernetes Deployment/Service manifest (Modules 05–06).
- Understands prompt versioning and offline evaluation datasets/metrics (Modules 07, 09, 10) — this module assumes you already know *how* to score a model/prompt combination and focuses on *tracking and comparing* those scores at scale.

### Key Terminology

| Term | Definition |
|---|---|
| **Run** | A single tracked execution of training, fine-tuning, evaluation, or an inference batch — the atomic unit MLflow logs against. |
| **Experiment** | A named, logical grouping of runs (e.g., "summarization-prompt-v2-eval"). |
| **Artifact** | Any file produced by a run and stored alongside it — a model file, a confusion matrix PNG, a JSONL of eval outputs, a config snapshot. |
| **Parameter (param)** | An input to a run, logged once and treated as immutable for that run (learning rate, prompt version, temperature, model checkpoint id). |
| **Metric** | A measured output of a run, optionally logged as a time series (loss per step, or a single scalar like `eval_f1`). |
| **Tag** | Free-form key/value metadata attached to a run or a registered model, used for filtering/search rather than for reproducibility math. |
| **Flavor (MLflow Models)** | A standard, tool-agnostic way of packaging a model (e.g., `sklearn`, `pytorch`, `pyfunc`) so any downstream tool can load it without knowing the training framework. |
| **Model Registry** | A versioned, governed catalog of registered models, separate from the free-for-all experiment tracking store — this is where promotion, staging, and rollback live. |
| **Alias** (MLflow 3.x pattern) | A mutable named pointer (e.g., `champion`, `shadow`, `canary`) to one specific model version — the modern replacement for the deprecated `Staging`/`Production`/`Archived` stage labels. |
| **TTFT** | Time-to-first-token — the latency from request start to the first streamed token, the dominant driver of perceived responsiveness for LLM apps. |
| **P95 / P99 latency** | The 95th/99th percentile of a latency distribution — the tail-latency numbers that matter far more than the mean for user experience. |
| **PII** | Personally identifiable information — anything that can identify a specific individual (raw prompt text, email, username, session content, etc.). |
| **Pareto frontier** | The set of runs where no metric can be improved without making another metric worse — the correct mental model for "quality vs. latency vs. cost" trade-off analysis. |

---

## 2. Why This Topic Matters, and Where It Fits in the Lifecycle

Every prior module in this course has assumed you can *produce* a version of something — a model checkpoint (Module 03), a container image (Module 05), a prompt template (Module 07), an evaluation score (Modules 09–11), a deployment gate decision (Module 12). Experiment tracking is the connective tissue that makes all of those versions *comparable* and *governable* over time. Without it, you get what almost every ML team eventually calls "spreadsheet hell": a shared Google Sheet of run names, hyperparameters typed in by hand, screenshots of eval numbers pasted from a terminal, and no reliable way to answer "which exact combination of model + prompt + dataset produced this metric six weeks ago?"

In the LLMOps era specifically, the problem gets worse, not better, for three reasons:

1. **The search space exploded.** Classical ML experiments varied hyperparameters (learning rate, batch size, regularization). LLM-application experiments vary model *and* prompt *and* retrieval config *and* temperature *and* dataset version simultaneously — the transcript example of "20 experiments across two models, five prompt variants, two temperatures" is a small, realistic slice of what a single sprint might produce.
2. **Quality is multi-dimensional and probabilistic.** A single F1 or accuracy number is no longer sufficient — you also need latency, cost per call, safety/PII flags, and often an LLM-judge score with its own noise characteristics (Module 11). Comparing runs on one axis while ignoring the others is how teams ship a model that scores best on paper and then blows the latency SLA in production.
3. **Production behavior must be watched continuously, not just measured once offline.** Offline evaluation (Modules 09–11) tells you how a candidate *should* behave. Experiment tracking extended into production — inference metrics, dashboards, alerts — tells you how it *actually* behaves under real traffic, which is where cost overruns, cache regressions, and silent quality drift actually surface.

Where this sits in the end-to-end lifecycle:

```
 Module 07-08          Module 09-11             THIS MODULE (13)              Module 12            Module 14
 Prompt/Model    -->   Offline Eval &     -->   Experiment Tracking   -->   Deployment Gate  -->  Production
 Candidates            LLM-as-Judge             (log, compare, tag,        (pass/fail on         Observability
                        Scoring                  promote via registry)      thresholds)           (OTel, dashboards,
                                                                                                    alerting)
```

Experiment tracking is not a side tool bolted onto training scripts — it is the system of record that Module 12's deployment gates read from, and it is the system that Module 14's production monitoring feeds *back into* (a production incident should trace back to an exact run, exact prompt version, and exact model version). Get this module wrong and every later module in the course inherits an untrustworthy foundation: your rollback story (Module 03) has nothing reliable to roll back *to*, and your dashboards (Module 12/14) have no historical baseline to compare against.

---

## 3. Main Concepts

### 3.1 Privacy-Safe Telemetry for LLM Inference

#### Theory

**What problem does this solve?** Production LLM systems need enough logged detail to debug a bad response, measure cost/latency, and trace lineage back to the exact model+prompt version that produced it. But LLM inputs and outputs are exactly the kind of free-text content most likely to contain PII — a user's email pasted into a support chatbot, a patient's symptoms in a health assistant, a customer's account number in a billing bot. The naive approach ("just log the whole request/response for debugging") is a data-protection and compliance liability (GDPR, CCPA, HIPAA where applicable) waiting to happen, and it also tends to bloat storage with content nobody will ever safely query.

**The core design principle:** default to *identifiers and metrics*, never to *raw content*, unless an explicit policy decision says otherwise for a specific, access-controlled use case (e.g., a human-review queue with strict RBAC and short retention).

**Trade-offs / when to relax the default:**

| Situation | Recommended stance |
|---|---|
| Standard production inference logging | IDs + metrics only, never raw text |
| Active incident investigation, human-in-the-loop review queue | Raw text may be surfaced, but behind extra authorization, short TTL, and its own audit trail — not the default telemetry pipeline |
| Regulated domain (health, finance) | Even metadata fields (e.g., "category") need care — a category field itself can be sensitive; run this past legal/compliance, don't assume engineering judgment is sufficient |
| Internal developer tooling / non-production sandbox | More logging latitude, but keep the habit consistent so it doesn't leak into prod code paths |

**What NOT to log by default:** raw query text, raw model responses, usernames, emails, session tokens, IP addresses in plaintext, or anything else that directly or indirectly identifies a person. If you must correlate events to the *same* user without knowing *who* they are, hash the identifier.

#### Architecture

```
                     Inference Request
                            |
                            v
              +--------------------------+
              |   Inference Service       |
              |  (FastAPI / gateway)      |
              +--------------------------+
                     |            |
          (1) serve   |            | (2) emit telemetry event
          response    |            v
                     |     +----------------------+
                     |     |  Telemetry Sanitizer |
                     |     |  - hash(user_id)      |
                     |     |  - strip raw text     |
                     |     |  - derive: length,    |
                     |     |    category, pii_flag |
                     |     +----------------------+
                     |                |
                     v                v
              End User          +--------------------------+
                                 |  telemetry.json event    |
                                 |  request_id, user_hash,  |
                                 |  model_version,          |
                                 |  prompt_version,         |
                                 |  tokens_in/out, cost,    |
                                 |  latency_ms, ttft_ms,    |
                                 |  pii_detected (bool),    |
                                 |  trace_id                |
                                 +--------------------------+
                                          |
                                          v
                          +-------------------------------+
                          | Encrypted S3 (30-day rolling   |
                          | retention, PII-minimized)      |
                          +-------------------------------+
                                          |
                                          v
                     +---------------------------------------+
                     | Downstream: cost dashboards, alerting, |
                     | experiment/run correlation (trace_id)  |
                     +---------------------------------------+
```

Notice the sanitizer sits *between* the serving path and the storage sink — it is not optional or "applied later." Retrofitting privacy onto an already-collected raw-text log store is far riskier (and often legally required to be reported as a breach window) than designing the sanitizer in from day one.

#### Examples

**Beginner** — a naive, unsafe log line many teams start with by accident:

```python
# DO NOT DO THIS
logger.info(f"user={user_email} prompt={raw_prompt} response={raw_response}")
```

This logs PII directly into application logs, which are typically retained far longer than 30 days, shipped to third-party log aggregators, and readable by a broad set of engineers — a textbook compliance failure.

**Intermediate** — a safer, structured telemetry event:

```python
import hashlib
import json
import time
import uuid

def hash_user_id(raw_user_id: str, salt: str) -> str:
    return hashlib.sha256(f"{salt}:{raw_user_id}".encode()).hexdigest()

def build_telemetry_event(
    raw_user_id: str,
    prompt_text: str,
    response_text: str,
    model_version: str,
    prompt_version: str,
    tokens_in: int,
    tokens_out: int,
    cost_usd: float,
    latency_ms: float,
    ttft_ms: float,
    pii_detector,  # a lightweight classifier/regex bundle, not the raw text
    salt: str,
) -> dict:
    return {
        "request_id": str(uuid.uuid4()),
        "trace_id": str(uuid.uuid4()),
        "user_id_hash": hash_user_id(raw_user_id, salt),
        "model_version": model_version,
        "prompt_version": prompt_version,
        "query_length": len(prompt_text),
        "query_category": classify_category(prompt_text),  # e.g. "billing", "support"
        "pii_detected": pii_detector.contains_pii(prompt_text),
        "tokens_in": tokens_in,
        "tokens_out": tokens_out,
        "cost_usd": round(cost_usd, 6),
        "latency_ms": latency_ms,
        "ttft_ms": ttft_ms,
        "timestamp": time.time(),
    }
```

Notice: `prompt_text` and `response_text` are *inputs* to derive safe fields (`query_length`, `query_category`, `pii_detected`) — they are never themselves written to the output dict. This is the pattern to internalize: raw content flows through short-lived memory only, never to the sink.

**Production-grade** — telemetry wired into an MLflow run so the same trace_id links an offline eval run to the production inference event that inspired it:

```python
import mlflow

def log_inference_event_to_mlflow(event: dict, experiment_name: str = "prod-inference-telemetry"):
    mlflow.set_experiment(experiment_name)
    with mlflow.start_run(run_name=event["request_id"]):
        mlflow.log_params({
            "model_version": event["model_version"],
            "prompt_version": event["prompt_version"],
        })
        mlflow.log_metrics({
            "tokens_in": event["tokens_in"],
            "tokens_out": event["tokens_out"],
            "cost_usd": event["cost_usd"],
            "latency_ms": event["latency_ms"],
            "ttft_ms": event["ttft_ms"],
            "pii_detected": int(event["pii_detected"]),
        })
        mlflow.set_tags({
            "trace_id": event["trace_id"],
            "user_id_hash": event["user_id_hash"],
            "query_category": event["query_category"],
        })
```

In practice, most teams do **not** create one MLflow run per production inference call at high QPS (the tracking server is not built for that write volume) — instead, raw events go to a time-series/log store (encrypted S3 + something like ClickHouse/BigQuery, covered more in Module 14), and only *aggregated* windows or *sampled* events get mirrored into MLflow for correlation with offline experiments. The snippet above is the right shape at low-to-moderate volume or for a sampled subset; at scale, batch-aggregate before writing to MLflow.

#### Comparison: what belongs in each storage tier

| Data | Where it lives | Retention | Access |
|---|---|---|---|
| Raw prompt/response text | Nowhere, by default | N/A | N/A unless explicit reviewed policy |
| Hashed user id, token counts, cost, latency, tags | Encrypted S3 / telemetry store | 30-day rolling (example policy) | Engineering, on-call |
| Experiment params/metrics/artifacts | MLflow tracking store | Long-term (this *is* your audit trail) | ML engineers, MLOps |
| Registered model + promotion history | MLflow Model Registry | Indefinite (versioned) | Gated — promotion requires review |

---

### 3.2 MLflow Tracking

#### Theory

MLflow Tracking answers one question reliably: *"What exactly did I run, and what came out of it?"* It solves the "spreadsheet hell" problem by giving every run a durable, queryable record of its parameters, metrics, artifacts, source code version, and environment — with zero required backend infrastructure to get started (a local `mlruns/` directory is a complete, if not production-scale, tracking store).

**Problems it solves:**
- Reproducibility gap between "I remember roughly what I ran" and "here is the exact recorded configuration."
- Comparison across dozens/hundreds of runs without manual spreadsheet transcription.
- A stable API surface (`mlflow.log_param`, `mlflow.log_metric`, `mlflow.log_artifact`) that works whether the backend is a local folder, a Postgres-backed team server, or a managed Databricks workspace.

**When to use:** any time you're iterating on model, prompt, or pipeline configuration and need the results to be comparable later — which in an LLMOps context includes prompt-engineering iteration, RAG configuration sweeps, and evaluation-harness runs, not just classical model training.

**When NOT to use as-is:** as noted above, do not point MLflow Tracking directly at raw high-QPS production inference traffic — it is an experiment/run store, not a metrics time-series database. Use Prometheus/OpenTelemetry (Module 14) for that, and use MLflow for the *experiment* layer that produced the deployed candidate.

**Key trade-off — local file store vs. remote tracking server:**

| | Local file store (`mlruns/`) | Remote tracking server (Postgres/MySQL backend + S3/GCS artifact store) |
|---|---|---|
| Setup effort | Zero — default behavior | Requires standing up a server + DB + object storage (Docker/K8s from Modules 05–06) |
| Team collaboration | Poor — each person has their own local runs | Good — shared source of truth, queryable by everyone |
| Scale | Fine for solo experimentation | Required once run volume or team size grows |
| CI/CD integration | Awkward (ephemeral CI runners lose local state) | Natural — CI jobs log to the same shared server |

#### Architecture

```
                         +-----------------------------+
                         |      MLflow Tracking API      |
                         |  mlflow.start_run()           |
                         |  mlflow.log_param/metric/     |
                         |  artifact/set_tag             |
                         +---------------+---------------+
                                         |
                     +-------------------+-------------------+
                     |                                       |
                     v                                       v
        +---------------------------+          +---------------------------+
        |  Backend Store              |          |  Artifact Store            |
        |  (params, metrics, tags,    |          |  (model files, plots,      |
        |   run metadata)             |          |   datasets, JSONL evals)   |
        |  - local: mlruns/           |          |  - local: mlruns/.../      |
        |  - prod: Postgres/MySQL     |          |    artifacts/              |
        +---------------------------+          |  - prod: S3 / GCS / Azure  |
                                                  |    Blob                    |
                                                  +---------------------------+
                                         ^
                                         |
                         +---------------+---------------+
                         |     MLflow Tracking Server     |
                         |  (HTTP API + UI, containerized |
                         |   per Module 05/06 patterns)   |
                         +---------------------------------+
                                         ^
              +--------------------------+--------------------------+
              |                          |                            |
    +------------------+     +------------------------+    +------------------------+
    |  Data scientist    |     |  CI/CD pipeline         |    |  Batch eval harness    |
    |  local notebook    |     |  (Module 02 patterns)   |    |  (Modules 09-11)       |
    +------------------+     +------------------------+    +------------------------+
```

#### Examples

**Beginner** — logging a single run:

```python
import mlflow

mlflow.set_tracking_uri("http://mlflow.internal:5000")  # or omit for local mlruns/
mlflow.set_experiment("support-bot-prompt-eval")

with mlflow.start_run(run_name="gpt4o-prompt-v3-temp0.2"):
    mlflow.log_param("model", "gpt-4o")
    mlflow.log_param("prompt_version", "v3")
    mlflow.log_param("temperature", 0.2)

    # ... run the evaluation harness from Module 09/10 ...
    correctness = 0.91
    p95_latency_ms = 640
    cost_per_call_usd = 0.0032

    mlflow.log_metric("correctness", correctness)
    mlflow.log_metric("p95_latency_ms", p95_latency_ms)
    mlflow.log_metric("cost_per_call_usd", cost_per_call_usd)
```

**Intermediate** — logging a full evaluation sweep with tags and artifacts:

```python
import itertools
import json
import mlflow

models = ["gpt-4o", "claude-sonnet-4.5"]
prompt_versions = ["v1", "v2", "v3", "v4", "v5"]
temperatures = [0.0, 0.7]

mlflow.set_experiment("support-bot-sweep-2026-07")

for model, prompt_version, temperature in itertools.product(models, prompt_versions, temperatures):
    run_name = f"{model}-{prompt_version}-t{temperature}"
    with mlflow.start_run(run_name=run_name):
        mlflow.set_tags({
            "model": model,
            "prompt_version": prompt_version,
            "dataset_version": "eval-set-v7-frozen",
            "evaluator": "claude-sonnet-4.5-judge-v2",
        })
        mlflow.log_params({
            "model": model,
            "prompt_version": prompt_version,
            "temperature": temperature,
        })

        results = run_eval_harness(model, prompt_version, temperature)  # from Module 09-11

        mlflow.log_metrics({
            "correctness": results["correctness"],
            "composite_score": results["composite_score"],
            "p95_latency_ms": results["p95_latency_ms"],
            "cost_per_call_usd": results["cost_per_call_usd"],
        })

        with open("/tmp/eval_outputs.jsonl", "w") as f:
            for row in results["raw_outputs"]:
                f.write(json.dumps(row) + "\n")
        mlflow.log_artifact("/tmp/eval_outputs.jsonl", artifact_path="eval_outputs")
```

This is the direct code-form of the transcript's "20 experiments across two models, five prompt variants, two temperatures" example — 20 runs (2 × 5 × 2), each fully tagged and comparable.

**Production-grade** — CI-triggered tracking run (ties Module 02's CI/CD patterns to MLflow):

```yaml
# .github/workflows/eval-on-pr.yml
name: Evaluate prompt/model candidate on PR
on:
  pull_request:
    paths:
      - "prompts/**"
      - "eval/**"

jobs:
  run-eval-sweep:
    runs-on: ubuntu-latest
    env:
      MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
      MLFLOW_TRACKING_TOKEN: ${{ secrets.MLFLOW_TRACKING_TOKEN }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - name: Run evaluation sweep and log to MLflow
        run: python scripts/run_sweep_and_log.py --pr-sha ${{ github.sha }}
      - name: Fail build if no candidate meets promotion bar
        run: python scripts/check_promotion_gate.py --pr-sha ${{ github.sha }}
```

This is exactly the deployment-quality-gate wiring from Module 12, with MLflow as the system that both *produces* the comparison data and *records* the gate decision as a run tag.

---

### 3.3 MLflow Projects

#### Theory

MLflow Projects solves a narrower but important problem: **"how do I package a run so that anyone — a teammate, a CI runner, a Kubernetes job — can execute it with the exact same environment and entry point, without me explaining it verbally?"** It is the reproducibility layer (directly extending Module 04) applied specifically to the unit of "one experiment run."

A project is any directory containing code, optionally with an `MLproject` file describing:
- **Entry points** — named commands (e.g., `main`, `evaluate`) with typed parameters.
- **Environment** — one of four managers: `virtualenv`, `conda`, Docker (reusing Module 05 images), or `system` (use whatever's already installed — least reproducible, fastest to iterate).

**When to use:** whenever a run needs to be reliably re-executed by someone/something other than the original author — CI pipelines, scheduled retraining, remote execution on Databricks/Kubernetes clusters.

**When NOT to use:** quick, throwaway local exploration where the overhead of writing an `MLproject` file outweighs the benefit — plain scripts logging via the Tracking API are fine until the run needs to travel.

#### Example — MLproject file and Docker-backed execution

```yaml
# MLproject
name: support-bot-eval

docker_env:
  image: registry.internal/llmops/eval-harness:2026.07.1

entry_points:
  main:
    parameters:
      model: {type: string, default: "gpt-4o"}
      prompt_version: {type: string, default: "v3"}
      temperature: {type: float, default: 0.2}
      dataset_version: {type: string, default: "eval-set-v7-frozen"}
    command: >
      python run_eval.py
      --model {model}
      --prompt-version {prompt_version}
      --temperature {temperature}
      --dataset-version {dataset_version}
```

```bash
mlflow run . \
  -P model=claude-sonnet-4.5 \
  -P prompt_version=v4 \
  -P temperature=0.0 \
  --experiment-name support-bot-sweep-2026-07
```

Because the environment is the exact Docker image built in your Module 05 CI pipeline, this run is bit-for-bit reproducible on a teammate's laptop, in CI, or on a Kubernetes Job — the same guarantee Module 04 taught for training environments, now scoped to the unit of a single MLflow run.

---

### 3.4 MLflow Models

#### Theory

MLflow Models solves the interoperability problem: a model trained with scikit-learn, one fine-tuned with PyTorch, and a wrapped third-party LLM API call are three completely different Python objects with different serialization needs — yet a deployment tool (a serving container, a batch job, a Kubernetes inference service) shouldn't need three different loading code paths. MLflow's answer is the **`MLmodel`** file plus the **flavor** abstraction: every model is saved with at least one standard flavor (`python_function`/`pyfunc` is the universal fallback flavor every other flavor also implements), so `mlflow.pyfunc.load_model(uri)` works identically regardless of what produced the model.

**What it gives you beyond a raw pickle/checkpoint file:**
- A **model signature** — the expected input/output schema, usually inferred automatically from an example input, which catches shape/dtype mismatches before they hit production.
- An auto-captured `conda.yaml`/`requirements.txt` snapshot of the exact library versions used — closing the loop with Module 04's reproducibility guarantees at the model-artifact level.
- A uniform loading API across 20+ built-in flavors (`sklearn`, `pytorch`, `tensorflow`, `xgboost`, `transformers`, generic `pyfunc` for anything else, including a wrapped LLM API call).

**When NOT to rely on it alone:** MLflow Models packages the artifact and its loading contract; it does not replace your serving infrastructure (Module 05/06) or your inference-time observability (Module 14) — it's the "what to load and how" layer, not the "how to scale replicas" layer.

#### Example — wrapping an LLM-API-backed "model" as a `pyfunc` for uniform tracking

```python
import mlflow
import mlflow.pyfunc

class PromptedLLMModel(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        self.prompt_template = open(context.artifacts["prompt_template"]).read()

    def predict(self, context, model_input):
        prompts = [self.prompt_template.format(query=q) for q in model_input["query"]]
        return [call_llm_api(p, model="gpt-4o", temperature=0.0) for p in prompts]

with mlflow.start_run(run_name="wrap-prompted-model-v3"):
    mlflow.pyfunc.log_model(
        artifact_path="model",
        python_model=PromptedLLMModel(),
        artifacts={"prompt_template": "prompts/support_bot_v3.txt"},
        pip_requirements=["openai==1.*"],
        registered_model_name="support-bot-responder",  # registers in one call
    )
```

Even though there is no traditional "training," wrapping the prompt + LLM call as a `pyfunc` model means it can be registered, versioned, promoted, and loaded through the exact same registry mechanics as a classical scikit-learn model — a genuinely useful trick for LLMOps teams standardizing tooling across both classical ML and LLM-based components.

---

### 3.5 MLflow Model Registry — Promotion via Tags and Aliases (Mid-2026 Pattern)

#### Theory

The Model Registry is the **governance layer** sitting above raw experiment tracking: it answers "which specific model version is the one currently serving traffic, and how did it get promoted?" — distinct from "which of my 200 experiment runs looked good?"

This is the piece where the 2026 workflow has genuinely changed since most 2023–2024 tutorials (and the 2021 "Machine Learning Engineering with MLflow" book) were written. The old pattern moved a model version through fixed **stages**: `None → Staging → Production → Archived`. That API still exists for backward compatibility but is deprecated in current MLflow guidance. The current recommended pattern is:

- **Tags** — arbitrary key/value metadata on a registered model or a specific version (`validated_by: "eval-pipeline-v7"`, `owner: "support-bot-team"`).
- **Aliases** — a small number of named, mutable pointers per registered model (commonly `champion` for the currently-serving version, `challenger`/`shadow` for a candidate being shadow-tested, `canary` for a partial-traffic rollout). Reassigning an alias is a single API call (`set_registered_model_alias`), and production code resolves the alias at load time (`models:/support-bot-responder@champion`) instead of hard-coding a version number.

**Why this is better than stage transitions:** stages were a single global label per version, which didn't map well onto real promotion workflows where you often want a version to be simultaneously "shadow-tested" and "tagged for the EU region" and "owned by team X" — multiple overlapping facts, not one linear state. Aliases + tags model that correctly: a version can hold several tags and (in principle) be pointed to by more than one alias if your workflow calls for it, whereas legacy stages forced an artificial single-state machine.

**When to still know about stages:** you will encounter them reading older codebases, blog posts, and possibly the aforementioned 2021 book — recognize the pattern (`client.transition_model_version_stage(...)`) as legacy, and if you inherit a codebase using it, plan a migration to aliases rather than extending it further.

#### Architecture

```
                     +-------------------------------------------+
                     |            Model Registry                  |
                     |  Registered Model: "support-bot-responder"  |
                     |                                             |
                     |  Version 12  tags: {eval_score: 0.87}        |
                     |  Version 13  tags: {eval_score: 0.91}  <--@champion
                     |  Version 14  tags: {eval_score: 0.93}  <--@challenger (shadow traffic)
                     +-------------------------------------------+
                                     ^
                                     |  register_model() after promotion gate passes
                                     |
                     +-------------------------------------------+
                     |     Experiment Tracking (many runs)         |
                     |  20 runs -> filter/sort -> top-3 candidates |
                     +-------------------------------------------+
                                     ^
                                     |  log_metric / log_param / set_tag per run
                                     |
                     +-------------------------------------------+
                     |     Evaluation sweep (Modules 09-11)        |
                     +-------------------------------------------+

  Production serving resolves:  models:/support-bot-responder@champion
  Promotion event:  client.set_registered_model_alias("support-bot-responder", "champion", version=14)
  Rollback event:   client.set_registered_model_alias("support-bot-responder", "champion", version=13)
```

Rollback (Module 03's core concept) becomes a one-line alias reassignment back to the previous version — no redeploy of code, only a registry pointer change, provided your serving layer resolves the alias at request time or on a short polling interval rather than baking a version number into a deployed image.

#### Code — end-to-end: log, compare, promote

```python
import mlflow
from mlflow import MlflowClient

client = MlflowClient(tracking_uri="http://mlflow.internal:5000")
MODEL_NAME = "support-bot-responder"

# 1. Register the best candidate run's model artifact as a new version
best_run_id = "a1b2c3d4e5f6"  # from the comparison step below
model_uri = f"runs:/{best_run_id}/model"
new_version = mlflow.register_model(model_uri=model_uri, name=MODEL_NAME)

# 2. Tag the version with governance metadata
client.set_model_version_tag(
    name=MODEL_NAME,
    version=new_version.version,
    key="validated_by",
    value="eval-pipeline-v7",
)
client.set_model_version_tag(
    name=MODEL_NAME,
    version=new_version.version,
    key="promotion_date",
    value="2026-07-30",
)

# 3. Promote by reassigning the "champion" alias (NOT a stage transition)
client.set_registered_model_alias(
    name=MODEL_NAME,
    alias="champion",
    version=new_version.version,
)

# 4. Production code always resolves via the alias
production_model = mlflow.pyfunc.load_model(f"models:/{MODEL_NAME}@champion")

# 5. Rollback is symmetric: point the alias back at a known-good version
def rollback(model_name: str, previous_version: int):
    client.set_registered_model_alias(model_name, "champion", previous_version)
```

---

### 3.6 Comparing Experiments Across Versions

#### Theory

This is the transcript's core practical lesson, and it deserves to be treated as a discipline, not an afterthought. The failure mode it prevents: a team runs many experiments, eyeballs a leaderboard sorted by one metric, and promotes the top row — without noticing that row also happens to have a P99 latency well outside the SLA, or that it was evaluated against a slightly different (non-frozen) dataset snapshot than its neighbors, making the "win" meaningless.

**The four non-negotiable rules (directly from the transcript, expanded):**

1. **Frozen evaluation dataset.** If the dataset changes between runs, the comparison is scientifically invalid — you are no longer holding the independent variable constant. Tag every run with a `dataset_version` and treat any change to that dataset as requiring a *full re-run* of all candidates you want to compare, not an incremental patch.
2. **Tag everything** — `model`, `prompt_version`, `dataset_version`, `evaluator` (which judge model/rubric scored the run — Module 11 material). Untagged runs are technically logged but practically unusable at scale — you'll have data but no way to slice it.
3. **Use Pareto / multi-metric analysis, not top-1 sorting.** A single "best" run by one metric is often dominated on another axis. Visual tools (W&B parallel coordinates) and explicit multi-condition filters (MLflow search) both exist to surface this.
4. **Export for statistical rigor.** A composite score's difference between rank 1 and rank 3 might not be statistically significant given evaluation noise (Module 09's statistical evaluation content is the prerequisite here) — exporting the comparison to a DataFrame lets you run paired t-tests or bootstrap confidence intervals before treating a ranking as real.

#### Architecture — the comparison funnel

```
   20 raw runs (2 models x 5 prompt variants x 2 temperatures)
                       |
                       v
   Filter: tags.model == "gpt-4o" AND metrics.correctness > 0.9
                       |
                       v
              ~6-8 runs survive
                       |
                       v
   Sort: composite_score DESC, p95_latency_ms ASC
                       |
                       v
              Top 3 candidates
                       |
                       v
   Export to DataFrame -> paired significance test vs. current champion
                       |
                       v
   Human review: cost analysis + qualitative spot-check of outputs
                       |
                       v
        Promotion decision (register_model + set_registered_model_alias)
```

#### Examples

**Beginner** — filtering and sorting with the MLflow search API:

```python
import mlflow

runs_df = mlflow.search_runs(
    experiment_names=["support-bot-sweep-2026-07"],
    filter_string="tags.model = 'gpt-4o' and metrics.correctness > 0.9",
    order_by=["metrics.composite_score DESC", "metrics.p95_latency_ms ASC"],
)
print(runs_df[["tags.mlflow.runName", "metrics.composite_score", "metrics.p95_latency_ms"]].head(3))
```

**Intermediate** — the `compare_runs.py` pattern from the transcript, expanded into working code:

```python
# compare_runs.py
import mlflow

EXPERIMENT_NAME = "support-bot-sweep-2026-07"
F1_THRESHOLD = 0.84
LATENCY_BUDGET_MS = 900

def compare_and_select():
    runs_df = mlflow.search_runs(experiment_names=[EXPERIMENT_NAME])

    for _, row in runs_df.iterrows():
        print(
            f"{row['tags.mlflow.runName']:35s} "
            f"f1={row['metrics.f1']:.3f}  "
            f"latency_ms={row['metrics.p95_latency_ms']:.0f}"
        )

    best_run = runs_df.sort_values("metrics.f1", ascending=False).iloc[0]

    meets_quality = best_run["metrics.f1"] >= F1_THRESHOLD
    meets_latency = best_run["metrics.p95_latency_ms"] <= LATENCY_BUDGET_MS

    if meets_quality and meets_latency:
        print(f"PROMOTE: {best_run['tags.mlflow.runName']} "
              f"(f1={best_run['metrics.f1']:.3f}, "
              f"latency={best_run['metrics.p95_latency_ms']:.0f}ms)")
        return best_run["run_id"]
    else:
        print("NO PROMOTION: best-scoring run fails a promotion gate "
              "(quality-only winner is not automatically deployable).")
        return None

if __name__ == "__main__":
    compare_and_select()
```

This is the exact production lesson from the transcript encoded as code: **the highest score does not automatically win** if it breaks a latency (or cost, or safety) budget. This is the same principle as Module 12's deployment quality gates, applied one level earlier — at experiment-selection time rather than at deploy-time.

**Production-grade** — Pareto-aware selection with statistical significance check:

```python
import mlflow
import numpy as np
from scipy import stats

def select_pareto_candidates(experiment_name: str, quality_metric: str, latency_metric: str, cost_metric: str):
    df = mlflow.search_runs(experiment_names=[experiment_name])
    df = df.dropna(subset=[f"metrics.{quality_metric}", f"metrics.{latency_metric}", f"metrics.{cost_metric}"])

    is_dominated = []
    for i, row in df.iterrows():
        dominated = False
        for j, other in df.iterrows():
            if i == j:
                continue
            better_or_equal_all = (
                other[f"metrics.{quality_metric}"] >= row[f"metrics.{quality_metric}"]
                and other[f"metrics.{latency_metric}"] <= row[f"metrics.{latency_metric}"]
                and other[f"metrics.{cost_metric}"] <= row[f"metrics.{cost_metric}"]
            )
            strictly_better_one = (
                other[f"metrics.{quality_metric}"] > row[f"metrics.{quality_metric}"]
                or other[f"metrics.{latency_metric}"] < row[f"metrics.{latency_metric}"]
                or other[f"metrics.{cost_metric}"] < row[f"metrics.{cost_metric}"]
            )
            if better_or_equal_all and strictly_better_one:
                dominated = True
                break
        is_dominated.append(dominated)

    pareto_frontier = df[~np.array(is_dominated)]
    return pareto_frontier.sort_values(f"metrics.{quality_metric}", ascending=False)

def significance_vs_champion(challenger_scores: list[float], champion_scores: list[float]) -> dict:
    t_stat, p_value = stats.ttest_rel(challenger_scores, champion_scores)
    return {"t_stat": t_stat, "p_value": p_value, "significant_at_0.05": p_value < 0.05}
```

This is deliberately more rigorous than "pick the top row" — it surfaces the *whole* non-dominated frontier (the runs where no other run beats it on every axis simultaneously), which is exactly what W&B's parallel-coordinates view is designed to help a human visually spot, and what this code does programmatically for automated gating.

#### Weights & Biases — Parallel Coordinates for the Same Job

W&B's parallel-coordinates panel plots one vertical axis per metric/hyperparameter and one polyline per run, so a human can visually trace, e.g., "runs that kept `p95_latency_ms` low *and* `correctness` high tend to cluster around `prompt_version=v4` and `temperature=0.0`" — a pattern that is often faster to *spot* visually than to *query* programmatically, especially early in an investigation before you know which axes matter.

```python
import wandb

run = wandb.init(project="support-bot-sweep", name="gpt4o-v4-t0.0")
wandb.config.update({
    "model": "gpt-4o",
    "prompt_version": "v4",
    "temperature": 0.0,
    "dataset_version": "eval-set-v7-frozen",
})
wandb.log({
    "correctness": 0.93,
    "p95_latency_ms": 610,
    "cost_per_call_usd": 0.0029,
})
run.finish()
```

Then, in the W&B UI, a parallel-coordinates panel is configured over `config.temperature`, `config.prompt_version`, `correctness`, `p95_latency_ms`, and `cost_per_call_usd` — no extra logging code is needed beyond what you already log for tracking; the visualization is a UI-side view over existing run data.

**Practical guidance on which tool for which step:** many production teams use *both*, not as redundancy but as division of labor — MLflow as the queryable system of record + registry (because you likely want it self-hosted and integrated with your CI/CD and registry/rollback story), and W&B for the exploratory, visual, "let a human's eye find the pattern" step during active experimentation. This is not wasted effort; it's using each tool for what it does best. Comet's positioning is discussed in the comparison table below.

---

### 3.7 Tracking Live Inference Metrics

#### Theory

Offline experiment comparison (3.6) tells you which candidate to promote. Once it's serving real traffic, the tracking discipline shifts from "which run wins" to "is the system currently healthy, and if not, page someone." This is the bridge into Module 14's full observability stack — this module covers the *metric selection and alerting-threshold* discipline; Module 14 covers the OpenTelemetry/Prometheus/Grafana implementation depth.

**The small, action-driven metric set (per the transcript), and why each one earns its place:**

| Metric | Why it matters | Example threshold |
|---|---|---|
| TTFT (time to first token) | Dominant driver of *perceived* responsiveness in streaming UIs — a slow total time is more tolerable if the first token arrives fast | P95 > 1000ms sustained → alert |
| Total latency (P95/P99) | Tail latency is what causes visible user complaints; means hide the problem | P95 > budget for N consecutive windows → alert |
| Cost per query / hourly cost accumulation | LLM cost scales with traffic and token usage in ways that can runaway silently (e.g., a prompt regression that triples output tokens) | Daily budget alarm |
| Error rate (HTTP errors + eval/pass failures) | Direct reliability signal | > 0.5% → immediate investigation |
| Cache hit rate | A drop indicates either a traffic-pattern shift or a caching-layer regression — either way, it's both a cost and latency lever | < 20% → investigate opportunity |
| Fallback usage rate | Indicates the primary model/path is failing and traffic is silently degrading to a fallback — easy to miss because the user-facing error rate looks fine | > 10% → page on-call |

**Why "a small set, add more only when they change decisions" matters:** dashboards and alert rules accumulate metric sprawl over time, and past a certain point more metrics *reduce* signal-to-noise for the on-call engineer rather than increasing it. The discipline is to start minimal and add a new tracked metric only when you can name the specific decision or action it would have changed.

#### Architecture

```
             Production Inference Traffic
                         |
                         v
          +-------------------------------+
          |     Inference Service          |
          |  emits per-call metrics:       |
          |  ttft_ms, total_latency_ms,    |
          |  cost_usd, http_status,        |
          |  cache_hit (bool),             |
          |  used_fallback (bool)          |
          +-------------------------------+
                         |
                         v
          +-------------------------------+
          |   Metrics pipeline (W&B logging |
          |   or Prometheus - Module 14)    |
          +-------------------------------+
                 |         |          |
                 v         v          v
      +-----------+ +-----------+ +--------------+
      | 5-min P95  | | 1-hr cost | | Weekly summary|
      | latency    | | accumul.  | | cost trend,   |
      | rolling    | |           | | cache hit,    |
      | window     | |           | | error rate    |
      +-----------+ +-----------+ +--------------+
                 |         |          
                 v         v          
          +-------------------------------+
          |    Alerting rules engine       |
          |  ttft P95 > 1000ms (N windows)  |
          |    -> Slack alert               |
          |  error_rate > 0.5%              |
          |    -> immediate investigation   |
          |  fallback_rate > 10%            |
          |    -> page on-call              |
          +-------------------------------+
```

#### Code — the `MetricMonitor` pattern from the transcript, expanded

```python
# metric_monitor.py
from dataclasses import dataclass
from collections import deque
import time

@dataclass
class WindowMetrics:
    error_rate: float
    p95_latency_ms: float
    fallback_rate: float

class MetricMonitor:
    """Evaluates a rolling 5-minute window against action thresholds."""

    ERROR_RATE_THRESHOLD = 0.03
    P95_LATENCY_THRESHOLD_MS = 1000
    FALLBACK_RATE_THRESHOLD = 0.10

    def __init__(self, window_seconds: int = 300):
        self.window_seconds = window_seconds
        self.events = deque()

    def record(self, is_error: bool, latency_ms: float, used_fallback: bool):
        now = time.time()
        self.events.append((now, is_error, latency_ms, used_fallback))
        self._evict_old(now)

    def _evict_old(self, now: float):
        while self.events and now - self.events[0][0] > self.window_seconds:
            self.events.popleft()

    def compute_window(self) -> WindowMetrics | None:
        if not self.events:
            return None
        n = len(self.events)
        error_rate = sum(e[1] for e in self.events) / n
        latencies = sorted(e[2] for e in self.events)
        p95_latency_ms = latencies[int(0.95 * (n - 1))]
        fallback_rate = sum(e[3] for e in self.events) / n
        return WindowMetrics(error_rate, p95_latency_ms, fallback_rate)

    def check_and_alert(self, alert_fn, page_fn):
        window = self.compute_window()
        if window is None:
            return
        if window.error_rate > self.ERROR_RATE_THRESHOLD:
            alert_fn(f"Error spike: {window.error_rate:.1%} over last {self.window_seconds}s")
        if window.p95_latency_ms > self.P95_LATENCY_THRESHOLD_MS:
            alert_fn(f"Latency spike: P95={window.p95_latency_ms:.0f}ms")
        if window.fallback_rate > self.FALLBACK_RATE_THRESHOLD:
            page_fn(f"High fallback usage: {window.fallback_rate:.1%} — paging on-call")
```

The design point worth internalizing: **different signal types route to different response mechanisms** — a Slack alert for something worth a look this afternoon, a page for something that needs a human awake right now. Conflating all signals into one severity is a common early-stage mistake that either causes alert fatigue (everything pages) or missed incidents (nothing pages).

---

## 4. Real-World Case Studies (Reasoned Inference)

> These describe how organizations with this class of engineering problem would plausibly architect a solution, based on publicly known engineering-blog patterns and industry norms — not confirmed internal specifics of any company's current stack.

**A frontier AI lab shipping a chat product (pattern seen at organizations like OpenAI/Anthropic).** A lab operating at very high inference QPS would almost certainly not write one experiment-tracking run per production request — the volume would overwhelm any general-purpose tracking backend. A more plausible architecture: a high-throughput streaming metrics/telemetry pipeline (similar in shape to Module 14's OpenTelemetry patterns) for every request, with sampling and offline batch jobs periodically pulling representative slices into an experiment-tracking-style system for correlation with model/prompt version experiments. Given the deprecation of MLflow's stage-based promotion in favor of aliases industry-wide, and given the scale of A/B and canary testing such labs run on prompt and system-message changes, an alias-like abstraction (a mutable pointer such as "production" or "canary-10pct" resolved at request time) is a natural fit regardless of whether the specific backing store is MLflow or an internal equivalent.

**A large-scale recommendation/personalization platform (pattern seen at organizations like Netflix, Spotify, Amazon).** These companies popularized the idea of *many concurrent, small A/B experiments* rather than "one champion model" — their internal experimentation platforms (well-documented in public engineering blogs from Netflix and others) generalize the same tagging discipline this module teaches: every experiment is tagged with an experiment ID, a variant/arm, and the exact model/config version serving it, so that any observed metric shift in production can be attributed to a specific experiment arm. The Pareto-style multi-metric comparison this module teaches (quality vs. latency vs. cost) maps directly onto their standard practice of *guardrail metrics* — secondary metrics that must not regress even when the primary metric being optimized improves.

**A ride-sharing / marketplace platform running many ML models across teams (pattern seen at organizations like Uber).** Uber's public writing on internal ML platforms (e.g., Michelangelo-era engineering blogs) describes a centralized model registry with governance and promotion workflows spanning many independent teams' models — a structurally similar problem to MLflow's Model Registry at a larger, multi-tenant scale. A plausible current-day architecture layers a company-wide registry and lineage system on top of (or instead of) an off-the-shelf tool like MLflow once the number of teams and models crosses a threshold where self-hosted MLflow's single-tenant assumptions become limiting — but the *concepts* (tags, aliases/promotion pointers, frozen eval sets per model family) remain the same regardless of which system implements them.

**A data/AI platform vendor (Databricks).** Databricks is the natural case study for "MLflow at scale in a managed environment" because MLflow originated there and Databricks' managed offering wraps the same open-source Tracking/Registry APIs with workspace-level permissions, notebook lineage, and Unity Catalog integration for governance. Teams already on Databricks would reasonably default to the managed MLflow rather than self-hosting, trading some infrastructure control for tighter integration with their existing data platform's access control and lineage.

**An enterprise software analytics company evaluating build-vs-buy for its own MLOps stack (pattern relevant to a company like CAST Software, or any mid-size engineering org).** A mid-size platform engineering team choosing between MLflow, W&B, and Comet would typically weigh: self-hosting effort and existing Kubernetes footprint (Modules 05–06) against the value of a managed collaboration UI, current spend on GPU/compute (relevant given W&B's 2025 acquisition by CoreWeave, a GPU-cloud company — a meaningful vendor-lock-in consideration if the org already buys compute elsewhere), and whether the roadmap is trending toward LLM/agent observability (where Comet's Opik and MLflow's GenAI tracing features have both invested heavily) versus classical ML experiment tracking.

---

## 5. Common Mistakes

1. **Logging raw prompts/responses "just in case," by default.** The single most common privacy mistake — treating verbose debugging as free. It isn't; it's a stored liability with a retention clock and an access-control burden.
2. **Comparing runs evaluated against different dataset snapshots.** Any time the "frozen" eval set silently changes (someone fixes a bad label, adds new examples) without a version bump, every downstream comparison becomes invalid without anyone noticing.
3. **Sorting by a single top metric and promoting the #1 row blindly.** This is exactly the failure the `compare_runs.py` gate pattern exists to prevent — the top scorer that blows the latency budget should not win.
4. **Under-tagging runs.** Teams that skip `dataset_version` or `evaluator` tags end up with hundreds of runs they can filter by model but not reliably distinguish otherwise — effectively write-only data.
5. **Treating a small composite-score gap as a real difference without a significance check.** A 0.02 difference in composite score across runs with noisy LLM-judge scoring (Module 11) is frequently well within noise — export to a DataFrame and test before trusting a ranking.
6. **Using MLflow's legacy stage-transition API in new code.** New code should use aliases; teams that keep extending `transition_model_version_stage` calls in 2026 are building on a deprecated pattern that will require a migration later.
7. **Hard-coding a specific model version number in serving code** instead of resolving via alias — this defeats the entire point of the alias pattern (instant promotion/rollback without a redeploy).
8. **One giant, unfocused dashboard with dozens of metrics and no clear owner action per alert.** Metric sprawl without a "what do I do when this fires" mapping causes alert fatigue and ignored pages.
9. **Conflating exploratory experiment tracking with high-volume production telemetry**, e.g., trying to log every production inference call as its own MLflow run and overwhelming the tracking server.
10. **No PII detection step before logging derived fields.** Even a "safe" field like `query_category` can leak information if the categorization itself is too granular (e.g., a category of "divorce-proceedings-legal-advice" is itself sensitive) — sanitize thoughtfully, not just mechanically.

---

## 6. Best Practices and Production Tips

**When to use MLflow (or a tracking tool at all) vs. not:**
- Use it from the very first non-trivial experiment — the cost of adopting tracking discipline late (backfilling metadata for old runs) is far higher than adopting it early.
- Do not use a full tracking server for truly one-off, throwaway scratch work — but be honest with yourself about what counts as "throwaway"; most things that start that way get reused.

**Alternatives and when they fit better:**
- Choose W&B when the priority is fast onboarding for a distributed team, strong exploratory visualization (parallel coordinates, sweep dashboards), and you're comfortable with a managed SaaS cost model.
- Choose Comet when LLM/agent observability (via Opik) is a first-class near-term need alongside classical experiment tracking, not an afterthought.
- Choose self-hosted MLflow when you need full control over data residency, want to avoid recurring per-seat/usage SaaS costs at scale, and already have the Kubernetes/Docker operational maturity (Modules 05–06) to run and maintain a tracking server.
- Many mature teams run more than one tool for different jobs (MLflow for registry/governance + CI integration, W&B for exploratory visualization) rather than treating the choice as strictly exclusive.

**Cost model considerations:**
- Self-hosted MLflow: infrastructure cost (compute + Postgres/MySQL + object storage) plus the engineering time to operate it; no per-seat licensing.
- Managed Databricks MLflow: rolled into Databricks workspace pricing; lower ops burden, less infrastructure control.
- W&B: usage/seat-based SaaS pricing; now part of CoreWeave's stack — evaluate this against your existing GPU/cloud vendor relationships for the vendor-concentration angle.
- Comet: similarly usage/seat-based SaaS, with an open-source path (Opik) for the LLM-observability piece specifically.

**Scaling:**
- Move from local file store to a Postgres/MySQL-backed remote tracking server before team size or run volume makes local `mlruns/` collaboration painful — this is a Module 05/06-style containerized service, not a special MLflow-only skill.
- For inference-time telemetry, never write one tracking run per request at meaningful QPS — aggregate/sample, and route raw high-frequency metrics through a proper metrics pipeline (Module 14).

**Monitoring and security:**
- Encrypt telemetry at rest (the transcript's "encrypted S3" detail) and enforce retention windows (30-day rolling was the transcript's example) as policy, not as an afterthought someone remembers to configure.
- Restrict who can reassign registry aliases in production — an alias flip is equivalent to a deploy and should go through the same review/approval path as your Module 02 CI/CD promotion gates.
- Audit alias-reassignment history; it is your model's deployment lineage and should be treated with the same rigor as a code-deploy audit log.

**Performance trade-offs:**
- Rich per-run artifact logging (full eval JSONL dumps, plots) is valuable for debugging but adds storage and I/O cost — log full artifacts for the runs that survive filtering, not for every candidate in a 100-run sweep, if storage cost becomes a concern.
- Parallel-coordinates-style visual comparison scales well up to dozens-to-low-hundreds of runs for human pattern-spotting; beyond that, lean on programmatic filtering/Pareto analysis first, then visualize only the shortlist.

---

## 7. Interview Questions

**Q1: Why did MLflow move away from Staging/Production/Archived stages toward tags and aliases? What real workflow problem does this solve?**
*Model answer:* Stages modeled promotion as a single linear state per model version, but real promotion workflows need to express multiple simultaneous, independent facts about a version (e.g., "currently shadow-tested" and "owned by team X" and "validated by pipeline v7"). Aliases are mutable named pointers (e.g., `champion`, `challenger`) that production code resolves at load time, decoupling "which version is promoted" from a rigid state machine, and tags carry the rest of the governance metadata orthogonally.

**Q2: You have 40 experiment runs and need to pick one to promote. Walk through your process.**
*Model answer:* First confirm all 40 ran against the same frozen `dataset_version` tag — if not, discard incomparable runs. Filter by any hard constraints (e.g., `model == target`), then compute the Pareto frontier across quality, latency, and cost rather than sorting by one metric. Run a significance test (e.g., paired t-test) between the top candidate and the current champion to confirm the improvement isn't noise. Only then check hard promotion gates (latency budget, error rate) before registering and promoting via alias.

**Q3: How would you design privacy-safe telemetry for an LLM chatbot without losing debuggability?**
*Model answer:* Never log raw prompt/response text by default; hash user identifiers (e.g., SHA-256 with a salt) so events can be attributed to the same user without exposing identity; log derived, safe metadata (query length, category, PII-detected boolean, token counts, cost, latency); attach a trace ID for cross-system correlation; and apply an encrypted store with a bounded retention window (e.g., 30 days rolling). Raw content is only surfaced through a separate, access-controlled review path when policy explicitly allows it.

**Q4: What is the difference between MLflow's Tracking component and its Model Registry, and why are they separate?**
*Model answer:* Tracking is a low-friction, high-volume store of every experimental run's params/metrics/artifacts — optimized for logging anything you might want later. The Registry is a curated, governed catalog of specific model *versions* promoted from tracking runs, with lifecycle metadata (tags, aliases) and access control appropriate for "this is what's serving production." Keeping them separate lets teams log liberally during experimentation without polluting the governance layer that production deploys depend on.

**Q5: A prompt/model combination has the best offline eval score in your sweep, but you decide not to promote it. Justify that decision to a stakeholder.**
*Model answer:* A single top score does not guarantee production suitability — it may violate a latency SLA, cost budget, or safety/error-rate constraint that a promotion gate enforces. Explain the specific gate it failed (e.g., "P95 latency was 1400ms against a 900ms budget") and that promoting purely on offline quality without respecting these constraints would likely cause a user-facing regression or a runaway cost, which is a worse outcome than a slightly lower-scoring but balanced candidate.

**Q6: How do you keep experiment comparisons valid over time as your evaluation dataset evolves?**
*Model answer:* Treat the evaluation dataset itself as a versioned artifact (a `dataset_version` tag on every run), and any change to it — even a small label fix — as producing a new version that invalidates direct comparison to runs tagged with the old version. To compare across dataset versions, either re-run all candidates you care about against the new frozen version, or maintain a documented, audited mapping of what changed and why, rather than silently assuming continuity.

**Q7: What's wrong with logging every production inference call as its own MLflow run?**
*Model answer:* MLflow's tracking store (whether file-based or DB-backed) is designed for experiment-scale write volume, not high-QPS production request volume — this would overwhelm the backend and doesn't match its data model (params/metrics per discrete "run," not a continuous metrics time series). The correct pattern is to route high-frequency inference telemetry through a dedicated metrics/observability pipeline (OpenTelemetry/Prometheus, Module 14) and only mirror sampled or aggregated events into MLflow when correlating production behavior back to the experiment that produced the serving model.

**Q8: How would you decide between MLflow, Weights & Biases, and Comet for a new team?**
*Model answer:* Weigh self-hosting appetite and existing Kubernetes/Docker maturity (favors MLflow, open-source, no per-seat cost) against desire for a managed, fast-onboarding, highly visual collaboration experience (favors W&B, now part of CoreWeave's stack — note the vendor-concentration angle) against near-term need for LLM/agent-specific observability (favors Comet's Opik). Many teams end up using more than one tool for different stages of the lifecycle rather than treating it as strictly either/or.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

This module took the experiment-comparison discipline from the source lesson — frozen datasets, consistent tagging, multi-metric Pareto analysis, and balanced promotion gates — and expanded it into a full, production-grade MLflow deep dive covering Tracking, Projects, Models, and the Model Registry's modern alias-based promotion pattern. It also covered privacy-safe telemetry design (hash IDs, never log raw text, encrypted bounded retention) and live inference-metric monitoring (TTFT, P95/P99 latency, cost, error rate, cache-hit rate, fallback rate) wired to action-driven alerting, bridging directly into Module 14's observability content.

### Key Takeaways

- Experiment comparison is only valid when the evaluation dataset is frozen and versioned; everything else follows from that constraint.
- Tags (model, prompt version, dataset version, evaluator) turn a pile of runs into a searchable, reproducible system.
- Promotion should be gated on a balance of quality, latency, and cost — never on the single best score alone.
- MLflow 3.x's alias pattern (`champion`/`challenger`/`canary`) has replaced Staging/Production/Archived stages as the recommended promotion mechanism — know both, teach the former as current best practice.
- Telemetry defaults to IDs and metrics; raw content is the exception, gated by explicit policy.
- Different production signals warrant different response mechanisms — alert vs. page — chosen deliberately, not uniformly.

### Production Checklist

- [ ] Every experiment run is tagged with `model`, `prompt_version`, `dataset_version`, and `evaluator`.
- [ ] The evaluation dataset is versioned and frozen for the duration of any comparison; any change bumps the version and invalidates stale comparisons.
- [ ] Promotion logic checks quality **and** latency **and** cost thresholds, not quality alone.
- [ ] Registry promotion uses aliases (`champion`/`challenger`/`canary`), not legacy stage transitions.
- [ ] Production serving code resolves models via alias URI (`models:/name@alias`), never a hard-coded version number.
- [ ] Alias reassignment is access-controlled and audited like a production deploy.
- [ ] Telemetry never stores raw prompt/response text by default; user identifiers are hashed; retention is time-bounded and encrypted at rest.
- [ ] Inference metrics (TTFT, P95/P99 latency, cost/query, error rate, cache-hit rate, fallback rate) are tracked with explicit alert thresholds mapped to a specific response (Slack alert vs. page).
- [ ] High-QPS production inference telemetry is routed through a metrics pipeline, not logged as individual MLflow runs.
- [ ] A statistical significance check runs before treating a small composite-score improvement as a real win.

---

## 9. Further Reading

Detailed, verified links (official MLflow/W&B/Comet documentation, GitHub repositories, comparison analyses, and supporting videos/books) are maintained separately in this same module folder — see `references.md`, `videos.md`, `books.md`, and `github.md`. Consult those files for exact URLs, difficulty ratings, and estimated time investment rather than this file, so citations stay in one maintained place.
