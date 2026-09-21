# Module 03 — Versioning, Registries, and Rollback

> "In traditional software, rollback means redeploying an older binary. In ML and LLM systems, that is usually not enough — production behavior depends on the exact combination of model, prompt, and dataset that was alive at that time."

This module is the backbone of release safety for ML and LLM systems. It answers three questions that every senior MLOps/LLMOps engineer must be able to answer under pressure, at 2 a.m., during an incident: **What exactly is running in production right now? How did it get there? And how do we get back to the last known good state, fast, without guessing?**

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Apply semantic versioning discipline to **models, prompts, and datasets** — not just application code — and explain what each major/minor/patch bump means for each artifact type.
2. Explain and implement the **deployment triple** concept (model version + prompt version + dataset version) and why it must be logged per inference for reproducibility.
3. Walk through the full **MLflow Model Registry** lifecycle in Python: logging a run, registering a model, applying aliases/tags, promoting, and archiving — using the *current* (2024+) alias-based API, not the deprecated stage-based API.
4. Compare **MLflow, Weights & Biases, Comet, and custom/homegrown registries** on governance, cost, integration depth, and operational fit, and make a defensible registry choice for a given team profile.
5. Define objective, code-enforced **promotion criteria** (accuracy thresholds, regression deltas, latency gates) instead of manual "looks fine to me" promotions.
6. Design and implement a **full-tuple rollback** (model + prompt + dataset together, not model alone) with lineage-based restore, canary rollout, and audit logging.
7. Produce a release manifest (`release.yaml`) that captures the deployable triple, compatibility constraints, and rollout state as a single auditable source of truth.

### Prerequisites

- Comfort with Python, Git, and basic ML lifecycle concepts (training, evaluation, serving).
- Familiarity with the earlier modules in this course: experiment tracking fundamentals and CI/CD basics for ML.
- A conceptual understanding of REST APIs and containerized deployment (Docker/Kubernetes) is helpful for the architecture sections but not mandatory.

### Key Terminology

| Term | Definition |
|---|---|
| **Semantic Versioning (SemVer)** | A `MAJOR.MINOR.PATCH` scheme where major = breaking change, minor = backward-compatible addition, patch = safe fix. Originating in software packaging (semver.org), adapted here to models, prompts, and datasets. |
| **Deployment triple / tuple** | The exact combination of (model version, prompt version, dataset/feature version) that is active in production at a point in time. The unit of reproducibility for an ML/LLM inference. |
| **Model Registry** | A system of record that stores model artifacts, their versions, lineage metadata, evaluation metrics, and approval/deployment state, and controls how a version moves toward production. |
| **Alias** (MLflow) | A named, mutable pointer (e.g. `@champion`, `@challenger`) to a specific model version — the replacement for the deprecated fixed "Stage" concept (`Staging`/`Production`/`Archived`). |
| **Prompt Registry** | A versioned store for prompt templates, analogous to a model registry, supporting diffs, commit messages, and environment aliases (native in MLflow 3+ GenAI). |
| **Promotion criteria** | Objective, automated gates (absolute accuracy, relative regression delta, latency, safety/eval score) a candidate model must pass before it is promoted, replacing manual promotion. |
| **Lineage** | The recorded graph of what produced a given artifact — training run, code commit, data snapshot, hyperparameters, evaluation results — enabling "how did we get here" queries. |
| **Rollback** | Restoring a previous known-good deployment state. In ML/LLM systems this means restoring the *entire* triple, not just the model binary. |
| **Canary release** | Routing a small percentage of production traffic to a new release before full rollout, to detect regressions on real traffic with bounded blast radius. |
| **Last known good (LKG) state** | The most recent deployment tuple that was validated in production and did not trigger any rollback criteria — the default rollback target. |
| **Release manifest** | A single declarative file (e.g. `release.yaml`) describing the exact deployable triple, compatibility rules, and rollout/approval state. |

---

## 2. Why This Topic Matters and Where It Fits in the Lifecycle

Consider the standard MLOps/LLMOps lifecycle:

```
 Data ──► Feature/Prompt Eng ──► Training/Tuning ──► Evaluation ──► Registry ──► Deployment ──► Monitoring ──► (feedback loop)
                                                                        │
                                                        THIS MODULE LIVES HERE AND AT THE ROLLBACK ARROW BACK
```

Every earlier module in this course produces *candidates*: a trained model, a designed prompt, a curated dataset. None of that matters operationally until you can answer, with confidence, three questions that recur in every senior MLOps interview and every real incident:

1. **What is running in production right now, precisely?** (not "the fraud model", but `fraud-detector:2.1.0` + `reviewer-prompt:1.3.2` + `claims-golden:5.0.0`)
2. **How did that combination get approved?** (what metrics, what run, what approver, what gate)
3. **If it's wrong, how do we undo it — safely, quickly, and without introducing a second bug while fixing the first?**

This is the difference between "we trained a good model" and "we run a production ML platform." Interviewers probe this directly because it separates people who have shipped models from people who have operated them under real failure conditions. It is also where the LLM era changes the picture: a chatbot's behavior is not just "the model" — it is model weights (or a hosted API version string), a system prompt, retrieval/dataset content, and inference parameters, all changing independently and all capable of causing a regression on their own.

**Where this connects to neighboring modules:**

- *Upstream*: experiment tracking (Module 02-ish) produces the runs and metrics that feed registration decisions here.
- *Downstream*: CI/CD and deployment modules consume the registry's promoted version as their deployment artifact; observability modules consume the lineage/version tags to correlate production metrics back to a specific release.
- *Parallel*: evaluation/LLM-judge pipelines (used elsewhere in this course) are what actually *compute* the promotion-criteria numbers discussed below.

---

## 3. Main Concepts

### 3.1 Semantic Versioning for Models, Prompts, and Datasets

#### Theory

Classic SemVer (`MAJOR.MINOR.PATCH`) exists so that a version number alone tells a consumer whether upgrading is safe. In application code, "breaking" is defined by API contracts. In ML/LLM systems, we need an equivalent contract for three artifact types that each break in *different* ways:

| Artifact | MAJOR bump means | MINOR bump means | PATCH bump means |
|---|---|---|---|
| **Model** | New architecture / new model family; a change that alters the input/output contract or fundamentally changes behavior | Retrained on new/expanded training data; meaningfully changed capability without breaking existing consumers | Hyperparameter tweak, minor fine-tune, no material behavior shift |
| **Prompt** | Task redesigned; the prompt's contract with downstream consumers changes (e.g. output schema changes) | Wording changed to improve quality/safety without changing the contract | Whitespace, typo fix, formatting-only change |
| **Dataset** | Schema change (columns added/removed/retyped, label definition changes) | New records added, new data slice included | Deduplication, metadata corrections, no semantic content change |

**Why this problem exists:** In traditional software, if the code doesn't change, behavior doesn't change. In ML/LLM systems, **behavior can change while the application code is frozen** — because someone swapped the model artifact, edited the system prompt, or refreshed the reference dataset backing a RAG pipeline. Without a versioning discipline, "nothing changed" becomes an unverifiable claim. Semantic versioning turns "did anything change" into a diffable, auditable fact.

**Tradeoffs / when to use / when not to:**

- Use this everywhere you have more than one person touching models or prompts, or any compliance/audit requirement. It costs almost nothing (discipline + tooling) and pays off the first time someone asks "what changed between Tuesday and today."
- It is *not* a substitute for evaluation — a version bump tells you *that* something changed and roughly how risky it might be, not *whether* it's actually better. You still need promotion criteria (Section 3.3) to decide whether a new version should ship.
- Over-indexing on version-number bureaucracy without lineage/logging behind it is cargo-culting: the number becomes decorative rather than useful. The real payoff comes only when versions are tied to reproducible artifacts and logged per-inference (next section).

#### The Deployment Triple

**Theory.** A single ML/LLM prediction in production is a function of three independently-versioned things:

```
prediction = f( model_version, prompt_version, dataset_version, input )
```

If you only track `model_version`, you can explain at most one-third of "why did the output change." This is *the* single most important operational rule in this module: **always deploy as a triple, and log the triple on every inference.**

```
Example logged inference record:
{
  "request_id": "a83f2e1c",
  "timestamp": "2026-07-30T02:47:11Z",
  "model_version": "fraud-detector:2.1.0",
  "prompt_version": "reviewer-prompt:1.3.2",
  "dataset_version": "claims-golden:5.0.0",
  "input_hash": "sha256:...",
  "output": {...},
  "latency_ms": 142
}
```

**Why it matters:** debugging, audits, incident response, and regulatory compliance (e.g. explaining a credit or fraud decision months later) all require reconstructing *exactly* what produced a given output. Without the full triple logged, "we don't know which prompt was live at that time" is a real and common failure mode in teams that version only the model.

#### Architecture: Versioning Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         VERSIONING & RELEASE FLOW                        │
│                                                                          │
│   ┌───────────┐     ┌────────────┐     ┌────────────┐                 │
│   │  Model     │     │  Prompt     │     │  Dataset    │                 │
│   │  Registry  │     │  Registry   │     │  Registry/  │                 │
│   │ (MLflow)   │     │ (MLflow 3   │     │  DVC        │                 │
│   │            │     │  GenAI)     │     │             │                 │
│   │ 2.1.0      │     │ 1.3.2       │     │ 5.0.0       │                 │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘                 │
│         │                  │                  │                        │
│         └─────────┬────────┴─────────┬────────┘                        │
│                    ▼                            ▼                        │
│           ┌────────────────────────────────────┐                       │
│           │   release.yaml  (deployment triple)│                       │
│           │   + compatibility rules             │                       │
│           │   + rollout state                   │                       │
│           └───────────────┬────────────────────┘                       │
│                            ▼                                            │
│                  ┌───────────────────┐                                  │
│                  │  Serving runtime   │  ──► logs triple per inference  │
│                  └───────────────────┘                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Examples

**Beginner** — a single-file convention, no tooling: a team names files `model_v2.1.0.pkl`, `prompt_v1.3.2.txt`, and stores the dataset hash in a README. Works for a solo project; breaks down fast with more than one contributor because nothing enforces consistency.

**Intermediate** — Git tags plus a manifest file per release (shown below), checked into version control, reviewed via pull request before deployment.

**Production-grade** — MLflow Model Registry (model versions + aliases) + MLflow 3 GenAI Prompt Registry (prompt versions + environment aliases) + DVC or a lakehouse table version (dataset versions), all referenced from a single `release.yaml`, with CI enforcing that a release cannot deploy unless all three components resolve to existing, evaluated versions.

#### Code: `release.yaml`

```yaml
# release.yaml — single source of truth for a deployable serving package
release:
  name: fraud-detection-service
  model:
    name: fraud-detector
    version: "2.1.0"
  prompt:
    name: reviewer-prompt
    version: "1.3.2"
  dataset:
    name: claims-golden
    version: "5.0.0"

compatibility:
  min_dataset_major: 5      # this model requires dataset major >= 5 (schema match)
  required_prompt_major: 1  # prompt contract must be major version 1

rollout:
  stage: staging
  approval: pending
  canary_percent: 0
  approved_by: null
  approved_at: null
```

This is a strong pattern precisely because **every serving release is described in one place**: the exact triple, the compatibility rules that prevent unsupported combinations from reaching production (e.g. a model that requires dataset schema v5+ must refuse to deploy against a v4 dataset), and the current rollout state. It becomes the artifact that is code-reviewed, that CI validates, and that an incident responder reads first.

#### Key Versioning Principles

1. **Consistency** — the same versioning logic applies every time; a version number means the same thing across teams.
2. **Traceability** — every version maps back to a specific change history and release decision.
3. **Compatibility awareness** — not every version combination is safe; compatibility rules must be explicit (see `compatibility:` block above).
4. **Reproducibility** — given a logged triple, you can reconstruct the exact release state that produced any historical output.

---

### 3.2 Model Registries: MLflow vs. Weights & Biases vs. Comet vs. Custom

#### Theory

Training produces an artifact. A **registry** is what turns that artifact into a governed, discoverable, promotable asset. Without one, "which model is in production" lives in someone's memory, a Slack thread, or a deploy script's hardcoded path — none of which survive team turnover or an incident at 2 a.m.

A registry's job, concretely:

- Store model **versions** (immutable, uniquely identified).
- Attach **metadata**: metrics, training run reference, data version, environment/library versions.
- Provide a **promotion mechanism** — a controlled way to say "this version is now the one serving traffic," gated by policy, not by someone clicking a button based on intuition.
- Provide **lineage**: given a registered version, trace back to the exact run, code commit, and data that produced it.

**Important 2026 correction:** MLflow's original registry model used fixed **Stages** — `None → Staging → Production → Archived`. This has been **deprecated since MLflow 2.9** in favor of **aliases** (arbitrary named pointers like `@champion`, `@challenger`, `@shadow`) plus **tags** for arbitrary metadata. If you see a tutorial or a candidate's answer describing `client.transition_model_version_stage(...)` as current best practice, that is describing a deprecated pattern — the correct mental model now is: a model version is immutable and gets **tagged and aliased**, and your own environment names (dev/staging/prod) are just aliases you define, not a fixed enum MLflow enforces for you. This distinction is a favorite senior-level interview trap in 2026.

#### Registry Comparison Table

| Dimension | MLflow Model Registry | Weights & Biases (W&B) Registry | Comet Model Registry | Custom / Homegrown Registry |
|---|---|---|---|---|
| **Purpose** | Open-source system of record for models (and, since MLflow 3, prompts) tightly coupled to experiment tracking | Registry layer on top of W&B's experiment tracking/artifacts, oriented around "collections" and access control | Registry integrated with Comet's experiment tracking, model versioning, and comparison UI | Purpose-built internal system, often a thin layer over a database + blob storage + internal approval tooling |
| **Governance model** | Aliases + tags + webhooks; permissions via backend store (e.g. Unity Catalog on Databricks) | Role-based access, staged promotion via "Registry" collections shared across teams | Stage-based workflow with UI-driven promotion and comments | Whatever the org builds — full flexibility |
| **Experiment tracking integration** | Native, same tool (MLflow Tracking → Registry is one hop) | Native, same tool (W&B Runs → Registry) | Native, same tool | Must be built/integrated manually |
| **Prompt/GenAI support** | Native Prompt Registry since MLflow 3 (versions, diffs, commit messages, environment aliases, `search_prompts`) — closes the "model+prompt" gap in one platform | Primarily model/artifact-centric; prompt versioning typically bolted on via artifacts, not first-class | Primarily model-centric; prompt tracking less mature than MLflow 3 | Fully custom — most orgs that need this build it themselves |
| **Open source / self-hostable** | Yes, fully open source (Apache-2.0), can self-host | Partially — core product is SaaS; open-source client library | Partially — SaaS-first with free tier; some OSS tooling | Yes, by definition |
| **Cost model** | Free (self-hosted) or bundled with Databricks-managed MLflow | Per-seat/usage SaaS pricing | Per-seat/usage SaaS pricing | Engineering time (build + maintain) instead of license cost |
| **Best fit** | Teams wanting the de facto open-source standard, deep customization, or Databricks-native lakehouse integration; also the best current choice if you want models AND prompts in one registry | Teams already heavily invested in W&B for experiment tracking who want registry + tracking in one pane of glass | Teams already using Comet for tracking; smaller ecosystems, simpler needs | Organizations with unusual compliance/approval requirements, deep internal platform integration needs, or regulatory constraints that off-the-shelf tools can't satisfy |
| **Weaknesses** | UI is functional but less polished than SaaS competitors; self-hosting requires operating the backend store yourself | Vendor lock-in to W&B ecosystem; cost scales with team/usage | Smaller ecosystem/community than MLflow; fewer LLMOps-specific features as of 2026 | High build and maintenance cost; reinventing lineage/audit features is easy to get wrong; onboarding new engineers is harder (no shared external documentation) |
| **Production evidence** | Confirmed to be the dominant open-source registry (mlflow/mlflow, on the order of ~27k GitHub stars); default choice across a large share of the industry | Used heavily by teams already standardized on W&B for tracking (common in research-heavy orgs) | Used by teams standardized on Comet; smaller footprint than MLflow/W&B | Large-scale platforms like Uber's Michelangelo built a custom registry ("Gallery") because their scale and internal deployment integration needs exceeded what any off-the-shelf tool provided at the time it was built |

#### Choosing the Right Registry — Decision Factors

The registry choice should follow the team's **operating model**, not tool popularity:

- **Release cadence**: quarterly releases can tolerate more manual review; daily releases need registries with strong automation/API support (MLflow's Python API is a strength here).
- **Governance load**: how many stakeholders must sign off? Regulated industries (finance, healthcare) want strong approval-workflow and audit-trail features.
- **Centralization**: is deployment centralized (one platform team owns rollout) or decentralized (each team deploys independently)? Decentralized setups benefit from a registry with strong RBAC and namespacing.
- **Lineage depth needed**: do you need to trace back to raw training data and feature snapshots, or is "which run produced this" sufficient?
- **Existing tooling gravity**: if the team already tracks experiments in W&B or Comet, the registry from the same vendor reduces integration friction — but check whether it satisfies the GenAI/prompt requirements you'll need in an LLMOps context, where MLflow 3 currently has a genuine edge.

#### Architecture: Registry-Centered Promotion Flow

```
┌────────────┐     ┌───────────────┐     ┌───────────────────┐     ┌────────────────┐
│  Training   │────►│  Evaluation    │────►│  Register Model    │────►│  Promotion Gate  │
│  Run        │     │  (metrics,     │     │  Version (MLflow)  │     │  (code-driven,   │
│ (MLflow     │     │  eval harness) │     │  + tags + lineage   │     │  automated       │
│  Tracking)  │     │                │     │                     │     │  checks)         │
└────────────┘     └───────────────┘     └─────────┬─────────┘     └────────┬───────┘
                                                     │                        │
                                                     ▼                        ▼
                                          ┌─────────────────────┐   ┌─────────────────┐
                                          │ alias: @challenger    │   │ alias: @champion  │
                                          │ (shadow / canary)     │   │ (serving prod)    │
                                          └─────────────────────┘   └─────────────────┘
```

#### Code: Full MLflow Model Registry Walkthrough (Python, current alias-based API)

```python
import mlflow
from mlflow import MlflowClient
from mlflow.models import infer_signature
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("fraud-detection")

client = MlflowClient()
MODEL_NAME = "fraud-detector"

# ── 1. Train + log a run ────────────────────────────────────────────────
with mlflow.start_run(run_name="rf-v2-retrain") as run:
    model = RandomForestClassifier(n_estimators=300, max_depth=12, random_state=42)
    model.fit(X_train, y_train)

    preds = model.predict(X_val)
    val_accuracy = accuracy_score(y_val, preds)

    mlflow.log_params({"n_estimators": 300, "max_depth": 12})
    mlflow.log_metric("val_accuracy", val_accuracy)
    mlflow.set_tag("dataset_version", "claims-golden:5.0.0")
    mlflow.set_tag("prompt_version", "n/a")  # classic ML model, no prompt leg

    signature = infer_signature(X_train, model.predict(X_train))

    # ── 2. Register the model artifact as a new version ────────────────
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        signature=signature,
        registered_model_name=MODEL_NAME,   # this call auto-creates version N
    )
    run_id = run.info.run_id

# Fetch the version number MLflow just assigned
latest_version = client.get_latest_versions(MODEL_NAME)[0].version
print(f"Registered {MODEL_NAME} version {latest_version} from run {run_id}")

# ── 3. Tag the version with semantic version + lineage metadata ────────
client.set_model_version_tag(MODEL_NAME, latest_version, "semver", "2.1.0")
client.set_model_version_tag(MODEL_NAME, latest_version, "dataset_version", "claims-golden:5.0.0")
client.set_model_version_tag(MODEL_NAME, latest_version, "run_id", run_id)

# ── 4. Point the "challenger" alias at the new version for evaluation ──
client.set_registered_model_alias(MODEL_NAME, "challenger", latest_version)

# ── 5. Automated, code-driven promotion gate (never manual) ────────────
def passes_promotion_criteria(challenger_version: str, champion_alias: str = "champion") -> bool:
    challenger_mv = client.get_model_version(MODEL_NAME, challenger_version)
    challenger_acc = float(client.get_run(challenger_mv.run_id).data.metrics["val_accuracy"])

    try:
        champion_mv = client.get_model_version_by_alias(MODEL_NAME, champion_alias)
        champion_acc = float(client.get_run(champion_mv.run_id).data.metrics["val_accuracy"])
    except mlflow.exceptions.RestException:
        champion_acc = 0.0  # no champion yet — first ever promotion

    absolute_ok = challenger_acc >= 0.88
    relative_delta = (challenger_acc - champion_acc) / max(champion_acc, 1e-9)
    regression_ok = relative_delta >= -0.01          # no worse than -1%
    latency_ok = measure_p95_latency_ms(challenger_mv) < 2000

    print(f"absolute={challenger_acc:.4f} (>=0.88? {absolute_ok}) | "
          f"delta={relative_delta:.4%} (>=-1%? {regression_ok}) | "
          f"p95_latency_ok={latency_ok}")

    return absolute_ok and regression_ok and latency_ok


if passes_promotion_criteria(latest_version):
    # ── 6. Promote: repoint the "champion" alias — this IS production ──
    client.set_registered_model_alias(MODEL_NAME, "champion", latest_version)
    client.set_model_version_tag(MODEL_NAME, latest_version, "promoted_at", str(datetime.utcnow()))
    print(f"Promoted version {latest_version} to @champion")
else:
    print(f"Version {latest_version} failed promotion gate — remains @challenger only")

# ── 7. Archive an old version once it's no longer needed anywhere ──────
client.set_model_version_tag(MODEL_NAME, "18", "archived_reason", "superseded by 2.1.0, regression on cohort B")
# Old-style stage transition (DEPRECATED since MLflow 2.9 — shown for contrast only):
# client.transition_model_version_stage(MODEL_NAME, "18", stage="Archived")  # DO NOT USE in new code

# ── 8. Serving code always resolves the alias, never a hardcoded version ─
production_model_uri = f"models:/{MODEL_NAME}@champion"
production_model = mlflow.pyfunc.load_model(production_model_uri)
```

Notice the design: **serving code never hardcodes a version number.** It resolves `models:/fraud-detector@champion` at load time. Promotion becomes an atomic alias repoint — no redeploy of serving code required, and rollback (Section 3.3) becomes "repoint the alias back," which is fast and safe.

#### MLflow 3 GenAI Prompt Registry (closing the "prompt" leg)

```python
import mlflow

# Register a new prompt version
prompt = mlflow.genai.register_prompt(
    name="reviewer-prompt",
    template="You are a claims reviewer. Given: {{claim_details}}\nDecide: APPROVE or FLAG, with reason.",
    commit_message="Add explicit reasoning requirement to reduce silent flags",
)
print(f"Registered prompt version: {prompt.version}")   # e.g. "1.3.2"-style semver via tags

mlflow.genai.set_prompt_alias(name="reviewer-prompt", alias="production", version=prompt.version)

# Resolve at serving time — mirrors the model alias pattern exactly
prod_prompt = mlflow.genai.load_prompt("prompts:/reviewer-prompt@production")
```

This is the practical fix for what used to require PromptLayer or LangSmith's Prompt Hub bolted onto MLflow — as of MLflow 3, the model registry and prompt registry live in the same platform, which directly supports logging the deployment triple from Section 3.1 out of one system.

#### Examples across maturity levels

- **Beginner**: a shared folder of `.pkl` files with a spreadsheet tracking which one is "live." Works for a hobby project; fails the moment two people touch it the same week.
- **Intermediate**: MLflow Tracking + Registry on a single VM or Docker Compose stack, alias-based promotion, manual review of the promotion gate output before repointing `@champion`.
- **Production-grade**: MLflow backed by a managed database (e.g. Postgres) and object storage (e.g. S3/GCS/Azure Blob) or a managed offering (Databricks Unity Catalog for models), promotion gates fully automated in CI, webhooks firing on alias changes to trigger deployment pipelines, and prompt registry entries versioned alongside model versions in the same release manifest.

---

### 3.3 Promotion Criteria: Turning Release Decisions into Engineering, Not Intuition

#### Theory

The single most important sentence in this module: **never promote to production manually.** Promotion must be driven by code and policy. This is not about distrust of engineers — it's about removing a class of failure where "it looked fine in the demo" substitutes for a statistically grounded comparison against the current production model on held-out or live-shadow traffic.

A minimal, defensible promotion gate combines at least three kinds of checks:

| Criterion | Example threshold | What it protects against |
|---|---|---|
| **Absolute quality floor** | accuracy ≥ 0.88 (or eval-score ≥ X for an LLM judge) | Shipping a model that is simply not good enough in isolation |
| **Relative regression guard** | relative delta ≥ −1% vs. current champion | Shipping a model that is technically "okay" but worse than what's already live — the most common real-world regression |
| **Latency/cost gate** | P95 latency < 2000 ms (or cost-per-request ceiling for LLM calls) | Shipping a model that is accurate but operationally unacceptable (SLA breach, budget blowout) |
| **Safety/behavioral gate (LLM-specific)** | LLM-as-judge score above threshold on a red-team eval set; toxicity/PII leakage below threshold | Shipping a model/prompt combination that regresses on safety even if task accuracy improves |

**When NOT to fully automate**: high-stakes regulated domains (e.g. a first deployment into a new legal jurisdiction, or a model touching credit decisions) often still require a human approval step *in addition to* the automated gates — the code enforces the objective floor, and a human signs off on the subjective/compliance dimension. The mistake to avoid is having *only* a human gate with no objective floor — that's the "manual promotion" anti-pattern this section warns against.

#### Example: promotion gate as a CI check (GitHub Actions)

```yaml
# .github/workflows/promote-model.yml
name: Promote Model Candidate
on:
  workflow_dispatch:
    inputs:
      model_version:
        description: "Candidate model version to evaluate for promotion"
        required: true

jobs:
  promotion-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install deps
        run: pip install mlflow scikit-learn
      - name: Run promotion criteria check
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: python scripts/check_promotion_criteria.py --version "${{ inputs.model_version }}"
      - name: Promote alias if gate passed
        if: success()
        run: python scripts/promote_alias.py --version "${{ inputs.model_version }}" --alias champion
```

This turns "promote to prod" into a reviewable, re-runnable, auditable CI job rather than a person clicking "Transition to Production" in a UI.

---

### 3.4 Rollback Strategy and Lineage Tracking

#### Theory

Rollback in ML/LLM systems must be **planned in advance, not improvised during an incident.** The core insight from this module's source material: a rollback is not "redeploy an older model," it is **restore the entire last-known-good deployment tuple** — model, prompt, and dataset together — because a regression can come from the *interaction* between these components, not any single one in isolation.

A rollback strategy must define, ahead of time:

1. What counts as "last known good" (LKG) — the last validated *deployment state*, not simply "the previous model version."
2. What metrics trigger a rollback (accuracy drop, latency spike, safety-eval failure, error-rate spike).
3. Who or what can authorize a rollback (automated trigger vs. human on-call approval).
4. How the previous version is mechanically restored.
5. How the rollback event itself is recorded for post-incident review.

**Why this is genuinely different from traditional software rollback:** infrastructure health checks do not catch this class of failure. The service can be up, latency normal, error rate at zero — and the *business metric* (accuracy, fraud catch rate, hallucination rate) can still have collapsed. This means ML/LLM on-call runbooks must include model-quality dashboards as a first-class signal, not just infra health.

#### Emergency Rollback Scenario (illustrative incident narrative)

```
02:47 AM  — Production accuracy alert fires: 0.89 → 0.81 (a material drop)
02:48 AM  — On-call runbook triggered
02:49 AM  — Step 1: Confirm the drop is real (not a monitoring blip / data pipeline gap)
02:51 AM  — Step 2: Identify currently deployed release tuple
                  model=fraud-detector:2.1.0, prompt=reviewer-prompt:1.4.5, dataset=claims-golden:5.1.0
02:53 AM  — Step 3: Locate last known good (LKG) tuple from lineage store
                  model=fraud-detector:2.0.1, prompt=reviewer-prompt:1.4.2, dataset=claims-golden:5.0.0
02:54 AM  — Step 4: Restore full tuple (not model alone)
02:55 AM  — Step 5: Record rollback event (reason, target versions, timestamp, operator)
02:58 AM  — Accuracy recovers to 0.89; incident moves to postmortem phase
```

The critical detail: the bad release combined `model:2.1.0 + prompt:1.4.5 + dataset:5.1.0`. If the on-call engineer had only rolled back the model (to `2.0.1`) while leaving the new prompt (`1.4.5`) and new dataset (`5.1.0`) in place, the regression might well have persisted — because the fault may have been in how the new prompt interacted with the new dataset's distribution, not in the model weights at all. **Restoring the full tuple, atomically, is the only version of rollback that is actually deterministic.**

#### Canary Releases as a Rollback-Prevention Layer

Rather than shipping a new release to 100% of traffic immediately, expose it to a small percentage first, observe quality/latency/stability, and only then expand.

```
Traffic Ramp for a New Release Tuple
──────────────────────────────────────────────────────────────
 t0        5% canary     ──► observe accuracy, latency, safety evals
 t0+30min  25% canary     ──► still healthy? continue
 t0+2hr    100% rollout   ──► promote @champion fully
                          │
                          └─► at ANY point, metrics breach threshold
                                    │
                                    ▼
                          automatic rollback to 0% (full tuple restore)
```

Canaries matter especially in ML/LLM systems because **some failures only appear on real production input distributions** — a held-out validation set cannot fully simulate adversarial or unusual real-world prompts/inputs. The canary is the safety buffer between "passed offline eval" and "safe at 100% production traffic."

#### Architecture: Lineage-Backed Rollback System

```
┌───────────────────────────────────────────────────────────────────────┐
│                          LINEAGE STORE                                │
│  (MLflow run metadata + registry tags + release.yaml history in git)  │
│                                                                        │
│  release_id=r118  model=2.1.0  prompt=1.4.5  dataset=5.1.0  BAD  ◄──── currently live
│  release_id=r117  model=2.0.1  prompt=1.4.2  dataset=5.0.0  GOOD ◄──── last known good
│  release_id=r116  model=2.0.1  prompt=1.4.0  dataset=5.0.0  GOOD
│  ...                                                                   │
└─────────────────────────────┬─────────────────────────────────────────┘
                               │  query: "last GOOD release before r118"
                               ▼
                     ┌───────────────────┐
                     │  Rollback Engine    │
                     │  1. resolve LKG tuple│
                     │  2. restore model    │
                     │  3. restore prompt   │
                     │  4. restore dataset  │
                     │  5. write audit event│
                     └─────────┬─────────┘
                               ▼
                     ┌───────────────────┐
                     │  Serving Runtime    │  (aliases repointed atomically)
                     └───────────────────┘
```

#### Sequence: Rollback Under Incident Conditions

```
On-call Engineer      Alerting System      Lineage Store        Rollback Engine       Serving Runtime
      │                     │                     │                     │                     │
      │◄── accuracy alert ──┤                     │                     │                     │
      │                     │                     │                     │                     │
      ├── confirm drop is real (check dashboards, rule out pipeline gap) ─────────────────────►│
      │                     │                     │                     │                     │
      ├── query current release tuple ───────────►│                     │                     │
      │                     │                     ├── returns r118 ────►│                     │
      │◄──────────────────────────────────────────┤                     │                     │
      │                     │                     │                     │                     │
      ├── request "last known good before r118" ─►│                     │                     │
      │                     │                     ├── returns r117 ────►│                     │
      │◄──────────────────────────────────────────┤                     │                     │
      │                     │                     │                     │                     │
      ├── authorize rollback to r117 ─────────────────────────────────►│                     │
      │                     │                     │                     ├── restore model ───►│
      │                     │                     │                     ├── restore prompt ──►│
      │                     │                     │                     ├── restore dataset ─►│
      │                     │                     │                     │◄── confirm live ────┤
      │                     │                     │◄── write audit event ┤                     │
      │◄── rollback complete, accuracy recovering ─────────────────────┤                     │
```

#### Code: Full-Tuple Rollback Script

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from mlflow import MlflowClient

client = MlflowClient()

@dataclass(frozen=True)
class ReleaseTuple:
    model_name: str
    model_version: str
    prompt_name: str
    prompt_version: str
    dataset_name: str
    dataset_version: str
    release_id: str


def get_last_known_good(before_release_id: str) -> ReleaseTuple:
    """Query the lineage store (here: MLflow tags on a tracking 'releases' run,
    or equivalently a release.yaml history in git) for the most recent
    release tagged GOOD prior to the given release id."""
    releases = client.search_runs(
        experiment_ids=[RELEASES_EXPERIMENT_ID],
        filter_string="tags.status = 'GOOD'",
        order_by=["attributes.start_time DESC"],
    )
    for run in releases:
        if run.data.tags["release_id"] < before_release_id:
            return ReleaseTuple(
                model_name=run.data.tags["model_name"],
                model_version=run.data.tags["model_version"],
                prompt_name=run.data.tags["prompt_name"],
                prompt_version=run.data.tags["prompt_version"],
                dataset_name=run.data.tags["dataset_name"],
                dataset_version=run.data.tags["dataset_version"],
                release_id=run.data.tags["release_id"],
            )
    raise RuntimeError("No known-good release found in lineage store")


def restore_full_tuple(target: ReleaseTuple, reason: str, operator: str) -> None:
    """Atomically restore model, prompt, and dataset together — never
    the model alone — then record an auditable rollback event."""

    # 1. Switch the model: repoint the serving alias
    client.set_registered_model_alias(target.model_name, "champion", target.model_version)

    # 2. Pin the prompt: repoint the prompt alias
    import mlflow
    mlflow.genai.set_prompt_alias(
        name=target.prompt_name, alias="production", version=target.prompt_version
    )

    # 3. Mount the dataset: point the serving config / feature store reference
    update_serving_dataset_pointer(target.dataset_name, target.dataset_version)

    # 4. Record the rollback event for audit and postmortem
    with mlflow.start_run(run_name="rollback-event", experiment_id=RELEASES_EXPERIMENT_ID):
        mlflow.set_tags({
            "event_type": "rollback",
            "reason": reason,
            "operator": operator,
            "restored_model_version": target.model_version,
            "restored_prompt_version": target.prompt_version,
            "restored_dataset_version": target.dataset_version,
            "restored_release_id": target.release_id,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })

    print(f"Rolled back to release {target.release_id}: "
          f"model={target.model_version}, prompt={target.prompt_version}, "
          f"dataset={target.dataset_version}")


# ── Practical example from the incident narrative above ────────────────
bad_release_id = "r118"  # model=2.1.0, prompt=1.4.5, dataset=5.1.0 — drifting in prod
lkg = get_last_known_good(before_release_id=bad_release_id)
restore_full_tuple(
    target=lkg,
    reason="Accuracy dropped 0.89 -> 0.81 within 15 minutes of r118 rollout; "
           "suspected prompt/dataset interaction regression",
    operator="on-call:jdoe",
)
```

Note what this script deliberately does **not** do: it does not attempt to roll back only the model while leaving the prompt or dataset at their current (bad) versions. That partial-rollback temptation — "just revert the model, it's probably the model" — is exactly the mistake this module exists to prevent.

#### Key Principles of Safe Rollback

1. **Determinism** — a rollback restores a known-good state *exactly*, not approximately.
2. **Traceability** — you always know what was deployed, when, and what upstream assets were involved.
3. **Speed with control** — rollback must be fast, but guarded (e.g. requiring the target to be a recorded LKG tuple, not an arbitrary version) so the response doesn't make things worse.
4. **Auditability** — every rollback event is recorded with reason, target version, timestamp, and operator, so incident reviews are based on facts, not memory.

---

## 4. Real-World Case Studies (Reasoned Inference)

> These describe how systems like this would plausibly be architected based on publicly documented engineering-blog patterns and industry norms — not confirmed internal implementation details.

**Uber (Michelangelo platform).** Uber's engineering blog describes Michelangelo training on the order of ~20,000 models per month and serving 5,000+ models concurrently in production. At that scale, manual promotion is simply not viable — a system like this would necessarily rely on its "Gallery" registry component plus automated deployment-safety and regression checks gating every promotion, exactly mirroring the "never promote manually" principle in this module. With thousands of concurrently-served models, lineage tracking would need to be a first-class, queryable system (not tribal knowledge) purely to make on-call rollback tractable at all — a human cannot hold "which of 5,000 models changed last week" in their head.

**Netflix.** Per reporting on Netflix's engineering practices (InfoQ, May 2026), Netflix introduced a "Model Lifecycle Graph" representing explicit model-training-data-code-deployment lineage to support governance and rollback at enterprise scale. This is a direct, real-world instance of the lineage-based restore pattern this module teaches: rather than treating "rollback" as an ad hoc script, a system like this treats lineage as a queryable graph so that "what was the last known good state before X" becomes a graph query rather than an archaeology exercise across logs, dashboards, and Slack history.

**OpenAI / Anthropic (LLM providers).** A production LLM platform like this would plausibly need to version at least four independently-evolving legs: base/fine-tuned model checkpoint (or hosted model snapshot string), system prompt / tool-use scaffolding, safety/guardrail configuration, and any retrieval corpus or few-shot example set backing a given deployment. Given the pace of prompt iteration in such systems, a registry-like internal tool with prompt diffing and environment aliasing (conceptually similar to MLflow 3's GenAI Prompt Registry) would be a natural fit, and canary/staged rollout of prompt or safety-configuration changes — given the blast radius of a regression across a very large user base — would plausibly be standard practice before any global rollout, mirroring the canary pattern in Section 3.4.

**Databricks.** As the primary commercial steward of MLflow, a platform like Databricks' managed MLflow would plausibly integrate the Model Registry directly with Unity Catalog for governance (access control, lineage across the full data-to-model chain) — extending the "registry as system of record" idea in this module to cover not just model versions but the full data-model-permission graph in one governed catalog.

**Spotify / Airbnb (recommendation-heavy consumer platforms).** Systems like this typically serve many concurrently-running model variants (per-market, per-experiment-arm) — a registry with strong tagging/metadata search (rather than a small fixed stage enum) becomes essential simply to keep track of which variant is live where, reinforcing why the industry moved away from MLflow's old fixed-stage model toward flexible aliases and tags.

---

## 5. Common Mistakes

1. **Versioning only the model, not the triple.** Teams track model versions carefully and then can't explain a production regression because the prompt or dataset also changed and nobody logged it.
2. **Using MLflow's deprecated Stages as if they were current best practice.** `Staging`/`Production`/`Archived` transitions are deprecated since MLflow 2.9 — new systems should use aliases (`@champion`/`@challenger`) and tags.
3. **Manual promotion ("looks good to me").** Skipping objective, code-enforced gates in favor of a person eyeballing a dashboard and clicking promote.
4. **Rolling back the model only.** During an incident, reverting just the model binary while leaving a newly-changed prompt or dataset in place — leaving the actual root cause untouched.
5. **No canary step — going straight to 100% traffic.** Skips the chance to catch a regression on a small, bounded blast radius before it affects all users.
6. **Treating version bumps as decorative.** Bumping a "minor" version without any actual evaluation behind it, so the number stops meaning anything to stakeholders.
7. **No audit trail on rollback events.** Rolling back without recording reason/timestamp/operator, making postmortems reconstruct events from memory or scattered logs.
8. **Conflating "last previous version" with "last known good."** The immediately-preceding version might itself have been a bad release; rollback logic must query for the last *validated* state, not simply decrement a counter.
9. **Registry as pure binary storage.** Treating the registry as "just a place model files live" instead of populating it with the metadata (metrics, run reference, data version, environment) needed to actually use it during an incident.
10. **No compatibility rules between components.** Deploying a model that requires a new dataset schema against an old dataset version because nothing enforced the constraint declared in the release manifest.

---

## 6. Best Practices and Production Tips

**When to use a formal registry + full versioning discipline:** any team with more than one contributor to models/prompts, any regulated domain, any system where a wrong output has real cost (financial, safety, reputational), or any team shipping more than a handful of times per year. Essentially: default to yes.

**When it may be overkill:** a single-person research prototype that will never see production traffic. Even then, lightweight versioning (git tags + a manifest file) costs little and pays off the moment the prototype graduates.

**Alternatives to a full registry**: for very small teams, a disciplined convention of Git-tagged release manifests (like the `release.yaml` shown above) plus DVC for dataset versioning can substitute for a full MLflow/W&B deployment, at the cost of losing built-in lineage queries and promotion automation.

**Cost considerations**: MLflow self-hosted is free but requires operating a backend store and object storage; managed offerings (Databricks-hosted MLflow, W&B, Comet) trade operational burden for subscription cost. At scale (Uber/Netflix-class), the calculus often shifts toward custom systems because the marginal engineering cost of a bespoke registry is justified by deep integration with existing deployment infrastructure.

**Scaling considerations**: as the number of concurrently-served models grows, favor tag/alias-based organization over any fixed enum of stages — you cannot scale "one Production slot" to thousands of models across dozens of markets/experiment arms.

**Monitoring**: promotion gates and rollback triggers should read from the same production-metrics pipeline that feeds your dashboards, not a separate offline-only evaluation path — otherwise you risk promoting on stale or unrepresentative numbers.

**Security**: registries often hold or reference sensitive training data lineage; apply the same access control rigor to the registry as to the underlying data (who can promote to `@champion`, who can read lineage metadata, audit log immutability).

**Performance tradeoffs**: canary percentages and gate evaluation windows trade rollout speed against blast-radius containment — a slower ramp catches more regressions but delays full benefit realization; tune the ramp schedule to the cost of a regression in your specific domain.

**Alerting**: business-metric dashboards (accuracy, hallucination rate, safety-eval score) must be first-class on-call signals alongside infra health — the module's central lesson is that infra can be perfectly healthy while the model is actively failing.

---

## 7. Interview Questions

1. **"Why isn't versioning the model enough in an ML/LLM system?"**
   *Model answer:* Because production behavior is a function of the full deployment triple — model, prompt, and dataset/feature version — all of which can change independently. Logging only the model version leaves you unable to explain regressions caused by prompt or data changes, and unable to fully reconstruct historical predictions for debugging or compliance.

2. **"MLflow used to have Staging/Production stages. What's the current (2026) way to do this, and why did it change?"**
   *Model answer:* Fixed Stages were deprecated starting MLflow 2.9 in favor of aliases (arbitrary named pointers like `@champion`/`@challenger`) plus tags. This is more flexible — organizations can define as many named environments/roles as needed instead of being locked into a fixed enum, and it scales better when serving many concurrent model variants.

3. **"Design a promotion gate for a new model version. What criteria would you include, and why must this be automated rather than manual?"**
   *Model answer:* At minimum: an absolute quality floor (e.g. accuracy ≥ 0.88), a relative regression guard against the current production model (e.g. delta ≥ −1%), and an operational gate (e.g. P95 latency < 2000ms). Automating this removes subjective, inconsistent human judgment from a decision that has real production risk, and makes the decision reviewable/reproducible in CI.

4. **"An incident occurs: accuracy dropped from 0.89 to 0.81 overnight, but the service is healthy (up, normal latency). Walk me through your response."**
   *Model answer:* First confirm the drop is real (rule out a monitoring/data pipeline issue). Then identify the exact currently-deployed release tuple (model+prompt+dataset versions) from the lineage store. Query for the last known-good tuple — the last validated deployment state, not simply the prior version. Restore the *entire* tuple atomically (not just the model), then record the rollback event with reason, timestamp, operator, and restored versions for the postmortem.

5. **"Why roll back the full tuple instead of just the model?"**
   *Model answer:* Because regressions can arise from the *interaction* between components — e.g. a new prompt combined with a new dataset distribution — not from any single component in isolation. Rolling back only the model may leave the actual fault (in the prompt or dataset) untouched, and the incident may persist or resurface.

6. **"How would you decide between MLflow, Weights & Biases, and building a custom registry?"**
   *Model answer:* Base the decision on the team's operating model, not tool popularity: release cadence, governance/approval load, number of stakeholders, centralized vs. decentralized deployment, and required lineage depth. MLflow is the strong open-source default (and now covers prompts too via MLflow 3's GenAI Prompt Registry). W&B/Comet fit teams already standardized on their tracking tools. Custom registries make sense only when scale or compliance needs exceed what off-the-shelf tools support — e.g. Uber's Michelangelo Gallery.

7. **"What's the role of canary releases in ML/LLM deployment, specifically vs. traditional software canaries?"**
   *Model answer:* Canaries expose a new release to a small percentage of production traffic before full rollout, bounding blast radius while observing quality, latency, and stability metrics on real input distributions. In ML/LLM systems this matters even more than in traditional software because some failure modes (drift, edge-case inputs, prompt/data interactions) only appear on real production traffic and cannot be fully captured by offline evaluation sets.

8. **"What does 'lineage' mean in this context, and why does it make rollback deterministic?"**
   *Model answer:* Lineage is the recorded graph connecting a deployed release to the exact training run, code commit, data snapshot, and prompt version that produced it. When lineage is captured properly, "roll back to the last known good state" becomes a deterministic lookup (query the lineage store for the last validated tuple) instead of a guess about which components belong together — which is what makes rollback reliable under incident pressure.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

Semantic versioning in ML/LLM systems extends beyond code to models, prompts, and datasets, because production behavior depends on all three. The deployment triple — logged per inference — is what makes reproducibility, debugging, and compliance possible. A model registry (MLflow being the current open-source default, with W&B and Comet as SaaS-native alternatives, and custom systems reserved for exceptional scale/compliance needs) turns registered artifacts into governed, promotable assets via code-driven promotion gates rather than manual judgment. When something goes wrong, rollback must restore the entire last-known-good tuple — not the model alone — using lineage data that makes "what was live and validated before" a deterministic query rather than a guess. Canary releases bound the blast radius of any release before it reaches full production traffic.

### Key Takeaways

- Version the **triple** (model + prompt + dataset), not just the model.
- MLflow's fixed Stages are **deprecated** (since 2.9) — use aliases and tags.
- Promotion must be **code-driven and criteria-gated**, never manual.
- Rollback restores the **full tuple atomically**, using **lineage**, targeting the **last known good** state — not simply the prior version.
- Canary rollouts bound blast radius and catch regressions that only appear on real production traffic.
- Registry choice follows the **team's operating model** (cadence, governance, centralization, lineage depth) — not tool popularity.

### Production Checklist

- [ ] Every model, prompt, and dataset has an explicit semantic version.
- [ ] Every inference logs the full deployment triple (model version, prompt version, dataset version).
- [ ] A `release.yaml`-style manifest defines the deployable triple, compatibility rules, and rollout state as one auditable source of truth.
- [ ] Model registry in place (MLflow/W&B/Comet/custom) with lineage metadata (run, metrics, data version, environment) attached to every registered version.
- [ ] Promotion is gated by automated, versioned criteria (absolute floor, relative regression guard, latency/cost gate, and safety/eval gate for LLMs) — no manual promotion path exists in production.
- [ ] Serving code resolves aliases (`@champion`) rather than hardcoded version numbers.
- [ ] Canary rollout stage exists before 100% traffic exposure, with automatic rollback triggers on threshold breach.
- [ ] Rollback runbook defines: what counts as last known good, what metrics trigger rollback, who/what can authorize it, how restoration happens mechanically, and how the event is logged.
- [ ] Rollback restores the full tuple (model + prompt + dataset), never the model in isolation.
- [ ] Every rollback event is recorded with reason, target versions, timestamp, and operator for postmortem review.
- [ ] Business-quality dashboards (accuracy, hallucination rate, safety-eval scores) are wired into on-call alerting, not just infrastructure health checks.

---

## 9. Further Reading

Detailed citations, official documentation links, notable repositories, and recommended videos/books for this module are collected in the companion files in this same folder — **`references.md`**, **`videos.md`**, **`books.md`**, and **`github.md`** — rather than repeated at length here.
