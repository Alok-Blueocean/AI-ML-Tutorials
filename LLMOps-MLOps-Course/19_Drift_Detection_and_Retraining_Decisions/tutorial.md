# Module 19 — Drift Detection and Retraining Decisions

> "A model that was correct yesterday is not guaranteed to be correct today — the world moved and nobody told the model." This module is about building the instruments that notice the world moved, and the discipline that decides what to do about it.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Distinguish **data (input) drift**, **behavioral (output) drift**, and **quality (performance) drift** in both classical ML and LLM systems, and explain why conflating them leads to wrong operational responses.
2. Compute and interpret the three workhorse statistical drift tests — **Population Stability Index (PSI)**, **Kullback-Leibler (KL) divergence**, and the **Kolmogorov-Smirnov (KS) test** — including which one to reach for on which kind of data, and what their numeric thresholds actually mean.
3. Detect drift in high-dimensional and unstructured inputs (text, embeddings) using embedding-distance and distributional methods, when PSI/KS on raw features no longer applies.
4. Build a production drift-monitoring pipeline with separate jobs for input drift and output/behavioral drift, so that troubleshooting can localize *where* a problem originates.
5. Design a drift dashboard (PSI, response length, refusal rate, evaluation score) and alerting rules with tuned thresholds that avoid both alert fatigue and late detection, including Slack-based notification.
6. Apply a **decision matrix** that maps combinations of drift signals to the correct action — retrain, update the prompt/pipeline, roll back, or investigate further — instead of retraining reflexively every time an alert fires.
7. Run a structured **incident runbook** (validate → assess impact → review recent changes → act → document) for a drift alert, the same way an SRE runs an incident for a latency page.
8. Compare and choose among drift-tooling options — Evidently AI, NannyML, whylogs/WhyLabs, Arize Phoenix, Deepchecks, Great Expectations — based on what each is actually built to detect (data drift vs. label-free performance estimation vs. LLM-trace-centric observability).
9. Identify common drift-monitoring failure modes (drift-without-impact false alarms, silent quality decay with no distributional signal, threshold-tuning theater, retraining as a reflex) and correct them.

### Prerequisites

- Module 14 (Observability, OpenTelemetry, and Production Monitoring) — this module assumes you already have traces/metrics/logs flowing out of a live service; drift monitoring consumes that same telemetry stream plus periodic batch snapshots.
- Module 09-11 (Prompt Lifecycle, Evaluation Datasets, LLM-as-Judge) — drift monitoring's "quality" signal is very often an offline/online evaluation score computed by exactly the judges and datasets built in those modules.
- Module 13 (Experiment Tracking and MLflow) — retraining decisions in this module route back into the registry/versioning workflow from Module 03 and 13.
- Basic probability/statistics: histograms, probability distributions, what a p-value is (a short refresher is included inline — no proof-level stats background required).
- Comfortable with pandas, NumPy, and reading a bit of SciPy.

### Key Terminology

| Term | One-line definition |
|---|---|
| **Data drift (input/covariate drift)** | A change in the statistical distribution of the model's input features or, for an LLM, the distribution of incoming user queries — the world feeding the model changed, independent of whether the model's outputs are wrong yet. |
| **Concept drift** | A change in the true relationship between inputs and the correct output (P(Y\|X) changes) — the same input now *should* map to a different answer, which is a strictly harder problem than input drift because it can be invisible until labels/feedback arrive. |
| **Behavioral drift** | For generative/LLM systems specifically: a change in the model's *outputs* — length, structure, tone, refusal rate — even when the input distribution looks stable. Often the first observable symptom of an upstream model, prompt, or pipeline change. |
| **Quality drift (performance drift)** | A drop in a directly-measured correctness/quality signal (accuracy, F1, an LLM-judge score, a business KPI) — the signal that tells you drift actually *matters*, as opposed to merely being present. |
| **Population Stability Index (PSI)** | A single-number distance metric between a baseline distribution and a current distribution, binned into buckets, widely used (originally in credit-risk scoring) as an interpretable "how much has this distribution moved" score. |
| **Kullback-Leibler (KL) divergence** | An information-theoretic, asymmetric measure of how one probability distribution diverges from a second, reference distribution — the mathematical basis PSI is a discretized/symmetrized approximation of. |
| **Kolmogorov-Smirnov (KS) test** | A nonparametric statistical test comparing two *continuous* one-dimensional distributions via the maximum distance between their empirical CDFs, producing a test statistic and a p-value. |
| **Covariate shift** | A specific, common form of data drift where P(X) changes but P(Y\|X) stays the same — the input distribution moved, but the "rules of the world" didn't, which is the more benign and more retrain-friendly case. |
| **Label-free / reference-free performance estimation** | Estimating model performance (e.g., accuracy) in production *without waiting for ground-truth labels*, by modeling how confidence/uncertainty relates to correctness on labeled historical data — the core trick behind NannyML's CBPE. |
| **Refusal rate** | The fraction of LLM responses that decline to answer, hedge excessively, or trigger a safety/guardrail refusal — an LLM-specific behavioral-drift signal with no classical-ML analogue. |
| **Decision matrix** | A structured mapping from *which signals fired, in what combination* to *which action to take* (retrain / prompt-fix / rollback / investigate) — the artifact that turns "something drifted" into "here is what we do about it." |
| **Runbook** | A written, repeatable, step-by-step procedure followed when a specific alert fires — see Module 14 §3.6; this module supplies the drift-specific runbook. |
| **Alert fatigue** | The failure mode where too many low-value or non-actionable alerts train engineers to ignore all alerts, including real ones — directly relevant to threshold tuning in §3.4. |

---

## 2. Why This Topic Matters — Where It Fits in the MLOps/LLMOps Lifecycle

Every module up to this point in the "monitoring trio" (Module 14 for raw observability, Module 18 for agent-level tracing, this one) has been building toward the same underlying question, asked at increasing levels of abstraction: **is the system still doing the right thing, and how would we know if it stopped?** Module 14 gives you the raw signals (latency, cost, errors, traces). This module answers the harder, quieter question those signals don't automatically answer: **the service is up, latency is fine, no exceptions are thrown — is the model's actual *behavior* still correct?**

```
 Live traffic --> Module 14 (traces/metrics/logs) --> "the system is up and fast"
                                                              |
                                                              v
                                        +---------------------------------------+
                                        |         THIS MODULE (Module 19)        |
                                        |  Data drift / Behavioral drift /       |
                                        |  Quality drift -- "is it still RIGHT?" |
                                        +---------------------------------------+
                                                              |
                                     drift confirmed + evidence gathered ------> |
                                                              v
                                     Retrain (Module 03/13 registry + Module 13
                                     MLflow) | Prompt/pipeline fix (Module 07-09)
                                     | Rollback (Module 03) | Investigate further
```

This is a different failure mode than anything Module 14 catches, and it is why it deserves its own chapter rather than being one more panel on the same dashboard. A classical service failure is loud: an exception, a 500, a timeout. **A drifted model fails silently.** It returns HTTP 200. It returns fluent, well-formatted, confident output. Nothing in the request/response cycle looks wrong. The only way to know something changed is to deliberately compare *today's* distribution of inputs, outputs, and quality against a *trusted baseline* — which is precisely what this module teaches how to build.

Two things make this materially different for LLM systems compared to the classical, tabular-feature ML this discipline originally grew up around (PSI's roots are in credit-risk scorecards monitoring FICO-score-like features):

**1. The "distribution" being monitored is no longer a handful of numeric features.** A credit model's input drift check is "did the distribution of `debt_to_income_ratio` shift?" An LLM's input is free text of unbounded variety — you cannot bin "the user's query" into ten buckets the way you bin a credit score. This forces a shift from raw-feature PSI/KS toward **embedding-space** distance measures, and a shift from monitoring *inputs alone* toward also monitoring the **outputs** (length, structure, refusal rate, tone) as first-class drift signals in their own right — there usually is no equivalent for a classical regression model, whose "output" is just a number that either matches the label or doesn't.

**2. Ground-truth labels are slower, sparser, or entirely absent.** A fraud model gets a confirmed chargeback label days later. A chat assistant may never get an explicit "was this answer correct" label at all — only weak proxies (thumbs up/down, session abandonment, an asynchronous LLM-judge score sampled after the fact). This is precisely why label-free performance-estimation tools like NannyML's CBPE exist, and why an LLM-judge score (Module 11) so often plays the role that a "quality/performance drift" metric plays for a classical model.

Framed as the questions a senior MLOps/LLMOps engineer must be able to answer for any live model:

| Question | Signal that answers it | Covered in |
|---|---|---|
| "Did the *inputs* we're receiving change?" | Data/input drift (PSI, KS, KL, embedding distance) | §3.1, §3.2 |
| "Did the *model's behavior* change, even if inputs look the same?" | Behavioral drift (output length/structure/tone, refusal rate) | §3.1, §3.4 |
| "Does any of this actually matter — is quality worse?" | Quality/performance drift (eval score, accuracy, business KPI) | §3.1, §3.5 |
| "Given what fired, what should we *do*?" | Decision matrix | §3.3 |
| "How do we operate this without noise or without missing real incidents?" | Dashboards + tuned alerting | §3.4 |
| "An alert fired — now what, step by step?" | Incident runbook | §3.6 |

---

## 3. Main Concepts

### 3.1 Data Drift vs. Behavioral Drift vs. Quality Drift

#### Theory

The single most important idea in this module, straight from the source material and worth over-stating because it is so often collapsed in practice: **these are three different signals, they can move independently, and the correct response depends on which one (or which combination) fired.**

- **Data (input) drift** — the incoming feature values or user queries look statistically different from the baseline the model was trained/validated on. This is a fact about the *world feeding the model*, not yet a fact about whether the model is wrong. A retail demand model seeing a distribution shift toward higher order volumes in December is input drift that is completely expected and requires no action; the same shift in July might warrant investigation.
- **Behavioral (output) drift** — the model's *own outputs* change in distribution, tone, structure, or length, even when the measurable input distribution looks stable. This is the LLM-era addition to the classical drift taxonomy: a prompt template edit, a silent model-provider upgrade (e.g., a vendor swaps the model behind an API alias), a system-prompt regression, or a tokenizer/context-window change can all shift *what comes out* without shifting what statistically goes in. Classical ML has a weak analogue (a classifier's predicted-class distribution shifting — "prediction drift") but the LLM version is richer: response length, refusal rate, structural conformance (does it still emit valid JSON), tone/sentiment.
- **Quality drift** — a directly measured drop in correctness: accuracy/F1 against eventual labels, an LLM-judge score against a golden set, a business KPI (conversion rate, resolution rate) trending down. This is the signal that answers "does any of this actually matter" — data drift and behavioral drift are both *leading indicators*; quality drift is the *lagging, ground-truth confirmation*.

**Why the distinction changes the response**, which is the core operational payoff of this whole module:

| Signals observed | Most likely cause | Typical response |
|---|---|---|
| Input drift **+** quality drift, together | The world genuinely changed and the model's learned patterns no longer fit it | **Retrain** on recent data |
| Behavioral drift **only** (input looks stable, quality looks stable so far) | A prompt, pipeline, or model-version change altered outputs | **Prompt/pipeline fix**, or roll back the recent change |
| Input drift **+** behavioral drift, together, quality not yet confirmed | Ambiguous — could be a legitimate world change the model is handling differently, or a confound | **Investigate the root cause** before acting — do not guess |
| Quality drift **alone**, with input and output distributions both looking stable | The measurement itself may be broken (labeling lag, judge miscalibration, eval-set staleness), or a subtle concept drift not visible in surface distributions | **Investigate the evaluation pipeline first**, then consider concept drift |

**When each check is or isn't sufficient on its own:** input drift monitoring alone is necessary but not sufficient — a model can be perfectly robust to a given input shift (a well-regularized model doesn't care that `debt_to_income_ratio`'s mean moved from 0.31 to 0.34), so drift-without-quality-impact is common and should not trigger automatic retraining. Conversely, quality drift alone, without any distributional signal to explain it, is a yellow flag that your *drift detectors themselves* may be blind to the actual failure mode — a strong prompt for extending §3.2's toolkit when this happens.

#### Architecture — the three-track pipeline

```
                              Production traffic
                                     |
              +----------------------+----------------------+
              v                                              v
    +-------------------+                          +---------------------+
    |  INPUT DRIFT JOB    |                          |  OUTPUT/BEHAVIORAL   |
    |  (scheduled, e.g.   |                          |  DRIFT JOB            |
    |   hourly/daily)     |                          |  (scheduled, same     |
    |                     |                          |  cadence)             |
    |  recent inputs      |                          |  recent outputs vs.   |
    |  vs. baseline        |                          |  baseline outputs     |
    |  window              |                          |  window                |
    |                     |                          |                       |
    |  PSI / KS / KL on    |                          |  length, structure,   |
    |  features; embedding |                          |  refusal rate, tone   |
    |  centroid distance   |                          |                       |
    |  for text            |                          |                       |
    +-------------------+                          +---------------------+
              |                                              |
              +----------------------+----------------------+
                                     v
                          +-----------------------+
                          |  QUALITY DRIFT JOB      |
                          |  (sampled eval / judge   |
                          |  score, or label-free    |
                          |  estimation e.g. NannyML |
                          |  CBPE, on a slower        |
                          |  cadence -- daily/weekly) |
                          +-----------------------+
                                     |
                                     v
                          +-----------------------+
                          |   DECISION MATRIX        |
                          |   (§3.3) combines all     |
                          |   three signals into      |
                          |   one recommended action  |
                          +-----------------------+
```

Keeping these three jobs **separate** — rather than one monolithic "drift score" — is itself the design decision that makes troubleshooting fast: when only the output-drift job fires, you already know to look at recent prompt/pipeline changes before touching training data at all.

#### Examples

**Beginner** — a plain dataclass-based signal bundle, matching the source material's "track input drift, output drift, quality drift in the pipeline" framing:

```python
from dataclasses import dataclass

@dataclass
class DriftSignals:
    input_drift_score: float      # e.g., PSI on key features / embedding distance
    input_drift_flagged: bool
    behavioral_drift_flagged: bool  # e.g., response length or refusal rate out of range
    quality_score: float           # e.g., eval/judge score, 0-1
    quality_drop_flagged: bool

def classify_drift(signals: DriftSignals) -> str:
    """Minimal version of the decision logic -- expanded fully in §3.3."""
    if signals.input_drift_flagged and signals.quality_drop_flagged:
        return "RETRAIN"
    if signals.behavioral_drift_flagged and not signals.quality_drop_flagged:
        return "PROMPT_OR_PIPELINE_FIX"
    if signals.input_drift_flagged and signals.behavioral_drift_flagged:
        return "INVESTIGATE"
    if signals.quality_drop_flagged:
        return "INVESTIGATE"
    return "NO_ACTION"
```

**Intermediate** — computing the behavioral signals from raw response logs (no drift math yet — that's §3.2):

```python
import re
import pandas as pd

REFUSAL_PATTERNS = re.compile(
    r"\b(i can'?t help|i'?m unable to|as an ai|i cannot assist|i won'?t)\b", re.IGNORECASE
)

def compute_behavioral_metrics(responses: pd.Series) -> dict:
    lengths = responses.str.split().str.len()
    refusals = responses.str.contains(REFUSAL_PATTERNS)
    return {
        "mean_response_len_tokens": lengths.mean(),
        "p95_response_len_tokens": lengths.quantile(0.95),
        "refusal_rate": refusals.mean(),
    }
```

**Production-grade** — the three jobs wired as independent, schedulable functions that write to a shared metrics store (shown fully assembled with alerting in §3.4-§3.5).

---

### 3.2 Statistical Drift Detection: PSI, KL Divergence, KS Test, and Embedding Distance

#### Theory

All input-drift detection reduces to one question, made rigorous three different ways: **given a baseline sample and a current sample, how different are their underlying distributions, and is that difference large enough to matter?**

**Population Stability Index (PSI).** Bin both the baseline and current distributions into the same buckets (deciles are typical for continuous features), compute each bucket's population *proportion* in both samples, and sum:

```
PSI = sum over bins i of  (current_i - baseline_i) * ln(current_i / baseline_i)
```

PSI is popular in production specifically because it collapses an entire distribution comparison into one interpretable number with widely-used, industry-standard rule-of-thumb thresholds (originating in credit-risk scorecard monitoring):

| PSI value | Interpretation |
|---|---|
| < 0.1 | No significant population change |
| 0.1 – 0.25 | Moderate change — worth watching |
| > 0.25 | Significant population change — investigate |

These thresholds are a starting heuristic, not physical law — always validate them against your own feature's historical noise floor (see §3.4) before trusting them blindly.

**KL divergence.** The information-theoretic quantity PSI is actually built from — PSI is precisely the *sum of two KL divergences* (baseline‖current and current‖baseline), which is why PSI is often described as a "symmetrized KL divergence." Raw KL divergence:

```
D_KL(P || Q) = sum over x of  P(x) * ln( P(x) / Q(x) )
```

is asymmetric (`D_KL(P‖Q) ≠ D_KL(Q‖P)`), undefined when `Q(x) = 0` for any `x` where `P(x) > 0` (a real practical hazard with sparse bins — always add a small smoothing epsilon), and has no fixed interpretable scale the way PSI's rule-of-thumb bands do. Use raw KL when you specifically need the asymmetry (e.g., "how surprised would a model trained on baseline P be, if reality is now Q") or when working with full continuous density estimates rather than discretized bins; use PSI when you want an interpretable, comparable, bin-based production monitoring metric.

**Kolmogorov-Smirnov (KS) test.** For continuous, one-dimensional data, KS compares the two samples' *empirical cumulative distribution functions* (ECDFs) directly, without binning, and returns both a test statistic (the max vertical gap between the two ECDFs) and a p-value:

```
D = max_x | F_baseline(x) - F_current(x) |
```

KS's advantage over PSI is that it doesn't require you to choose bin boundaries (a real source of PSI instability with small samples) and gives you a formal p-value with a null hypothesis ("these two samples come from the same distribution"). Its disadvantage is that it is a *hypothesis test*, not a magnitude score — with enough samples (production traffic volumes routinely provide "enough"), KS will report a statistically significant p-value for even a trivially small, operationally meaningless shift. **This is the single most common KS-test mistake in production drift monitoring: treating a tiny p-value as "big drift" when it may just mean "big sample size."** Always look at the KS statistic's magnitude and a practical effect-size threshold alongside the p-value, not the p-value alone.

**Choosing among the three:**

| Method | Data type | Gives a magnitude? | Gives a formal p-value? | Sensitive to sample size? | Typical production use |
|---|---|---|---|---|---|
| **PSI** | Binned continuous or categorical | Yes (interpretable bands) | No | Moderately (small samples destabilize bin proportions) | Default choice for tabular feature monitoring; dashboards |
| **KL divergence** | Probability distributions (binned or modeled) | Yes, but no fixed scale | No | Sensitive to zero-probability bins | Research/internal comparisons; basis for PSI |
| **KS test** | Continuous, 1-D | Yes (D statistic) | Yes | Very sensitive — large N inflates significance | Confirmatory test alongside PSI; smaller/exploratory samples |
| **Chi-squared test** | Categorical | Yes | Yes | Sensitive to sparse categories | Categorical feature drift (not detailed here — same family as KS conceptually) |
| **Embedding/centroid distance** | Text, images, any unstructured input | Yes (a distance, e.g. cosine) | No (not a classical hypothesis test) | Depends on embedding model stability | LLM query drift, RAG corpus drift |

**Embedding-space drift for text.** Raw feature drift tests don't apply to free text — there is no "bin" for a user's query. The production pattern: embed a baseline sample of queries and a current sample of queries with the same embedding model, then compare either (a) the distance between the two samples' **centroid** vectors, or (b) a distributional test (e.g., Maximum Mean Discrepancy, or simply PSI/KS applied to a lower-dimensional projection such as the first few PCA components of the embeddings). Centroid cosine distance is the cheap, interpretable, "did the *center of mass* of what people are asking about move" signal; it will miss a shift that only affects the *spread* (e.g., queries becoming much more varied in topic while the average stays put) — for that, watch the embedding-space variance/dispersion alongside the centroid.

#### Architecture

```
        Baseline window                         Current window
   (e.g., last "known-good" 30 days,       (e.g., last 24h / 7d,
    frozen at last validated deploy)         rolling)
              |                                       |
              v                                       v
   +---------------------------+       +---------------------------+
   |  Numeric/categorical       |       |  Numeric/categorical       |
   |  features -> bin into      |       |  features -> bin into      |
   |  same deciles/categories   |       |  same deciles/categories   |
   +---------------------------+       +---------------------------+
              |                                       |
              +------------------+--------------------+
                                 v
                      PSI per feature, KS per feature
                                 |
              +------------------+--------------------+
              v                                       v
   +---------------------------+       +---------------------------+
   |  Free-text queries          |       |  Free-text queries          |
   |  -> embed (same model)       |       |  -> embed (same model)       |
   +---------------------------+       +---------------------------+
              |                                       |
              +------------------+--------------------+
                                 v
                    Centroid cosine distance,
                    embedding-space dispersion delta
                                 |
                                 v
                   Feature-level + query-level drift report
                   (per-feature PSI/KS table + text drift score)
```

#### Code — PSI, KL divergence, and KS test from first principles

```python
import numpy as np
from scipy.stats import ks_2samp

def compute_psi(baseline: np.ndarray, current: np.ndarray, n_bins: int = 10, epsilon: float = 1e-4) -> float:
    """Population Stability Index between two 1-D numeric arrays.
    Bin edges are derived from the BASELINE's quantiles -- this is important:
    using current-data quantiles would make PSI compare a distribution to
    itself and always understate drift.
    """
    bin_edges = np.quantile(baseline, np.linspace(0, 1, n_bins + 1))
    bin_edges[0], bin_edges[-1] = -np.inf, np.inf  # catch out-of-range values in current

    baseline_counts, _ = np.histogram(baseline, bins=bin_edges)
    current_counts, _ = np.histogram(current, bins=bin_edges)

    baseline_pct = baseline_counts / len(baseline) + epsilon  # epsilon avoids div-by-zero / ln(0)
    current_pct = current_counts / len(current) + epsilon

    psi = np.sum((current_pct - baseline_pct) * np.log(current_pct / baseline_pct))
    return float(psi)


def compute_kl_divergence(baseline: np.ndarray, current: np.ndarray, n_bins: int = 10, epsilon: float = 1e-4) -> float:
    """Raw KL divergence D_KL(current || baseline) over the same binning as PSI.
    Asymmetric: swapping arguments gives a different number -- report the
    direction you actually care about (usually "how far has reality moved
    from what the model expects", i.e. current relative to baseline).
    """
    bin_edges = np.quantile(baseline, np.linspace(0, 1, n_bins + 1))
    bin_edges[0], bin_edges[-1] = -np.inf, np.inf

    baseline_counts, _ = np.histogram(baseline, bins=bin_edges)
    current_counts, _ = np.histogram(current, bins=bin_edges)

    p = current_counts / len(current) + epsilon
    q = baseline_counts / len(baseline) + epsilon
    return float(np.sum(p * np.log(p / q)))


def compute_ks_test(baseline: np.ndarray, current: np.ndarray) -> dict:
    """Kolmogorov-Smirnov two-sample test. Report BOTH statistic and p-value --
    with large production sample sizes, p-value alone will look "significant"
    for trivial shifts (see theory section)."""
    statistic, p_value = ks_2samp(baseline, current)
    return {"ks_statistic": float(statistic), "p_value": float(p_value)}


def psi_verdict(psi: float) -> str:
    if psi < 0.1:
        return "stable"
    elif psi < 0.25:
        return "moderate_drift"
    return "significant_drift"
```

```python
# Embedding-space drift for free text (LLM query drift)
import numpy as np

def embedding_centroid_drift(baseline_embeddings: np.ndarray, current_embeddings: np.ndarray) -> float:
    """Cosine distance between the mean embedding vector of two samples.
    Cheap, interpretable "did the center of what's being asked move" signal.
    baseline_embeddings / current_embeddings: shape (n_samples, embedding_dim),
    produced by the SAME embedding model/version for both windows.
    """
    baseline_centroid = baseline_embeddings.mean(axis=0)
    current_centroid = current_embeddings.mean(axis=0)

    cosine_sim = np.dot(baseline_centroid, current_centroid) / (
        np.linalg.norm(baseline_centroid) * np.linalg.norm(current_centroid) + 1e-9
    )
    return float(1 - cosine_sim)  # 0 = identical direction, up to 2 = opposite


def embedding_dispersion_delta(baseline_embeddings: np.ndarray, current_embeddings: np.ndarray) -> float:
    """Did the SPREAD of topics change, even if the centroid didn't?
    Average distance of each point to its own sample's centroid, compared
    across the two windows.
    """
    def avg_dispersion(embeddings: np.ndarray) -> float:
        centroid = embeddings.mean(axis=0)
        return float(np.mean(np.linalg.norm(embeddings - centroid, axis=1)))

    return avg_dispersion(current_embeddings) - avg_dispersion(baseline_embeddings)
```

**Beginner example** — run PSI/KS on one feature of a synthetic dataset:

```python
import numpy as np

np.random.seed(0)
baseline_income = np.random.normal(loc=50000, scale=12000, size=5000)
current_income = np.random.normal(loc=58000, scale=13000, size=1200)   # a real shift

print("PSI:", compute_psi(baseline_income, current_income))
print("KS:", compute_ks_test(baseline_income, current_income))
# PSI will land solidly in "moderate" to "significant" territory; KS will
# report a tiny p-value because the shift is real and the sample is large.
```

**Intermediate example** — a per-feature drift report over a whole feature table:

```python
import pandas as pd

def feature_drift_report(baseline_df: pd.DataFrame, current_df: pd.DataFrame, numeric_cols: list[str]) -> pd.DataFrame:
    rows = []
    for col in numeric_cols:
        psi = compute_psi(baseline_df[col].dropna().values, current_df[col].dropna().values)
        ks = compute_ks_test(baseline_df[col].dropna().values, current_df[col].dropna().values)
        rows.append({
            "feature": col,
            "psi": round(psi, 4),
            "psi_verdict": psi_verdict(psi),
            "ks_statistic": round(ks["ks_statistic"], 4),
            "ks_p_value": ks["p_value"],
        })
    return pd.DataFrame(rows).sort_values("psi", ascending=False)
```

**Production-grade example** — combining tabular and text drift into one report, ready to feed the decision matrix in §3.3 (shown assembled end-to-end in §3.5).

#### Common mistakes with statistical drift tests

- **Computing bin edges from the current window instead of the baseline.** This silently makes PSI understate drift because the "current" bins are, by construction, evenly populated relative to themselves.
- **Trusting a KS p-value alone at production sample sizes.** Pair it with the KS statistic's magnitude or a minimum practically-meaningful effect size.
- **Applying PSI/KS to raw free text.** These tests are for numeric/categorical/binned data — text needs embedding-space methods instead.
- **Ignoring epsilon/zero-probability bins in KL/PSI**, causing `ln(0)` or division-by-zero crashes in production jobs at the worst possible time (during an actual large shift, when some bin genuinely goes to zero).
- **Recomputing a "baseline" from a rolling recent window instead of freezing it at the last validated deploy.** A rolling baseline drifts *with* the data, silently raising your own detection threshold over time — the textbook way to make a monitor go blind exactly when it matters ("baseline creep").

---

### 3.3 The Decision Matrix — Retrain vs. Prompt-Fix vs. Rollback vs. Investigate

#### Theory

The source material's central operational insight, worth stating as plainly as possible: **a drift alert is not an instruction to retrain. It is an instruction to look at evidence and choose one of several possible actions.** Automatic, reflexive retraining on every drift signal is expensive (compute, engineering time, risk of shipping a *worse* model trained on a small or biased recent window), slow to close the loop compared to a prompt fix, and often simply the wrong tool — many behavioral-drift incidents are caused by a pipeline bug or a prompt regression that retraining a model cannot fix at all.

The decision matrix, combining both source transcripts' framings into one table:

| Input drift | Behavioral drift | Quality drift | Recent change nearby? | Recommended action |
|---|---|---|---|---|
| High | — | Dropping | — | **Retrain** — the world changed and the model's learned patterns no longer fit it |
| Low/none | High | Not (yet) dropping | Yes (prompt/model/pipeline deploy) | **Roll back** the recent change, or **update the prompt/pipeline** if the change was intentional and just needs tuning |
| Low/none | High | Not (yet) dropping | No recent change identified | **Update prompt/pipeline** cautiously, but first **investigate** why behavior moved with no apparent cause (silent upstream model swap, provider-side update) |
| High | High | Unknown/unclear | — | **Investigate root cause first** — do not act until you know whether this is a benign world change the model handles fine, or a real problem |
| Low/none | Low/none | Dropping | — | **Investigate the evaluation/measurement pipeline** — labeling lag, judge miscalibration, golden-set staleness — before assuming concept drift |
| High | Low/none | Not dropping | — | **No action / monitor** — this is often benign covariate shift the model is robust to (e.g., expected seasonality) |

**Why "investigate first" is itself a first-class action, not a cop-out:** when both input and behavioral drift fire together without a clear quality signal yet, acting on either "retrain" or "roll back" is a guess with real cost attached — a wrong guess wastes the retraining budget or reverts a change that was actually a genuine improvement responding correctly to a real shift. The investigation step (root-cause review, ideally under 30-60 minutes for an experienced on-call engineer with good dashboards) converts a guess into an evidence-based decision, which is exactly the principle the source material calls out: **"decisions should be based on measurable evidence, not guesswork."**

**When to lean toward retraining specifically:** input drift that is (a) large by your PSI/KS thresholds, (b) sustained across multiple monitoring windows (not a one-off blip), and (c) correlated with a real, confirmed quality drop against ground truth or a trusted judge. All three together is the strongest retrain signal this framework has.

**When retraining is very likely the *wrong* answer:** any drift that appeared in lockstep with a known deploy timestamp (prompt edit, model-version bump, pipeline config change, upstream vendor update). Retraining a model cannot fix a broken prompt or a regressed pipeline — check "what changed recently" before touching training data, every time.

#### Architecture

```
                     Drift alert fires (any of §3.1's three signals)
                                        |
                                        v
                     +-----------------------------------+
                     |   STEP 1: What fired?               |
                     |   input / behavioral / quality /     |
                     |   some combination                    |
                     +-----------------------------------+
                                        |
                     +-----------------------------------+
                     |   STEP 2: Any recent change?         |
                     |   check deploy log / prompt registry  |
                     |   / model version / pipeline config   |
                     |   (Module 03/13 registries answer this)|
                     +-----------------------------------+
                                        |
                +----------+-----------+----------+-----------+
                v          v           v            v
           RETRAIN   PROMPT/PIPELINE  ROLLBACK   INVESTIGATE
           (Module   FIX (Module      (Module    FURTHER
           03/13)    07-09)           03)        (§3.6 runbook)
```

#### Code — the decision matrix as executable logic

```python
from dataclasses import dataclass
from enum import Enum

class Action(str, Enum):
    RETRAIN = "retrain"
    PROMPT_OR_PIPELINE_FIX = "prompt_or_pipeline_fix"
    ROLLBACK = "rollback"
    INVESTIGATE = "investigate"
    MONITOR_ONLY = "monitor_only"


@dataclass
class DriftEvidence:
    input_drift_high: bool
    behavioral_drift_high: bool
    quality_dropping: bool
    recent_change_detected: bool   # from deploy/prompt/model registry, e.g. last 24-72h


def decide_action(evidence: DriftEvidence) -> Action:
    """Direct implementation of the decision matrix in §3.3. Order matters --
    more specific / higher-confidence rules are checked first."""

    if evidence.input_drift_high and evidence.quality_dropping:
        return Action.RETRAIN

    if evidence.behavioral_drift_high and evidence.recent_change_detected:
        return Action.ROLLBACK

    if evidence.behavioral_drift_high and not evidence.quality_dropping and not evidence.recent_change_detected:
        return Action.INVESTIGATE

    if evidence.input_drift_high and evidence.behavioral_drift_high:
        return Action.INVESTIGATE

    if evidence.quality_dropping and not evidence.input_drift_high and not evidence.behavioral_drift_high:
        return Action.INVESTIGATE  # measurement pipeline suspect first

    if evidence.input_drift_high:
        return Action.MONITOR_ONLY  # likely benign covariate shift

    return Action.MONITOR_ONLY
```

```python
# Example runs -- read these as worked cases from the theory table above.
print(decide_action(DriftEvidence(True, False, True, False)))    # -> retrain
print(decide_action(DriftEvidence(False, True, False, True)))    # -> rollback
print(decide_action(DriftEvidence(False, True, False, False)))   # -> investigate
print(decide_action(DriftEvidence(True, True, False, False)))    # -> investigate
print(decide_action(DriftEvidence(False, False, True, False)))   # -> investigate
print(decide_action(DriftEvidence(True, False, False, False)))   # -> monitor_only
```

This function is deliberately small and readable — the point of a decision matrix is that an on-call engineer at 2 a.m. can read the *table*, not reverse-engineer a black-box scoring model, which is why this stays rule-based rather than, say, a learned classifier over historical incidents (a tempting but usually premature idea — you rarely have enough labeled past incidents to train that reliably, and an opaque model choosing "retrain vs. rollback" is a worse on-call experience than a transparent table).

---

### 3.4 Drift Dashboards and Alerting

#### Theory

Detecting drift computationally (§3.2) is necessary but useless if nobody sees it in time. The source material's framing is exactly right: **dashboards provide visibility, alerts provide timely response, and the two must be designed together** — a dashboard nobody checks and an alert with no dashboard to triage against are both incomplete halves of the same system.

**What belongs on a drift dashboard**, pulling together input, output, and quality signals in one place (mirroring Module 14's "live ops vs. weekly trend" dashboard split, applied to drift specifically):

| Panel | Signal type | What it answers |
|---|---|---|
| PSI per key feature (or per embedding-drift score for text) | Input drift | "Are we still seeing the kind of traffic we validated against?" |
| Response length (mean, P95) trend | Behavioral drift | "Is the model answering with a fundamentally different shape of response?" |
| Refusal rate trend | Behavioral drift | "Is the model declining to help more/less often than baseline?" |
| Evaluation/judge score trend | Quality drift | "Is the thing that actually matters — correctness — holding up?" |
| Time-since-last-retrain / time-since-last-prompt-change | Context | "How stale is the current deployed artifact?" |

**Threshold tuning is the actual hard part, not the dashboard itself.** The source material's warning deserves to be stated as a law: **too-sensitive thresholds train teams to ignore alerts (alert fatigue, exactly as in Module 14 §3.6); too-loose thresholds mean real degradation is caught too late.** There is no universal correct threshold — PSI's 0.1/0.25 bands are a starting point, not gospel — the calibration procedure that actually works in production:

1. **Compute the metric retrospectively over a known-healthy historical period** (e.g., the trailing 90 days before your monitoring existed, or a period you're confident had no incidents).
2. **Find that metric's natural noise floor** — what values does it take on week-to-week with nothing wrong? A refusal rate that naturally oscillates between 2% and 5% needs a very different alert threshold than one that sits rock-steady at 1.0% ± 0.1%.
3. **Set the alert threshold a small number of standard deviations above the noise floor**, not at a round number picked by intuition (e.g., "refusal rate alert at mean + 3σ", recalibrated periodically, rather than a flat "alert if refusal rate > 10%" chosen without reference to what's normal for *this* system).
4. **Require sustained breach (a `for` duration), not a single data point** — identical to Module 14 §3.6's alerting principle; one noisy hour should not page anyone.
5. **Review false-positive/false-negative rate on a cadence** (monthly is reasonable) and retune — thresholds are not "set once," they are living configuration that decays as traffic patterns evolve.

#### Architecture

```
                Input drift job      Output/behavioral job     Quality job
                (PSI/KS/embedding)   (length/refusal/tone)     (eval/judge score)
                       |                     |                       |
                       +----------+----------+-----------+-----------+
                                  v
                     +---------------------------+
                     |     Drift metrics store      |
                     |  (time series -- Prometheus,  |
                     |   or a warehouse table if     |
                     |   batch-cadence)               |
                     +---------------------------+
                          |                  |
                          v                  v
              +-------------------+   +-------------------+
              |   Grafana / BI       |   |   Alert evaluator    |
              |   drift dashboard    |   |   (thresholds + `for`|
              |   (PSI, length,      |   |   sustained-breach   |
              |   refusal rate,      |   |   logic)              |
              |   eval score, all    |   +-------------------+
              |   trended together)  |             |
              +-------------------+             v
                                        +-------------------+
                                        |   Slack alert         |
                                        |   (actionable, owned,  |
                                        |   links to runbook)    |
                                        +-------------------+
```

#### Code — the source material's pattern, expanded into a production-shaped module

```python
import time
from dataclasses import dataclass, field

@dataclass
class DriftThresholds:
    psi_warning: float = 0.1
    psi_critical: float = 0.25
    response_len_z_warning: float = 2.5     # standard deviations from historical mean
    refusal_rate_warning: float = 0.08      # tuned per §3.4's calibration procedure, not a guess
    refusal_rate_critical: float = 0.15
    eval_score_drop_warning: float = 0.05   # absolute drop vs. baseline
    eval_score_drop_critical: float = 0.10


@dataclass
class DriftSnapshot:
    psi_scores: dict            # {feature_name: psi_value}
    response_len_zscore: float
    refusal_rate: float
    eval_score: float
    baseline_eval_score: float
    timestamp: float = field(default_factory=time.time)


def evaluate_thresholds(snapshot: DriftSnapshot, thresholds: DriftThresholds) -> list[dict]:
    """Collect drift metrics, check whether thresholds are crossed, return a
    list of structured alert events -- mirrors the source material's
    'collects metrics, checks thresholds, creates a summary' pattern."""
    alerts = []

    for feature, psi in snapshot.psi_scores.items():
        if psi >= thresholds.psi_critical:
            alerts.append({"severity": "critical", "signal": "input_drift",
                            "detail": f"PSI for {feature} = {psi:.3f} (>= {thresholds.psi_critical})"})
        elif psi >= thresholds.psi_warning:
            alerts.append({"severity": "warning", "signal": "input_drift",
                            "detail": f"PSI for {feature} = {psi:.3f} (>= {thresholds.psi_warning})"})

    if abs(snapshot.response_len_zscore) >= thresholds.response_len_z_warning:
        alerts.append({"severity": "warning", "signal": "behavioral_drift",
                        "detail": f"Response length z-score = {snapshot.response_len_zscore:.2f}"})

    if snapshot.refusal_rate >= thresholds.refusal_rate_critical:
        alerts.append({"severity": "critical", "signal": "behavioral_drift",
                        "detail": f"Refusal rate = {snapshot.refusal_rate:.1%}"})
    elif snapshot.refusal_rate >= thresholds.refusal_rate_warning:
        alerts.append({"severity": "warning", "signal": "behavioral_drift",
                        "detail": f"Refusal rate = {snapshot.refusal_rate:.1%}"})

    eval_drop = snapshot.baseline_eval_score - snapshot.eval_score
    if eval_drop >= thresholds.eval_score_drop_critical:
        alerts.append({"severity": "critical", "signal": "quality_drift",
                        "detail": f"Eval score dropped {eval_drop:.3f} vs. baseline"})
    elif eval_drop >= thresholds.eval_score_drop_warning:
        alerts.append({"severity": "warning", "signal": "quality_drift",
                        "detail": f"Eval score dropped {eval_drop:.3f} vs. baseline"})

    return alerts
```

```python
# Slack alerting -- summary + notification, matching the source material's
# "creates a summary and sends a Slack alert when needed" pattern.
import json
import urllib.request

def send_slack_alert(webhook_url: str, alerts: list[dict], snapshot: DriftSnapshot) -> None:
    if not alerts:
        return  # no news is good news -- do not post a "nothing happened" message

    critical = [a for a in alerts if a["severity"] == "critical"]
    warning = [a for a in alerts if a["severity"] == "warning"]

    lines = [":rotating_light: *Drift Monitor Alert*"]
    if critical:
        lines.append(f"*{len(critical)} CRITICAL:*")
        lines += [f"  - {a['signal']}: {a['detail']}" for a in critical]
    if warning:
        lines.append(f"*{len(warning)} warning:*")
        lines += [f"  - {a['signal']}: {a['detail']}" for a in warning]
    lines.append(f"<https://runbooks.internal/drift-response|Open the drift runbook>")

    payload = json.dumps({"text": "\n".join(lines)}).encode("utf-8")
    req = urllib.request.Request(webhook_url, data=payload, headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=5)


def weekly_drift_report(snapshots: list[DriftSnapshot]) -> dict:
    """A rollup report for regular team review -- the source material's
    'weekly drift report' that complements real-time Slack alerts."""
    return {
        "period_start": snapshots[0].timestamp,
        "period_end": snapshots[-1].timestamp,
        "avg_refusal_rate": sum(s.refusal_rate for s in snapshots) / len(snapshots),
        "min_eval_score": min(s.eval_score for s in snapshots),
        "max_psi_seen": max(max(s.psi_scores.values(), default=0.0) for s in snapshots),
    }
```

The production-grade equivalent as a Prometheus alerting rule (real deployments typically run this continuously rather than a hand-rolled polling loop, exactly as in Module 14 §3.6):

```yaml
groups:
  - name: drift-monitor-alerts
    rules:
      - alert: HighPSIInputDrift
        expr: max(llm_feature_psi) by (feature) >= 0.25
        for: 30m
        labels:
          severity: critical
          team: ml-platform
        annotations:
          summary: "PSI for {{ $labels.feature }} crossed 0.25 for 30+ minutes"
          runbook_url: "https://runbooks.internal/drift-response"

      - alert: ElevatedRefusalRate
        expr: llm_refusal_rate >= 0.15
        for: 15m
        labels:
          severity: critical
          team: llm-platform
        annotations:
          summary: "Refusal rate >= 15% sustained for 15+ minutes"
          runbook_url: "https://runbooks.internal/drift-response"

      - alert: EvalScoreRegression
        expr: (llm_eval_baseline_score - llm_eval_current_score) >= 0.10
        for: 1h
        labels:
          severity: critical
          team: ml-platform
        annotations:
          summary: "Evaluation score dropped >= 0.10 vs. baseline for 1+ hour"
          runbook_url: "https://runbooks.internal/drift-response"
```

#### Common dashboard/alerting mistakes

- **One giant "drift score"** that blends input/output/quality into a single number — exactly the anti-pattern §3.1 warns against; it destroys the very distinction that tells you what action to take.
- **Copying PSI's 0.1/0.25 thresholds verbatim** without checking your own feature's noise floor — some features are naturally noisier and will false-alarm constantly at 0.1; others are so stable that 0.25 is already a five-alarm fire.
- **No `for`/sustained-window requirement** — a single bad batch or a small-sample statistical fluke pages someone unnecessarily.
- **Alerting without a linked runbook** — see Module 14 §3.6's rule restated here: every alert that can fire must have a runbook, or it trains engineers to context-switch into "figure out what to even check" instead of acting.
- **A weekly report nobody reads** because it's a raw metrics dump instead of a synthesized summary with a clear "here's what changed and here's what we recommend" framing.

---

### 3.5 Drift-Monitoring Tools: Evidently AI, NannyML, whylogs/WhyLabs, and the Broader Landscape

#### Theory

You rarely need to hand-roll all of §3.2's statistics from scratch in a mature production system — a small set of well-established, purpose-built libraries cover the overwhelming majority of drift-monitoring needs, and knowing which tool solves which specific problem is itself a senior-level skill (the wrong tool choice here usually looks like "we integrated a library and it doesn't answer the question we actually have").

| Tool | Core idea | Best for | Not designed for |
|---|---|---|---|
| **Evidently AI** | Open-source Python library (plus a paid platform) generating drift/quality reports and dashboards, with built-in PSI/KS/Wasserstein-distance tests per column and prebuilt "data drift," "data quality," and "classification/regression performance" report presets. | Fast, batch-style drift reports over tabular data or LLM-output metadata; ad-hoc analysis and CI-pipeline drift gates (fail a build if PSI exceeds a threshold on a validation slice). | Continuous label-free performance estimation without ground truth (that's NannyML's specialty) and full distributed-trace-level LLM observability (that's Arize Phoenix's specialty). |
| **NannyML** | Open-source library specializing in **label-free performance estimation** — algorithms like CBPE (Confidence-Based Performance Estimation) and DLE (Direct Loss Estimation) estimate a classifier/regressor's real-world accuracy *before ground-truth labels ever arrive*, by learning the relationship between prediction confidence and correctness on a labeled reference set. | The single hardest problem in this whole module: "is my model still accurate right now," when labels lag by days/weeks or never arrive at all. | Free-text/LLM behavioral drift (refusal rate, tone) — NannyML's core algorithms target structured, labeled classification/regression problems. |
| **whylogs / WhyLabs** | Lightweight, streaming-friendly statistical profiling library (whylogs) that computes compact, mergeable "profile" summaries (distribution sketches, cardinality, null rates) of any dataset or stream, paired with a hosted monitoring platform (WhyLabs) for drift/anomaly alerting on top of those profiles. | High-volume streaming pipelines where computing full drift stats on raw data at every step is too expensive — profiles are small, mergeable, and can be computed at the edge/in-pipeline with minimal overhead. | Deep LLM-trace debugging (no built-in tracing/span model) — it's a statistics/profiling layer, not an observability/tracing platform. |
| **Arize Phoenix** | Open-source LLM/ML observability platform centered on traces, embeddings visualization, and evaluation — includes drift detection over embedding space specifically (UMAP-projected embedding drift visualizations) alongside LLM-specific eval integrations. | LLM and RAG-specific observability: embedding drift, retrieval quality, trace-level debugging tightly coupled with evaluation — a natural pairing with Module 18's agent tracing. | Classical tabular-only pipelines with no embedding/LLM component — its differentiators are LLM/embedding-centric. |
| **Deepchecks** | Open-source library for one-shot and continuous "check suites" — pre-built validation checks spanning data integrity, train-test drift, and model performance, runnable in CI or on a schedule. | Comprehensive pre-deployment and CI-gate validation (does this new data/model pass a broad battery of sanity checks) more than always-on production streaming monitoring. | Real-time/streaming drift detection at scale — its check-suite model is more naturally a periodic/CI-triggered pattern than a continuous streaming one. |
| **Great Expectations (GX)** | Open-source **data-quality/validation** framework — you write "expectations" (schema, null-rate, range, distribution checks) against a dataset and get pass/fail validation results, with some distributional-drift-style expectations available. | Data-pipeline correctness gates upstream of the model entirely (is this table well-formed, are nulls within bounds) — the data-engineering half of the drift story, not primarily a statistical-drift-modeling tool. | Nuanced statistical drift magnitude/severity scoring (PSI bands, KS statistics) — GX is closer to Module 04's reproducibility/validation concerns than to this module's drift-magnitude concerns, though the two overlap at the edges. |
| **MLflow** | Experiment tracking and model registry (Module 13) — not a drift-detection tool itself, but the system of record that a retraining decision from this module's decision matrix writes back into (new model version, new run, new stage transition). | Closing the loop: once §3.3's decision matrix says "retrain," the retrained model's lineage, metrics, and promotion decision live in MLflow. | Detecting drift in the first place — MLflow has no built-in drift-statistics engine; it's the downstream system these tools feed into. |

**The practical shape this takes in a real stack:** it is common to run more than one of these together rather than picking exactly one — e.g., whylogs profiles computed cheaply at ingestion time, feeding into an Evidently AI drift report generated daily, with NannyML supplying the label-free accuracy estimate that closes the "does this actually matter" loop, and Arize Phoenix (or an OTel + Grafana stack per Module 14) supplying the LLM-specific trace/embedding view. There is no single tool that owns "drift monitoring" end to end — the discipline is composing 2-3 of these around the specific gaps in your own pipeline (structured features vs. free text vs. label latency).

#### Code — Evidently AI drift report (illustrative API shape)

```python
# Evidently AI's exact API surface changes across versions -- treat this as
# illustrative of the *shape* of the workflow (reference vs. current dataset,
# a preset report, per-column drift results), and check references.md for
# the current API before copying verbatim into production.

from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=baseline_df, current_data=current_df)

result = report.as_dict()
drifted_columns = [
    metric["result"]["column_name"]
    for metric in result["metrics"]
    if metric.get("result", {}).get("drift_detected")
]
print("Columns with detected drift:", drifted_columns)

# Fail a CI/CD pipeline step if too many columns have drifted -- a common
# pattern for gating a scheduled retraining/deploy job on drift evidence.
dataset_drift = next(
    m["result"] for m in result["metrics"] if m["metric"] == "DatasetDriftMetric"
)
if dataset_drift["share_of_drifted_columns"] > 0.3:
    raise RuntimeError("More than 30% of features drifted -- escalate before deploying.")
```

#### Code — NannyML label-free performance estimation (illustrative API shape)

```python
# NannyML's CBPE estimates performance (e.g., ROC AUC) WITHOUT ground-truth
# labels on the analysis period -- it learns the confidence-to-correctness
# relationship on a labeled reference period and projects it forward.
# Illustrative shape; confirm exact estimator/column arguments against the
# current NannyML docs (see references.md) before production use.

import nannyml as nml

estimator = nml.CBPE(
    y_pred_proba="predicted_probability",
    y_pred="predicted_label",
    y_true="actual_label",           # only required/available in the reference period
    timestamp_column_name="timestamp",
    metrics=["roc_auc"],
    chunk_size=5000,
)
estimator.fit(reference_df)                  # reference period: HAS ground-truth labels
estimated_performance = estimator.estimate(analysis_df)  # analysis period: NO labels needed
estimated_performance.plot().show()

results_df = estimated_performance.to_df()
below_threshold = results_df[results_df[("roc_auc", "value")] < 0.80]
if not below_threshold.empty:
    print("Estimated performance dropped below threshold in", len(below_threshold), "chunks -- investigate.")
```

#### Code — whylogs streaming profile (illustrative API shape)

```python
# whylogs computes small, mergeable statistical "profiles" -- cheap enough
# to compute inline in a high-volume pipeline, then compared for drift later.
import whylogs as why

reference_profile = why.log(pandas=baseline_df).view()
current_profile = why.log(pandas=current_df).view()

# Profiles support a drift-style comparison via whylogs' own utilities /
# the WhyLabs platform; the core value proposition is that the PROFILE,
# not the raw data, is what gets shipped and stored for comparison --
# important when raw data cannot leave a boundary for privacy/cost reasons.
reference_profile.to_pandas().to_csv("baseline_profile.csv")
current_profile.to_pandas().to_csv("current_profile.csv")
```

---

### 3.6 Retraining Decisions and the Incident Runbook

#### Theory

This section closes the loop the whole module has been building toward: once a drift alert fires and the decision matrix (§3.3) has pointed toward an action, **how does a team actually execute that action without improvising under pressure?** The source material's runbook is deliberately simple and sequential, precisely because a runbook's job during a live incident is to remove decisions, not add them:

1. **Validate** — confirm the alert is real, not a monitoring bug, a data pipeline outage silently zeroing out a feature, or a one-off blip that a sustained-window check should have already filtered (§3.4). Re-run the drift computation manually against a fresh pull of the same data if there's any doubt.
2. **Assess impact** — is there an actual business or quality consequence, or is this drift-without-impact (the common, benign case from §3.1's decision table)? Check the quality/eval signal specifically, not just the drift signal that triggered the alert.
3. **Review recent changes** — what deployed, in prompts, models, pipeline configuration, or upstream dependencies, in the window immediately preceding the drift signal? This is almost always the fastest path to a root cause and directly feeds the decision matrix's "recent change detected" input.
4. **Act** — choose retrain, prompt/pipeline update, rollback, or further investigation, using the evidence gathered in steps 1-3 and the decision matrix in §3.3 — not gut feeling.
5. **Document** — record what was observed, what was decided, why, and what happened next. This is the step teams skip under time pressure and the one that most determines whether the *next* incident is faster or exactly as slow as this one.

**Why "document" is not busywork.** A runbook executed without a written record produces no institutional memory — the next on-call engineer facing a similar drift signature starts from zero, re-litigating the same investigation. A short, structured incident record (even five lines: what fired, what was checked, what was decided, what changed as a result, follow-up owner) compounds into an increasingly fast, increasingly confident on-call rotation over time — this is the same principle Module 14 makes about runbooks generally, applied specifically to the retrain/rollback/prompt-fix decision.

**Standard, repeatable processes over case-by-case improvisation** is the source material's other core principle here, worth stating directly: the runbook exists precisely so an engineer "does not improvise" during an incident — improvisation under pressure is where inconsistent, evidence-free decisions creep in (the retrain-as-reflex failure mode this whole module argues against).

#### Architecture — the runbook as a state machine

```
   Drift alert fires
          |
          v
   +-------------+     alert is a false positive /       +----------------+
   |  1. VALIDATE  | ---  monitoring bug / stale data --->  |  Close as noise |
   +-------------+                                       |  (log why, tune  |
          |  confirmed real                               |  threshold if    |
          v                                               |  recurring)      |
   +-------------+                                       +----------------+
   |  2. ASSESS     |
   |  IMPACT        |  no measurable quality/business impact --> log as
   +-------------+     "monitor only", re-check next window
          |  real impact confirmed
          v
   +-------------+
   |  3. REVIEW     |
   |  RECENT        |
   |  CHANGES       |
   +-------------+
          |
          v
   +-------------+
   |  4. ACT        |  --->  decide_action() from §3.3  --->  retrain /
   |  (decision      |                                       rollback /
   |  matrix)        |                                       prompt fix /
   +-------------+                                       further investigation
          |
          v
   +-------------+
   |  5. DOCUMENT   |  incident record: signals, evidence, decision,
   |                |  rationale, owner, follow-up date
   +-------------+
          |
          v
   Continue monitoring (verify the action actually resolved the drift
   signal in the following window(s) -- close the loop)
```

#### Code — the runbook as a lightweight, structured workflow

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional

@dataclass
class IncidentRecord:
    alert_id: str
    signals_fired: list[str]
    validated_real: Optional[bool] = None
    business_impact_confirmed: Optional[bool] = None
    recent_changes_found: list[str] = field(default_factory=list)
    action_taken: Optional[str] = None
    rationale: str = ""
    owner: str = ""
    opened_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    closed_at: Optional[str] = None
    follow_up_check_at: Optional[str] = None


class DriftRunbook:
    """Executable scaffold of the 5-step runbook. Each step is a deliberate,
    named method so an on-call engineer (or a future automation layer) can
    see exactly which step of the process they are in -- this is the code
    equivalent of 'a fixed runbook so an engineer does not improvise.'"""

    def __init__(self, alert_id: str, signals_fired: list[str], owner: str):
        self.record = IncidentRecord(alert_id=alert_id, signals_fired=signals_fired, owner=owner)

    def validate(self, recompute_fn) -> bool:
        """Step 1: re-run the drift computation on a fresh pull. If it
        doesn't reproduce, close as noise immediately -- do not proceed."""
        confirmed = recompute_fn()
        self.record.validated_real = confirmed
        return confirmed

    def assess_impact(self, quality_dropping: bool, business_metric_dropping: bool) -> bool:
        """Step 2: does this drift actually matter?"""
        impact = quality_dropping or business_metric_dropping
        self.record.business_impact_confirmed = impact
        return impact

    def review_recent_changes(self, changes: list[str]) -> list[str]:
        """Step 3: pull from the deploy/prompt/model registry (Module 03/13)."""
        self.record.recent_changes_found = changes
        return changes

    def act(self, evidence) -> str:
        """Step 4: delegate to the decision matrix from §3.3."""
        from_matrix = decide_action(evidence)   # DriftEvidence -> Action, see §3.3
        self.record.action_taken = from_matrix.value
        return from_matrix.value

    def document(self, rationale: str, follow_up_check_at: str) -> IncidentRecord:
        """Step 5: close the loop with a written record."""
        self.record.rationale = rationale
        self.record.follow_up_check_at = follow_up_check_at
        self.record.closed_at = datetime.now(timezone.utc).isoformat()
        return self.record
```

```python
# Worked example: an on-call engineer walking through a real drift alert.
runbook = DriftRunbook(alert_id="drift-2026-07-30-001",
                       signals_fired=["behavioral_drift:refusal_rate"],
                       owner="a.ranjan@castsoftware.com")

if runbook.validate(recompute_fn=lambda: True):
    impact = runbook.assess_impact(quality_dropping=False, business_metric_dropping=False)
    changes = runbook.review_recent_changes(changes=["prompt v14 deployed 6h before alert"])

    evidence = DriftEvidence(
        input_drift_high=False,
        behavioral_drift_high=True,
        quality_dropping=impact,
        recent_change_detected=len(changes) > 0,
    )
    action = runbook.act(evidence)   # -> "rollback", per the decision matrix

    record = runbook.document(
        rationale="Refusal rate rose from 3% to 17% within 6h of prompt v14 deploy; "
                   "no quality-score drop yet but behavioral shift is large and time-correlated. "
                   "Rolling back prompt to v13 pending a fixed v15.",
        follow_up_check_at="2026-07-31T09:00:00Z",
    )
    print(record)
```

#### Comparison — the four possible actions, side by side

| Action | Cost/speed | Reverses what | Best when | Risk if wrong |
|---|---|---|---|---|
| **Retrain** | Slow (hours-days), compute-expensive | Nothing — produces a new model artifact | Confirmed input drift + confirmed quality drop, sustained over time | Ships a model trained on a small/biased recent window; doesn't fix a pipeline/prompt bug |
| **Prompt/pipeline fix** | Fast (minutes-hours) | The specific prompt/pipeline logic | Behavioral drift with no model/data root cause | May mask a deeper data/model issue if applied without investigating first |
| **Rollback** | Fastest (minutes) | A specific recent deploy | Drift is time-correlated with a known recent change | Reverts a change that might have been a genuine, correct improvement |
| **Investigate further** | Variable (the "cost" is delay) | Nothing yet | Signals are ambiguous or conflicting | Delays a real fix if drawn out too long — should have a time-box |

---

## 4. Common Mistakes (Module-Wide)

1. **Retraining as a reflex.** Treating every drift alert as "time to retrain" — expensive, slow, and frequently the wrong fix for a prompt or pipeline regression (§3.3).
2. **Collapsing input, behavioral, and quality drift into one score.** Destroys the exact information (§3.1) that tells you what action to take.
3. **Using current-window quantiles as PSI bin edges.** Silently understates drift by construction (§3.2).
4. **Trusting KS p-values without checking the effect size at production sample sizes.** Statistically significant does not mean operationally meaningful (§3.2).
5. **Applying tabular drift tests to raw free text.** Text needs embedding-space methods, not PSI/KS on strings (§3.2).
6. **Copying PSI's textbook thresholds without recalibrating against your own feature's noise floor.** Some features are noisier than the textbook bands assume (§3.4).
7. **Alerting without a linked, specific runbook.** Trains engineers to either ignore alerts or improvise a response from scratch each time (§3.4, §3.6).
8. **Skipping the "document" step of the runbook under time pressure.** The single biggest determinant of whether the next incident is faster or just as slow (§3.6).
9. **Letting the drift baseline roll forward silently.** A baseline that recomputes from "the last 30 days" rather than freezing at the last validated deploy drifts together with production data and stops detecting anything (§3.2).
10. **Picking one drift tool and expecting it to cover the whole problem.** No single tool in §3.5 owns input + behavioral + label-free-quality drift end to end; composing 2-3 is normal.

## 5. Best Practices / Production Tips

- Keep input-drift, behavioral-drift, and quality-drift jobs **architecturally separate** (separate schedules, separate outputs) even if they report into one dashboard — this is the single highest-leverage design decision in the whole module for troubleshooting speed.
- **Freeze your baseline at the last validated deploy**, not a rolling window, and version the baseline itself (store which deploy/date it corresponds to) so a drift report can say "compared against the baseline validated on 2026-06-15."
- Recalibrate alert thresholds against your own historical noise floor **on a recurring cadence** (quarterly is a reasonable default), not once at launch.
- Always pair a drift alert's **magnitude** (PSI value, KS statistic) with its **duration** (a `for` sustained-window requirement) before paging anyone.
- Route every confirmed retrain/rollback/prompt-fix decision back into the versioning and registry systems from Module 03/13 — a drift decision that isn't recorded in the model/prompt lineage is invisible to the next investigation.
- For LLM systems specifically, monitor **refusal rate and response length** even when nothing about the visible input distribution has changed — this is often the *earliest* available signal of an upstream vendor-side model update you didn't initiate.
- Time-box the "investigate further" action — an open-ended investigation with no deadline tends to silently become "no action," which is different from a deliberate "monitor only" decision.
- Treat NannyML-style label-free estimation as a **complement to**, not a replacement for, eventual ground-truth evaluation — it is an estimate under a confidence-correctness relationship learned on the reference period, and that relationship can itself drift.

## 6. Real-World Case Studies (Reasoned Inference)

The following are reasoned inferences about how organizations with large-scale ML/LLM systems likely approach drift monitoring, based on their publicly known architecture and engineering blog posts — not confirmed internal implementation details.

**Netflix** has published extensively on its internal ML platform (Metaflow) and its recommendation systems, which serve highly seasonal, rapidly shifting viewing behavior (new releases, regional catalog differences, day-of-week effects). Given the sheer number of models Netflix operates simultaneously across recommendation, personalization, and content-ranking surfaces, it is reasonable to infer that a large share of their monitoring investment goes into **distinguishing expected seasonal/catalog-driven input drift from genuine model staleness** — precisely the covariate-shift-vs-quality-drift distinction this module centers on — since naively retraining on every seasonal shift across that many models would be operationally unsustainable.

**Uber's** publicly documented Michelangelo ML platform blog posts describe centralized feature stores and model-monitoring tooling built specifically because Uber operates many geographically and temporally distinct markets (a rider-demand model's "normal" input distribution in one city during a local event looks like severe drift by another city's baseline). This is consistent with the architectural pattern in §3.2 of comparing against a *per-segment* baseline rather than one global baseline — a natural inference from the scale and heterogeneity Uber has described, rather than a confirmed implementation detail.

**OpenAI and Anthropic**, as LLM providers operating models that are also frequently updated behind stable-looking API aliases (a model alias can point to an updated underlying checkpoint), create exactly the "behavioral drift with no input drift" scenario this module discusses as a first-class case (§3.1, §3.3): downstream applications built on top of these APIs can see refusal-rate or response-style shifts with zero change to their own prompts or user traffic, purely from an upstream model update. It is a reasonable inference — consistent with how both companies communicate model versioning and deprecation publicly — that downstream teams building production systems on these APIs should treat "pin to a dated model snapshot rather than a rolling alias" as a direct mitigation for exactly this failure mode, and should specifically monitor behavioral signals (not just input drift) for this reason. This is inference about a sound architectural practice given publicly known API versioning behavior, not a claim about either company's internal monitoring implementation.

---

## 7. Summary, Key Takeaways, and Production Checklist

### Summary

Drift monitoring exists to answer the question a passing HTTP 200 can never answer on its own: is the model still right? This module built that answer in three layers — **input drift** (did the world feeding the model change), **behavioral drift** (did the model's own outputs change, an LLM-specific addition to the classical taxonomy), and **quality drift** (does any of it actually matter, measured against ground truth or a trusted judge) — backed by three statistical workhorses (PSI, KL divergence, KS test) plus embedding-space methods for free text. None of this is useful without a **decision matrix** that turns "something fired" into "here is the one correct action" (retrain, prompt/pipeline fix, rollback, or investigate), a **dashboard and tuned alerting layer** that surfaces the signal without training people to ignore it, and a **five-step runbook** (validate → assess impact → review recent changes → act → document) that removes improvisation from the highest-pressure moment an ML team faces.

### Key Takeaways

1. Data drift, behavioral drift, and quality drift are different signals that can move independently — never collapse them into one score.
2. PSI is the interpretable, production-standard magnitude metric; KS gives a formal (but sample-size-sensitive) hypothesis test; KL divergence is PSI's mathematical ancestor; embeddings extend all of this to free text.
3. A drift alert is evidence, not an instruction — the decision matrix, not reflex, decides retrain vs. prompt-fix vs. rollback vs. investigate.
4. Threshold tuning is the actual hard part of alerting — calibrate against your own historical noise floor, require sustained breach, and revisit periodically.
5. No single tool (Evidently AI, NannyML, whylogs, Arize Phoenix, Deepchecks, Great Expectations) covers the whole problem — compose 2-3 around your system's actual gaps.
6. The runbook's "document" step is the one most often skipped under pressure and the one that most compounds into faster future incident response.

### Production Checklist

- [ ] Input-drift, behavioral-drift, and quality-drift jobs run on **separate schedules** and report as **separate signals**, not one blended score.
- [ ] Baseline is **frozen at the last validated deploy** and versioned, not a silently rolling window.
- [ ] PSI/KS/KL bin edges and reference windows are derived from the **baseline**, never from the current window.
- [ ] Free-text/LLM inputs are monitored via **embedding-space** distance, not raw-string PSI/KS.
- [ ] Behavioral signals (response length, refusal rate, structural conformance) are tracked even when input drift looks quiet.
- [ ] Alert thresholds are calibrated against **historical noise floor**, require a **sustained window**, and are revisited on a recurring cadence.
- [ ] Every alert links to a **specific, named runbook** — no alert fires into a vacuum.
- [ ] A **decision matrix** (not ad hoc judgment) determines retrain vs. prompt-fix vs. rollback vs. investigate.
- [ ] Every retrain/rollback/prompt-fix decision is **recorded** (incident record) and **fed back** into the model/prompt registry (Module 03/13).
- [ ] The "investigate further" action is **time-boxed**, not left open-ended.
- [ ] A **weekly/periodic drift report** exists for team review, separate from real-time Slack alerts, so trend-level decay is caught even when no single threshold breach fires.
