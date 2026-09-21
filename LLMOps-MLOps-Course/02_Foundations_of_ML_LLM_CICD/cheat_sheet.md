# Module 02 Cheat Sheet — Foundations of ML and LLM CI/CD

One-page reference. See `tutorial.md` for full explanations.

---

## The One Sentence That Matters Most

```
Traditional DevOps CI/CD:   behavior = f(code)                → test the code is enough
ML / LLM CI/CD:             behavior = f(code, data, model, [prompt/retrieval config])
                            → data, model, and prompt can each regress with ZERO code change
```

**A green pipeline (infra healthy, deploy succeeded) is NOT the same as a good model.**
That gap — "silent failure" — is the entire reason this module's gates exist.

---

## The Three Extra Gates

| # | Gate | Catches | Tool (default choice) | Fails via |
|---|---|---|---|---|
| 1 | **Data validation** | Schema drift, nulls, partial loads, population drift | Pandera (schema) + SciPy/PSI (drift) + custom row-count | `sys.exit(1)` |
| 2 | **Model evaluation** | A model that trained "successfully" but is objectively worse | MLflow (`mlflow.log_metric` / `mlflow.evaluate`) | Custom threshold + `sys.exit(1)` |
| 3 | **Response/prompt quality** (LLM only) | Hallucination, incoherence, unsafe output from a new prompt/model/RAG version | LLM-as-judge vs. a fixed golden eval set | Custom threshold + `sys.exit(1)` |

**Golden rule:** every gate emits **structured JSON logs** *and* **exits non-zero on
failure**. Logging a warning and continuing is not a gate — it's a comment.

---

## DevOps vs. MLOps vs. LLMOps — Nested Scopes

```
 DevOps  release unit:   [ code ]
 MLOps   release unit:   [ code | data | model artifact ]
 LLMOps  release unit:   [ code | model/API version | prompt template | retrieval/index config | eval-set version ]
```

| Dimension | DevOps | MLOps | LLMOps |
|---|---|---|---|
| Tested | Unit/integration/static analysis | + data validation + model eval vs. baseline | + response/prompt quality, hallucination, safety, retrieval relevance |
| Can regress w/ zero code change | Never | Data drift, stale training data | + FM version bump, index rebuild, silent prompt edit |
| Release cadence driver | Feature velocity | Data freshness / drift schedule | + FM release cycle + prompt iteration (often fastest-changing) |

---

## Schema Validation (Pandera) — Snippet

```python
import pandera.pandas as pa
from pandera.pandas import Column, DataFrameSchema, Check

ChurnSchema = DataFrameSchema({
    "customer_id": Column(str, nullable=False, unique=True),
    "transaction_count_30d": Column(int, Check.ge(0)),
    "avg_spend_usd": Column(float, Check.ge(0.0)),
    "churn_label": Column(int, Check.isin([0, 1])),
}, strict=False)

try:
    ChurnSchema.validate(df, lazy=True)   # lazy=True: collect ALL failures in one pass
except pa.errors.SchemaErrors as err:
    print(err.failure_cases.to_dict(orient="records"))
    sys.exit(1)
```

## Drift Validation — KS-test and PSI

```python
from scipy.stats import ks_2samp
stat, p_value = ks_2samp(baseline[col], current[col])
if p_value < 0.05:      # distributions likely different -> drift
    sys.exit(1)
```

| PSI value | Interpretation |
|---|---|
| < 0.10 | Stable — no significant shift |
| 0.10 – 0.25 | Slight drift — watch, don't necessarily block |
| > 0.25 | Significant drift — investigate, likely block |

---

## Model Evaluation Gate (MLflow) — Snippet

```python
import mlflow
from sklearn.metrics import roc_auc_score

auc = roc_auc_score(y, preds)
with mlflow.start_run(run_name="churn_eval"):
    mlflow.log_metric("auc", auc)
    if auc < baseline_auc:          # e.g. baseline_auc * 0.99 for a small tolerance
        sys.exit(1)                  # do NOT register — prior model keeps serving
```

---

## Fail-Fast Ordering Rule

```
CHEAPEST, MOST-LIKELY-TO-CATCH-A-PROBLEM  ──────────────────►  MOST EXPENSIVE

row_count check (ms)  →  schema check (Pandera, ms-sec)  →  drift check (sec)
     →  TRAIN (minutes-hours)  →  model eval (sec-min)  →  (LLM) response quality (min, $)
```
Never spend an expensive step's compute validating what a cheap step could have caught.

---

## GitHub Actions Skeleton (Trigger + Gates + Idempotency)

```yaml
on:
  schedule: [{cron: "0 3 * * *"}]
  workflow_dispatch: {}
concurrency: {group: churn-retrain, cancel-in-progress: false}

jobs:
  retrain:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - run: python pipeline/fetch_data.py --out data/latest.parquet
      - name: "GATE 1 — data validation"
        run: python pipeline/validate_data.py --input data/latest.parquet   # sys.exit(1) on fail
      - run: python pipeline/train.py --input data/latest.parquet --dataset-hash-out artifacts/dataset.hash
      - name: "GATE 2 — model evaluation"
        run: python pipeline/evaluate_model.py --baseline-auc 0.82          # sys.exit(1) on fail
      - if: success()
        run: python pipeline/register_model.py --dataset-hash artifacts/dataset.hash
      - if: success()
        run: curl -X POST ... "${{ secrets.SLACK_WEBHOOK_URL }}"           # notify success
      - if: failure()
        run: curl -X POST ... "${{ secrets.SLACK_WEBHOOK_URL }}"           # notify failure
```
**Idempotency:** hash the input dataset (SHA-256) → tag the MLflow registration with it
→ re-running against the same window is a safe no-op, not a duplicate "new" version.

---

## Decision Rule — Which Gate Tooling to Reach For

| Question | Reach for |
|---|---|
| Is the data structurally what I expect? | Pandera `DataFrameSchema` (few cols → hand-rolled `assert`) |
| Has the distribution changed? | 1-2 features → `scipy.stats.ks_2samp` / hand-rolled PSI. Many features → Evidently AI Test Suites |
| Is there enough data? | Custom `assert len(df) >= MIN_ROWS` |
| Is the newly trained model good? | `mlflow.log_metric` + manual threshold (default) → `mlflow.evaluate()` as harness matures |
| Is the LLM's output good? | Small golden eval set (10s–100s of prompts) + LLM-as-judge, versioned like a test suite |

## GH Actions: One Job vs. Separate Jobs

| | Single job, sequential steps | Separate jobs w/ `needs:` |
|---|---|---|
| Use when | Default / starting point | Gate 3 is expensive, or gates need different credential scopes |
| Pro | Simplest; shared workspace; failure auto-skips later steps | Per-job runner size/timeout/secrets; parallelizable |
| Con | One timeout/runner for everything | Needs `upload-artifact`/`download-artifact` plumbing |

---

## Common Mistakes (Quick-Scan List)

- Treating "unit tests pass" as "the model is fine"
- Validating schema but not distribution (or vice versa) — they catch different things
- Logging a failure as a warning instead of `sys.exit(1)` — not enforced, just theater
- A static evaluation threshold nobody owns or revisits
- No fail-fast ordering — training before checking the data has the right columns
- No idempotency — retries can silently create a different "same-day" model
- Conflating "deployed cleanly" with "business metrics are fine" (silent failure)
- Over-gating exploratory/research notebooks with full production ceremony
- For LLMs: treating "API returned 200" as "the response was good"

---

## Key Terms, One Line Each

- **Quality gate** — pass/fail checkpoint; implemented as a step that exits non-zero on
  failure.
- **CT (Continuous Training)** — automatically retraining on new data/drift, no human
  kickoff; depends on the *schedule* trigger.
- **PSI** — scalar summarizing feature distribution shift, baseline vs. current.
- **Idempotency** — same input → same output (verifiable model hash); safe retries.
- **Silent failure** — infra green, model quality silently degraded — this module's
  central risk.
- **Release manifest** — the LLMOps release unit made explicit/diffable: code commit +
  model/API version + prompt version + retrieval index version + eval-set version.
