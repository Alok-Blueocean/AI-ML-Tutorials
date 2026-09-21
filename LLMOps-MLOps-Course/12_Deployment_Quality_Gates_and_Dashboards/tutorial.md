# Module 12 — Deployment Quality Gates and Release Dashboards

> "A model that passed evaluation once is a rumor. A model that passes evaluation every time it tries to ship is a fact." — the operating principle behind everything in this module.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain the four canonical evaluation trigger points in an LLM deployment pipeline (PR, push-to-main, schedule, data-update) and justify why each exists independently of the others.
2. Design a tiered evaluation strategy that trades off cost, latency, and coverage across those trigger points, and estimate the weekly evaluation spend for a given team's PR/push cadence.
3. Distinguish absolute gates from relative gates, explain why some checks (format validity, hallucination rate) must be hard blocks with zero tolerance, and implement both gate types in code.
4. Wire an evaluation gate directly into a GitHub Actions workflow so that a failing evaluation halts the pipeline via a non-zero exit code — not merely logs a warning.
5. Design a Grafana dashboard for LLM release readiness: the panels it needs, the thresholds that drive panel color, and the alert rules that page a human instead of waiting to be read.
6. Read an evaluation dashboard the way a release manager does: headline status first, dimension breakdown second, trend third, regression-vs-baseline fourth — and turn that reading into a ship/hold/rollback decision.
7. Identify the common failure modes of quality-gate systems (threshold instability, missing hard blocks, dashboards nobody reads, gates without an action path) and describe the organizational fix for each.

### Prerequisites

- Modules 01–06 (CI/CD foundations, versioning/registries/rollback, reproducibility, Docker, Kubernetes) — this module assumes you can already build and deploy a containerized service through a pipeline; it focuses on the *gate* that pipeline must pass through.
- Module 09–11 territory (prompt lifecycle evaluation, building evaluation datasets, LLM-as-judge design) is the natural predecessor: this module assumes you already have *an evaluation function* that takes a model version and returns metrics. If you don't yet have that, skim Module 10 and 11 first — this module is about what happens *after* you can score a model, not how to build the scorer.
- Working familiarity with GitHub Actions YAML syntax (Module 01/02) and with Prometheus/Grafana as a concept (metrics, dashboards, alerting) — Module 14 goes deep on OpenTelemetry/Prometheus instrumentation; this module borrows Grafana as the dashboard layer without re-deriving it.

### Key Terminology

| Term | One-line definition |
|---|---|
| **Evaluation trigger** | A pipeline event (PR opened, push to main, cron schedule, data change) that causes an evaluation run to fire automatically. |
| **Tiered evaluation** | Running progressively larger/slower/costlier evaluation suites at progressively less-frequent trigger points, so cheap checks run often and expensive checks run rarely. |
| **Quality gate** | An automated pass/fail checkpoint in a pipeline that blocks promotion of an artifact when a metric fails to meet a defined bar. |
| **Absolute gate** | A gate that compares a metric against a fixed, version-independent floor (e.g., "hallucination rate must be < 3%"), regardless of what production currently does. |
| **Relative gate** | A gate that compares the candidate version's metric against the current production baseline (e.g., "composite score must not drop by more than 1% vs. production"). |
| **Hard block** | A gate that is treated as non-negotiable and cannot be overridden by a human approval — typically reserved for format-validity and hallucination/safety checks. |
| **Frozen evaluation set** | A version-pinned, immutable set of evaluation examples (e.g., "eval-set v1.0") used so that score changes reflect model changes, not test-set drift. |
| **Release readiness signal** | The green/yellow/red traffic-light summary a dashboard shows to answer "can we ship this?" in one glance. |
| **Composite score** | A single weighted aggregate of multiple evaluation dimensions (correctness, relevance, format, safety) rolled into one headline number. |
| **Canary evaluation** | Running the evaluation suite (or a proxy of it, via real traffic sampling) against a small percentage of live traffic before full rollout. |
| **Fail-fast pipeline** | A CI/CD design where a failing check calls a non-zero `sys.exit()` (or equivalent) immediately, stopping all downstream jobs rather than continuing and reporting a warning. |
| **Dimension breakdown** | Decomposing a composite score into its constituent per-dimension scores (correctness, format, relevance, latency, safety) so failures can be root-caused. |

---

## 2. Why This Topic Matters — Where It Fits in the MLOps/LLMOps Lifecycle

Every prior module in this course builds toward a moment: someone wants to ship a change. It might be a new prompt template, a fine-tuned checkpoint, a new retrieval index, a new system message, or a dependency bump that silently changed tokenizer behavior. Modules 01–06 gave you the mechanics to package and deploy that change reliably. Modules 09–11 gave you the ability to *measure* whether a candidate is good. This module is the connective tissue: **when** does the measurement happen, and **what happens automatically** as a result of the measurement?

```
 [09] Prompt lifecycle    [10] Eval datasets      [11] LLM-as-judge
   & statistical eval        (frozen, versioned)     (scoring engine)
        \                         |                        /
         \                        |                       /
          v                       v                      v
                 +----------------------------------+
                 |   AN EVALUATION FUNCTION EXISTS   |
                 |   eval(model_version) -> metrics  |
                 +----------------+-------------------+
                                  |
                                  v
                 +----------------------------------------+
                 |         THIS MODULE (12)                |
                 |  WHEN does eval() run? (triggers)        |
                 |  WHAT decides pass/fail? (gates)         |
                 |  WHO sees the result, and how? (dashboards)|
                 +----------------+-------------------------+
                                  |
                                  v
                 [13] Experiment tracking / MLflow — where gate
                       results and dashboard history are stored
                 [14] OpenTelemetry & monitoring — where live
                       production signal feeds back into gates
```

Without this module, a team can have a perfectly good evaluation harness and still ship regressions constantly, for one of three reasons:

1. **The evaluation only runs when someone remembers to run it.** A harness that exists but isn't wired to a trigger is a manual step, and manual steps get skipped under deadline pressure — precisely when they matter most.
2. **The evaluation runs but nothing enforces the result.** A dashboard showing "score dropped 4%" that nobody is required to look at before merging is a historical record, not a gate. Multiple companies' postmortems (and the two 2026 arXiv preprints on release gating referenced in this module's `references.md`) converge on the same root cause pattern for LLM production incidents: an evaluation existed, the regression was visible in a chart, and the release still shipped because nothing *blocked* it.
3. **The evaluation is single-dimensional.** A release can be "more correct" and still be unacceptable if P95 latency doubled, cost tripled, or hallucination rate quietly crept up. Production readiness is inherently multi-dimensional, and a single pass/fail number hides which dimension actually regressed.

This module closes all three gaps: triggers make evaluation automatic, gates make the result enforceable, and dashboards make the result legible to the humans who still own the final ship/hold/rollback call on judgment-heavy releases.

**Where this sits in a senior MLOps/LLMOps interview.** "How would you make sure a bad prompt change never reaches production users?" and "design a release process for an LLM-backed feature" are near-canonical senior questions. The expected answer is not "we have an eval script" — it's exactly the three-layer structure this module teaches: tiered triggers (cost-aware), absolute + relative gates (with named hard blocks), and a dashboard whose headline status maps 1:1 onto the same thresholds the gate code enforces.

---

## 3. Main Concepts

### 3.1 Evaluation Triggers — When Should Evaluation Run?

#### Theory

The naive mental model of "run eval before deploy" hides an important question: deploy triggered by *what*? A production LLM system changes for more reasons than a human clicking "merge." The source material identifies four distinct trigger points, and each exists to catch a *different class* of regression:

| Trigger | What changed | What it catches | Typical cadence |
|---|---|---|---|
| **Pull request** | Proposed code/prompt/config diff, not yet merged | Obvious regressions before they touch shared history | Every PR |
| **Push to main** | Merged, about-to-be-released code | Full-fidelity regression check before an artifact is registered | Every merge |
| **Schedule (cron)** | Nothing in the repo — the *world* around a static model | Real-world drift: user behavior shift, upstream data shift, silent provider-side model updates | Weekly (or daily for high-risk systems) |
| **Data update** | Training data, RAG corpus, or fine-tuning set changed | Data-induced regressions that no code diff would reveal | On every data pipeline run that touches production-facing data |

The critical insight the transcript makes explicit: **these are not redundant.** A PR check only proves the diff didn't regress against a small sample — it says nothing about drift that appears three weeks after merge because the world moved and the model didn't. A push-to-main check proves this exact code is fit to register as an artifact — it says nothing about whether *last month's* already-deployed artifact has quietly drifted. A data-update trigger exists because the single most common cause of "the model got worse but nobody changed the model" is that the retrieval corpus, few-shot examples, or fine-tuning set changed underneath it.

**When to use each:** all four, for anything customer-facing or safety-relevant. **When to relax:** internal tools, experimental branches, and pure research iterations can skip the schedule and data-update triggers to save cost — but any path that can reach production traffic needs at minimum the PR and push gates.

**Tradeoff to internalize:** every trigger point costs money and CI minutes. The tiered strategy in §3.2 exists specifically to make "run evaluation more often" affordable rather than a binary "afford it or skip it" choice.

#### Architecture

```
 PULL REQUEST OPENED                PUSH TO main                 CRON (weekly)              DATA PIPELINE RUN
        |                                |                            |                          |
        v                                v                            v                          v
 +---------------+              +----------------+          +------------------+       +-------------------+
 | Tier 1: Fast   |              | Tier 2: Full    |          | Tier 3: Drift     |       | Tier 2/3: Data-    |
 | correctness    |              | eval, all       |          | + hallucination   |       | induced regression |
 | check, 50      |              | metrics, 200     |          | check, 500        |       | check on affected  |
 | examples       |              | examples         |          | prod-sampled      |       | slice               |
 +-------+--------+              +-------+---------+          | examples           |       +---------+-----------+
         |                                |                    +---------+---------+                 |
         v                                v                              v                             v
   pass -> allow merge            pass -> register            pass -> no action              pass -> allow data
   fail -> block PR,              artifact, proceed           fail -> alert on-call           promotion
   comment on PR                  to deploy                   (production already live,       fail -> block pipeline,
   fail -> sys.exit(1)            fail -> sys.exit(1),         cannot "block" the past —       hold data version
   blocks GH Actions               block deploy, no artifact   only page + investigate)
```

#### Examples

**Beginner.** A weekend hobby project runs `pytest` on 10 hand-written examples whenever anyone opens a PR. No push trigger, no schedule. This catches typos and obvious breakage; it will never catch drift because nothing runs after merge.

**Intermediate.** A startup's customer-support bot runs the PR check (50 examples, correctness only) and a push-to-main check (200 examples, full metrics) before every deploy. This is a solid baseline — it catches regressions at the moment of change — but it has a blind spot: if the underlying hosted model provider silently changes model behavior (a known, real risk with third-party LLM APIs), nothing will notice until a customer complains.

**Production-grade.** A regulated fintech's LLM-backed document-summarization service runs all four triggers: PR (fast), push (full), weekly schedule against a rotating sample of real anonymized production traffic (drift detection), and a triggered re-eval whenever the RAG knowledge base ingests a new document batch. The weekly and data-triggered runs write results to the same metrics store as the CI runs, so the release dashboard (§3.3) shows one continuous timeline regardless of which trigger produced each data point.

#### Code — a tiered trigger implementation in GitHub Actions

```yaml
# .github/workflows/llm-eval-triggers.yml
name: LLM Evaluation Triggers

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"        # Monday 06:00 UTC — weekly drift check
  workflow_dispatch:            # manual trigger, e.g. after a data pipeline run
    inputs:
      reason:
        description: "Why is this being run manually (e.g. data-update)"
        required: true
        default: "data-update"

jobs:
  tier-1-pr-check:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements-eval.txt
      - name: Fast correctness eval (50 examples)
        run: |
          python -m evals.run \
            --eval-set data/eval_sets/v1.0_smoke50.jsonl \
            --checks correctness \
            --timeout-minutes 5 \
            --out results/tier1.json
      - name: Gate on tier-1 result
        run: python -m evals.gate --input results/tier1.json --tier 1
        # gate script calls sys.exit(1) on failure -> job fails -> PR blocked

  tier-2-push-check:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements-eval.txt
      - name: Full eval (200 examples, all metrics)
        run: |
          python -m evals.run \
            --eval-set data/eval_sets/v1.0_full200.jsonl \
            --checks correctness,format,hallucination,relevance,latency \
            --timeout-minutes 20 \
            --out results/tier2.json
      - name: Gate on tier-2 result (absolute + relative)
        run: python -m evals.gate --input results/tier2.json --tier 2 --baseline registry://production
      - name: Register artifact (only reached if gate passed)
        run: python -m registry.register --version ${{ github.sha }} --results results/tier2.json

  tier-3-scheduled-drift-check:
    if: github.event_name == 'schedule' || github.event.inputs.reason == 'data-update'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements-eval.txt
      - name: Drift + hallucination eval (500 prod-sampled examples)
        run: |
          python -m evals.run \
            --eval-set data/eval_sets/prod_sample_500.jsonl \
            --checks hallucination,drift \
            --model production \
            --out results/tier3.json
      - name: Page on-call if drift gate fails (cannot block already-live traffic)
        if: failure()
        run: python -m alerting.page --channel oncall-llm --severity high --context results/tier3.json
```

This mirrors the cost model from the source material almost exactly: Tier 1 (~$0.36/run, 5-minute timeout, correctness-only) is cheap enough to run on every PR without a second thought; Tier 2 (~$1.44/run, 20-minute timeout, full metrics) is reserved for merge events where a bad artifact would otherwise get registered; Tier 3 (500 examples, hallucination + drift) is expensive enough that weekly cadence — not per-commit — is the right amortization. At roughly 20 PRs/week and 5 pushes/week, the blended weekly evaluation spend lands near $14.40/week — trivial next to the cost of one undetected hallucination regression reaching customers.

| Tier | Trigger | Examples | Timeout | Checks | Approx. cost/run |
|---|---|---|---|---|---|
| 1 | PR | 50 | 5 min | correctness only | ~$0.36 |
| 2 | Push to main | 200 | 20 min | correctness, format, hallucination, relevance, latency | ~$1.44 |
| 3 | Weekly schedule | 500 (prod-sampled) | — | hallucination, drift | amortized weekly |

---

### 3.2 Quality Gates — Absolute vs. Relative, and Hard Blocks

#### Theory

A "quality gate" is the decision function that turns a metrics dictionary into a boolean: ship or don't. The single most important design decision is that **one number is not enough.** Production readiness is multi-dimensional — quality, latency, safety, cost — and a release can fail on any one axis even while excelling on the others.

The source material draws a sharp, useful distinction between two *kinds* of threshold:

- **Absolute gates** compare a metric to a fixed floor that does not depend on what production currently looks like. Example: "hallucination rate must be below 3%, full stop." Absolute gates encode a minimum acceptable bar — the answer to "is this good enough to exist in production at all," independent of history.
- **Relative gates** compare the candidate to the *current production baseline*. Example: "composite score must not regress by more than 1% versus the version currently live." Relative gates protect against slow, cumulative degradation — the death-by-a-thousand-cuts failure mode where each release is individually "good enough" on an absolute scale but the trend line quietly slides downward release after release.

Both are necessary and neither substitutes for the other. A model can pass every absolute gate (it clears the minimum bar) while still regressing relative to production (a genuine step backward) — and a model can show a relative *improvement* while still being unacceptable in absolute terms (e.g., "20% better than a production version that was already broken" is not a shippable release).

**Hard blocks** are a special category of absolute gate that the source material calls out explicitly: **format validity and hallucination rate always block release, with zero tolerance and no override path.** This is a deliberate policy decision, not a technical limitation — teams choose to make these non-negotiable because a malformed output (breaks the calling application) or a hallucinated fact (actively misinforms the user) are failure modes with no acceptable "we'll fix it in the next release" grace period. Contrast this with, say, a P95 latency regression, which is serious but often *is* releasable with an incident ticket and a fast-follow, because the failure mode (slow but correct) is less harmful than the failure modes hard blocks guard against.

**When to use gates at all:** any system where the cost of a bad release (reputational, financial, safety) exceeds the cost of the CI minutes to check for it — which is essentially every production LLM system serving real users. **When NOT to over-gate:** early-stage research/prototyping loops where the friction of a strict gate blocks legitimate experimentation; use gates on the path to production, not on every experimental branch.

#### Architecture

```
                         PUSH TO main
                              |
                              v
        +---------------------------------------------+
        | 1. Load frozen eval set (v1.0) + baseline     |
        |    metrics from the model/experiment registry |
        +---------------------+-------------------------+
                              v
        +---------------------------------------------+
        | 2. Run candidate model on all 200 examples;   |
        |    compute per-example + aggregated metrics   |
        +---------------------+-------------------------+
                              v
                 +------------+-------------+
                 |   ABSOLUTE GATE CHECK      |
                 |  format_valid_rate == 1.0? |<--- HARD BLOCK, no override
                 |  hallucination_rate<0.03?  |<--- HARD BLOCK, no override
                 |  eval_score       >=0.82?  |
                 |  p95_latency_ms  <=1200?   |
                 +------------+-------------+
                              |
                fail -> sys.exit(1), pipeline stops, reason logged
                              |
                            pass
                              v
                 +------------+-------------+
                 |   RELATIVE GATE CHECK      |
                 |  score_candidate >=        |
                 |  score_production - 0.01   |
                 +------------+-------------+
                              |
                fail -> sys.exit(1), pipeline stops, "regression vs prod" reason
                              |
                            pass
                              v
                  register artifact, proceed to deploy
                  (canary -> full rollout, see Module 06)
```

#### Examples

**Beginner.** A single `if score < 0.8: raise SystemExit(1)` check. Catches egregious regressions; has no concept of relative comparison or hard-block categories, so a version that's "still above 0.8 but noticeably worse than what's live" ships anyway.

**Intermediate.** Separate absolute checks for score, latency, and hallucination — but no relative/baseline comparison. This catches every absolute failure mode but is blind to the slow-regression pattern: five consecutive releases can each individually clear 0.82 while the actual trend goes 0.90 -> 0.88 -> 0.86 -> 0.84 -> 0.82, a real 8-point degradation that no single-release absolute check ever flags.

**Production-grade.** The full pattern below: absolute + relative + explicit hard-block category + machine-readable reason codes that a dashboard (§3.3) and a Slack/PR-comment integration can both consume.

#### Code — a release gate implementation

```python
# evals/gate.py
"""Release gate: decides whether a candidate model version may be promoted.

Exit code 0  -> release approved, safe to register/deploy.
Exit code 1  -> release blocked; reason is printed and written to results/gate_reason.json
               for the dashboard and PR-comment bot to consume.
"""
import json
import sys
from dataclasses import dataclass, field


@dataclass
class GateResult:
    passed: bool
    blocked_reasons: list[str] = field(default_factory=list)
    hard_block: bool = False   # True if a zero-tolerance check failed


# --- Absolute gates: fixed floors, independent of production baseline ---
ABSOLUTE_THRESHOLDS = {
    "eval_score_min": 0.82,
    "p95_latency_ms_max": 1200,
    "hallucination_rate_max": 0.03,
    "format_valid_rate_min": 1.00,   # hard block: any malformed output fails the release
}

# --- Relative gate: candidate must not regress more than this vs. production ---
RELATIVE_REGRESSION_TOLERANCE = 0.01  # 1 percentage point


def check_absolute_gates(metrics: dict) -> GateResult:
    reasons = []
    hard_block = False

    if metrics["format_valid_rate"] < ABSOLUTE_THRESHOLDS["format_valid_rate_min"]:
        reasons.append(
            f"HARD BLOCK: format_valid_rate={metrics['format_valid_rate']:.3f} "
            f"< required {ABSOLUTE_THRESHOLDS['format_valid_rate_min']}"
        )
        hard_block = True

    if metrics["hallucination_rate"] > ABSOLUTE_THRESHOLDS["hallucination_rate_max"]:
        reasons.append(
            f"HARD BLOCK: hallucination_rate={metrics['hallucination_rate']:.3f} "
            f"> allowed {ABSOLUTE_THRESHOLDS['hallucination_rate_max']}"
        )
        hard_block = True

    if metrics["eval_score"] < ABSOLUTE_THRESHOLDS["eval_score_min"]:
        reasons.append(
            f"BLOCK (quality): eval_score={metrics['eval_score']:.3f} "
            f"< required {ABSOLUTE_THRESHOLDS['eval_score_min']}"
        )

    if metrics["p95_latency_ms"] > ABSOLUTE_THRESHOLDS["p95_latency_ms_max"]:
        reasons.append(
            f"BLOCK (latency): p95_latency_ms={metrics['p95_latency_ms']} "
            f"> allowed {ABSOLUTE_THRESHOLDS['p95_latency_ms_max']}"
        )

    return GateResult(passed=not reasons, blocked_reasons=reasons, hard_block=hard_block)


def check_relative_gate(candidate_score: float, production_score: float) -> GateResult:
    delta = candidate_score - production_score
    if delta < -RELATIVE_REGRESSION_TOLERANCE:
        return GateResult(
            passed=False,
            blocked_reasons=[
                f"BLOCK (regression): candidate={candidate_score:.3f} vs. "
                f"production={production_score:.3f} (delta {delta:+.3f}, "
                f"tolerance {-RELATIVE_REGRESSION_TOLERANCE:+.3f})"
            ],
        )
    return GateResult(passed=True)


def run_gate(candidate_metrics: dict, baseline_metrics: dict) -> None:
    absolute = check_absolute_gates(candidate_metrics)
    relative = check_relative_gate(candidate_metrics["eval_score"], baseline_metrics["eval_score"])

    all_reasons = absolute.blocked_reasons + relative.blocked_reasons
    passed = absolute.passed and relative.passed

    result = {
        "passed": passed,
        "hard_block": absolute.hard_block,
        "reasons": all_reasons,
        "candidate_metrics": candidate_metrics,
        "baseline_metrics": baseline_metrics,
    }
    with open("results/gate_reason.json", "w") as f:
        json.dump(result, f, indent=2)

    if not passed:
        print("RELEASE BLOCKED:")
        for reason in all_reasons:
            print(f"  - {reason}")
        sys.exit(1)   # <-- this is what actually stops the GitHub Actions pipeline

    print("RELEASE APPROVED: all absolute and relative gates passed.")
    sys.exit(0)


if __name__ == "__main__":
    with open("results/tier2.json") as f:
        candidate = json.load(f)
    with open("results/production_baseline.json") as f:
        baseline = json.load(f)
    run_gate(candidate, baseline)
```

This is close to a direct implementation of the pattern Arize's reference CI documentation shows for gating a GitHub Actions job on `experiment.get_evaluations()['score'].mean() > 0.8` — the specific numbers differ by team, but the shape (load metrics, compare to threshold, non-zero exit on failure) is now close to an industry-standard template, not a bespoke invention.

#### Comparison table — absolute vs. relative gates

| Aspect | Absolute gate | Relative gate |
|---|---|---|
| Compares against | A fixed, policy-defined floor | The current production baseline |
| Protects against | Shipping something below minimum acceptable quality | Slow cumulative degradation across many "individually fine" releases |
| Typical examples | Hallucination rate, format validity, P95 latency SLA | Composite score delta vs. production |
| Can be a hard block? | Yes — commonly format & hallucination | Rarely — usually allows a small human-reviewable tolerance band |
| Requires a stored baseline? | No | Yes — must fetch production's current metrics from the registry |
| Failure mode if missing | Ships objectively broken/unsafe output | Ships a release that's "fine" alone but part of a downward trend |

---

### 3.3 Reading Evaluation Dashboards for Release Decisions

#### Theory

A dashboard's job is not to display numbers — plenty of dashboards do that and still fail to prevent bad releases. A dashboard's job is to answer, as fast as possible and with as little interpretation required as possible, the question a release manager actually has: **can we ship this?** The source material's structure for that answer has four layers, always in this order:

1. **Headline status (traffic light).** Green = every important dimension clears its threshold and there's no regression vs. baseline — safe to auto-approve. Yellow = a marginal pass (e.g., within 1% of a threshold) — requires human review, not auto-approval. Red = an absolute gate failed or a regression was detected — auto-blocked, no human override needed to *stop* it (a human can still investigate and re-run).
2. **Dimension breakdown.** The composite score decomposed into correctness, relevance, format, latency, safety — so that "why is it yellow/red" is answerable without re-running anything.
3. **30-day trend.** A rolling time series of the same metrics, because a single evaluation run cannot distinguish "stable good" from "the last data point of a slow decline." Trend view is what makes drift visible before it becomes an incident.
4. **Regression vs. production baseline.** The same comparison the relative gate computes, shown visually so a human can sanity-check the automated decision rather than trust it blindly.

The critical design principle: **the dashboard's headline thresholds must be the exact same thresholds the CI gate code enforces.** If the gate blocks at `eval_score < 0.82` but the dashboard shows green at 0.80, the dashboard is actively misleading — it will train reviewers to distrust or ignore automated blocks. This module treats the CI gate config and the dashboard threshold config as one source of truth, not two independently maintained numbers.

#### Architecture — the release-manager reading order

```
   RELEASE MANAGER OPENS DASHBOARD FOR v2.3.1
                    |
                    v
    +----------------------------------+
    | STEP 1: Headline status           |   composite score 4.32/5 (baseline 4.21) -> GREEN, +2.6%
    |  (traffic light + composite score) |
    +-----------------+------------------+
                       v
    +----------------------------------+
    | STEP 2: Dimension breakdown       |   correctness 4.40  [green]
    |  (per-dimension green/yellow/red) |   p95 latency 1840ms [green]
    |                                    |   format        4.87  [green]
    +-----------------+------------------+
                       v
    +----------------------------------+
    | STEP 3: 30-day trend view         |   stable, no degradation, no drift signal
    +-----------------+------------------+
                       v
    +----------------------------------+
    | STEP 4: Regression vs. production |   vs v2.2.0: all deltas positive
    |  (v2.2.0 baseline comparison)      |
    +-----------------+------------------+
                       v
              DECISION: APPROVE RELEASE
```

Notice this is exactly a depth-first drill from "is it safe" to "why do I believe that" — never the reverse. A dashboard that opens on 40 raw metric tiles with no headline status forces every viewer to redo the aggregation step in their head, which is slow and inconsistent across reviewers.

#### Examples

**Beginner.** A single Grafana panel showing `eval_score` over time with no thresholds, no color, no baseline comparison. Technically a dashboard; not actionable — every viewer has to know the passing threshold from memory or a wiki page.

**Intermediate.** A panel with a threshold line drawn at 0.82 and a red/green background based on the latest value. Better — encodes the absolute gate visually — but still has no dimension breakdown and no baseline comparison, so "why did it drop" still requires opening logs.

**Production-grade.** The full four-layer dashboard described above, backed by the same metrics store the CI gate reads from, with alert rules (§ below) that page on-call the moment a scheduled (Tier 3) drift check goes red — because that failure mode has no CI pipeline to block; it's already live.

#### Code — a minimal structured readiness payload

This is the shape of object the gate script and the dashboard's data source should agree on — it is the contract between "what CI computed" and "what the dashboard renders":

```json
{
  "version": "v2.3.1",
  "baseline_version": "v2.2.0",
  "composite_score": 4.32,
  "baseline_composite_score": 4.21,
  "status": "green",
  "dimensions": {
    "correctness": {"score": 4.40, "status": "green"},
    "format": {"score": 4.87, "status": "green"},
    "p95_latency_ms": {"score": 1840, "status": "green"},
    "hallucination_rate": {"score": 0.011, "status": "green"}
  },
  "trend_30d": "stable",
  "regression_vs_baseline": {
    "correctness_delta": 0.12,
    "format_delta": 0.03,
    "hallucination_rate_delta": -0.004
  }
}
```

#### Grafana dashboard design — panels, thresholds, alerting

A production LLM release-readiness Grafana dashboard, built on metrics exported by the evaluation harness (typically via a Prometheus pushgateway or an OTel metrics exporter — see Module 14 for the instrumentation layer), should be organized as two rows mirroring the two-part shape the industry has converged on: **operational metrics** and **evaluation/quality metrics**, kept visually distinct because they answer different questions ("is it running well" vs. "is it correct/safe").

```
+-----------------------------------------------------------------------------------+
| ROW 1: RELEASE READINESS (headline)                                                |
|  [ Composite Score Stat, green/yellow/red ]   [ vs. Baseline Delta Stat ]           |
|  [ Overall Status Traffic Light ]              [ Last Eval Run Timestamp ]          |
+-----------------------------------------------------------------------------------+
| ROW 2: DIMENSION BREAKDOWN (bar gauge per dimension, thresholded)                    |
|  correctness | relevance | format | hallucination_rate | p95_latency_ms             |
+-----------------------------------------------------------------------------------+
| ROW 3: 30-DAY TREND (time series, one line per dimension, threshold lines overlaid) |
+-----------------------------------------------------------------------------------+
| ROW 4: OPERATIONAL (cost, token volume, request rate, error rate)                    |
+-----------------------------------------------------------------------------------+
```

**Panel-by-panel design:**

| Panel | Visualization | Thresholds (example) | Alert rule |
|---|---|---|---|
| Composite score | Stat panel, big number | Green ≥0.82, Yellow 0.80–0.82, Red <0.80 | Fire if red for >1 evaluation run (avoid single-flake alerts) |
| Hallucination rate | Stat panel | Green <0.02, Yellow 0.02–0.03, Red ≥0.03 | Fire immediately on any red (hard block — zero tolerance) |
| Format valid rate | Stat panel | Green =1.00, Red <1.00 | Fire immediately (hard block) |
| P95 latency | Stat panel + threshold line | Green ≤1000ms, Yellow 1000–1200ms, Red >1200ms | Fire if red on 2 consecutive scrapes |
| 30-day trend | Time series, multi-line | Threshold line overlay per dimension | Fire on sustained downward slope (Grafana's decreasing-trend alert condition), not single-point dips |
| Regression vs. baseline | Bar chart of deltas | Green ≥0, Yellow -0.01 to 0, Red <-0.01 | Fire on red |
| Cost / token volume | Time series | No hard threshold — budget guardrail annotation | Fire on % change vs. 7-day rolling average exceeding a set budget delta |

```yaml
# Example Grafana alert rule (provisioning YAML) for the hard-block hallucination panel
apiVersion: 1
groups:
  - orgId: 1
    name: llm-release-gates
    folder: LLM Release Readiness
    interval: 5m
    rules:
      - uid: hallucination-hard-block
        title: "Hallucination rate hard block breached"
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: llm_eval_hallucination_rate{env="production_candidate"}
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              expression: A
              conditions:
                - evaluator: { type: gt, params: [0.03] }
        noDataState: NoData
        execErrState: Alerting
        for: 0s              # alert immediately — this is a zero-tolerance hard block
        labels:
          severity: critical
          gate_type: hard_block
        annotations:
          summary: "Candidate hallucination rate {{ $values.A }} exceeds hard-block threshold 0.03"
```

Note the `for: 0s` on the hard-block rule — unlike a typical infra alert (which usually debounces for a few minutes to avoid flapping), a hard-block quality gate should page or block *immediately*, because the entire point of a hard block is zero tolerance. Soft/relative gates, by contrast, are reasonable candidates for a short `for:` debounce window to avoid paging on a single noisy evaluation run — LLM judges and sampled evaluations do have run-to-run variance, which is exactly why promptfoo's GitHub Action ships `repeat` / `repeat-min-pass` settings: re-running a flaky check a few times before deciding is often the right response to non-determinism, rather than lowering the threshold or removing the alert.

---

## 4. Real-World Case Studies (Reasoned Inference)

These are informed inferences about how organizations with public engineering-blog patterns would plausibly architect this layer — not confirmed internal specifics.

**A frontier lab shipping a chat assistant (pattern like OpenAI/Anthropic).** Given the scale of daily prompt/system-message/tool-definition iteration at this kind of organization, it is highly plausible that PR-level evaluation is not 50 hand-picked examples but a curated, actively-maintained regression suite spanning safety, instruction-following, and known-failure-mode categories, likely with categorized hard blocks (e.g., specific safety-policy violations) that no amount of aggregate-score improvement can override — mirroring the "hard block on hallucination/format regardless of overall score" pattern this module teaches, just with a richer taxonomy of what counts as a hard block. Given the cost and latency of running a large model as an LLM-judge at this scale, a tiered strategy (cheap heuristic/classifier checks on every PR, full judge-based eval on merge, broader red-team-style sampling on a schedule) is a near-certain architectural choice purely on cost grounds.

**A recommendation/personalization platform (pattern like Netflix/Spotify).** These organizations are well documented (via public engineering blogs) as heavy users of long-running A/B and canary experimentation infrastructure predating the LLM era. It's reasonable to infer that when they add LLM-backed features (conversational search, generated descriptions, playlist naming), they would plug quality gates into that *existing* experimentation platform rather than building a parallel one — meaning the "relative gate vs. baseline" concept in this module likely maps directly onto their existing "treatment vs. control" statistical significance testing infrastructure, just with LLM-specific dimensions (hallucination, tone, format) added as new metrics inside an established framework.

**A large-scale infrastructure/cloud provider (pattern like Google/Microsoft/Amazon/NVIDIA).** At the scale of a hyperscaler serving many internal teams' models, it is plausible that quality gates are offered as a shared *platform capability* — a standardized "release gate" service that internal teams configure with their own thresholds rather than each team reimplementing gate logic — analogous to how Google's SRE practice standardizes error budgets and alerting as a shared discipline rather than a per-team invention (see the Google SRE Book chapter on monitoring distributed systems in this module's references). This would explain why "the gate is a config file, not custom code" is a recurring theme in modern MLOps tooling (promptfoo, DeepEval, Evidently) — it mirrors how large infra orgs standardize this pattern internally.

**A ride-sharing / marketplace platform (pattern like Uber/Airbnb).** These companies are known for extensive internal ML platform tooling (e.g., Uber's publicly documented Michelangelo). It's reasonable to infer that when LLM features (support chat, listing descriptions, trust & safety text classification) are added, the quality-gate logic would be integrated into the same deployment pipeline as their classical ML models rather than treated as a separate LLM-only system — meaning the four-trigger pattern in this module (PR/push/schedule/data-update) likely applies uniformly across both classical-ML and LLM artifacts in such a platform, just with different evaluation harnesses plugged into the same gate framework.

**A data/ML platform vendor (pattern like Databricks).** Given Databricks' public positioning around MLflow (Module 13) as an experiment-tracking and model-registry system, it's a reasonable inference that their internal (and customer-facing) guidance for LLM release gating treats the model/prompt registry's "stage" transitions (staging -> production) as the enforcement point for gates — i.e., a candidate cannot transition to a "production" stage tag until gate criteria attached to that transition pass, which is architecturally identical to the "register artifact only if gate passes" step in this module's GitHub Actions example.

---

## 5. Common Mistakes

1. **Treating the dashboard as the gate.** A chart that shows a regression is not the same as a mechanism that blocks the regression from shipping. If passing the gate requires a human to notice a red panel and manually stop a deploy, the gate will eventually fail to fire during a busy week.
2. **No hard-block category.** Averaging hallucination rate into a single composite score means a large correctness improvement can mathematically outweigh a hallucination regression, producing a "pass" on a release that actually introduced unsafe output.
3. **Relative gate without an absolute gate (or vice versa).** Relative-only gating lets a badly-performing production baseline drag the bar down release after release ("as long as we're not worse than the already-bad thing we shipped last time"). Absolute-only gating misses slow cumulative decay across many individually-passing releases.
4. **Changing thresholds reactively, release by release.** If a threshold is quietly loosened whenever a release is about to fail it, the gate stops meaning anything and the team stops trusting pass/fail results — exactly the failure mode the source material calls out ("changing gates too often weakens trust").
5. **Evaluating against a non-frozen eval set.** If the evaluation examples themselves change between runs, a score change conflates "the model changed" with "the test changed" — defeating the entire point of comparing scores over time. Always version the eval set explicitly (e.g., `eval_set_v1.0`) and treat changing it as a deliberate, reviewed action.
6. **No path from "red" to "action."** A trigger or gate that fires but has no defined downstream response (rollback, page on-call, block merge) is a notification system, not a quality gate. Every gate needs a wired action, not just a wired check.
7. **Single-run evaluation trust for inherently non-deterministic LLM outputs.** Treating one evaluation run's score as ground truth ignores run-to-run variance in sampled generation and LLM-judge scoring; use repeat-and-require-majority-pass strategies (as promptfoo's `repeat`/`repeat-min-pass` options do) rather than a single noisy sample.
8. **Ignoring cost of evaluation as it scales.** A tier-2-style full evaluation run on every PR (instead of only on push) can silently balloon CI spend as PR volume grows; the tiered design in §3.1 exists specifically to prevent this.

---

## 6. Best Practices and Production Tips

**When to use this pattern:** any LLM system with real users, and any system where a bad release has a cost (reputational, financial, safety, regulatory) that exceeds a few dollars and CI minutes of evaluation compute — which describes essentially all production LLM systems.

**When NOT to over-engineer it:** a research notebook, an internal proof-of-concept with no users, or a one-off experiment does not need four trigger points and hard blocks — apply the full pattern at the point where a change can reach production traffic, not earlier.

**Alternatives to consider:**
- For teams without in-house eval infrastructure, promptfoo's GitHub Action (`fail-on-threshold`, `repeat`, `repeat-min-pass`) or DeepEval's pytest-native `deepeval test run` can implement the PR/push tiers in a few dozen lines of CI config rather than a custom harness.
- For RAG-heavy systems, Ragas' synthetic test-set generation can help keep the frozen eval set fresh without manual authoring, while still keeping the set version-pinned between comparison runs.
- For continuous production monitoring (the schedule/data-update triggers), Evidently's Test Suites or Arize Phoenix's Experiments API can serve as the scoring engine that feeds the same gate logic shown in §3.2.

**Cost and scaling:** the tiered strategy is the primary lever — keep Tier 1 cheap and frequent, Tier 2 moderate and merge-gated, Tier 3 expensive and rare. As PR volume grows, resist the temptation to add more checks to Tier 1; instead consider raising the sample size or check depth only at Tier 2/3, and monitor weekly evaluation spend as its own tracked metric (it belongs on the operational row of the dashboard, not hidden in a cloud bill).

**Monitoring the gates themselves:** track gate pass/fail rate over time as a metric. A gate that fails 40% of the time is either catching real problems (investigate root cause) or is miscalibrated (investigate threshold validity) — either way, a gate's own failure rate is a signal worth dashboarding, not just its pass/fail outcome per run.

**Security:** evaluation datasets, especially ones sampled from production traffic (Tier 3), can contain PII or sensitive customer data — treat frozen eval sets with the same access controls and anonymization requirements as production data itself, not as "just test fixtures."

**Performance tradeoffs:** running a full 200-example, multi-metric evaluation on every single PR (rather than gating it to push-to-main) will slow developer iteration loops and inflate CI cost with little corresponding safety benefit, since most PRs get revised before merge anyway — this is precisely why the tiered design exists.

**Alert fatigue:** a hard-block alert should fire immediately (`for: 0s`) and rarely, because it represents a genuinely zero-tolerance condition. A soft/relative-gate alert should debounce briefly and possibly aggregate over a few runs, because LLM evaluation has run-to-run variance and a single flaky sample should not page anyone at 3 a.m.

---

## 7. Interview Questions

1. **"Why do you need four different evaluation trigger points instead of just running eval before every deploy?"**
   *Model answer:* Because "deploy" is not the only event that can change system behavior. PR and push catch code-driven changes at different confidence/cost tiers; schedule catches drift in an already-deployed, unchanged model as the world around it shifts; data-update catches regressions introduced by a changed corpus/fine-tuning set with no corresponding code diff. Each covers a distinct failure surface the others miss.

2. **"What's the difference between an absolute gate and a relative gate, and why do you need both?"**
   *Model answer:* Absolute gates enforce a fixed minimum bar (e.g., hallucination rate < 3%) regardless of history; relative gates compare the candidate against the current production baseline to catch regressions even when the candidate still clears the absolute floor. Absolute-only gating misses slow cumulative decay across releases; relative-only gating lets a bad baseline anchor future releases too low. Both are needed to protect against different failure modes.

3. **"Why should format-validity and hallucination checks be hard blocks with no override, while a latency regression might not be?"**
   *Model answer:* Hard blocks are reserved for failure modes with no acceptable "ship now, fix later" story — a malformed output can break the calling application immediately, and a hallucination actively misinforms a user. A latency regression is undesirable but often survivable with a fast-follow fix, so it's reasonable to gate it without making it fully non-negotiable. The distinction is a deliberate policy choice about which risks are acceptable to defer.

4. **"How would you design a release dashboard so a release manager can make a ship/hold decision in under a minute?"**
   *Model answer:* Structure it depth-first: headline traffic-light status and composite score first (answers "can we ship"), dimension breakdown second (answers "why, if not green"), 30-day trend third (answers "is this a one-off or a pattern"), and regression-vs-baseline last (answers "is this specifically worse than what's live"). Critically, the thresholds driving dashboard color must be identical to the thresholds the CI gate enforces — otherwise the dashboard misleads reviewers about what will actually happen in CI.

5. **"A release keeps passing your gate, but users are reporting quality problems in production. What would you check?"**
   *Model answer:* First check whether the eval set is frozen and still representative — a stale or narrow eval set can pass while missing categories of real user queries. Second, check whether this is a Tier-3/schedule-detectable drift issue rather than something a PR/push gate could ever catch (the model may not have changed; the input distribution did). Third, verify the gate's thresholds haven't been loosened over time, and check the gate's own pass-rate trend for signs of miscalibration.

6. **"How do you handle the non-determinism of LLM outputs when a gate check occasionally fails due to sampling variance rather than a real regression?"**
   *Model answer:* Don't lower the threshold to compensate for flakiness — that erodes the gate's meaning. Instead use a repeat-and-require-majority-pass strategy (run the check N times, require a minimum pass rate), which is exactly what promptfoo's GitHub Action's `repeat`/`repeat-min-pass` options implement, and apply a short alert debounce window for soft gates while keeping hard blocks (format, safety) alerting immediately and without debounce.

7. **"Design the weekly cost budget conversation: how would you justify a $14/week evaluation spend to a skeptical engineering manager?"**
   *Model answer:* Frame it against the counterfactual cost of one undetected regression reaching users — a hallucination incident, a broken output format breaking downstream parsing, or a slow cumulative quality decline that silently erodes user trust over months. $14/week amortized across 20 PRs and 5 pushes is a rounding error next to the cost of an incident response, a customer-trust hit, or (in regulated domains) a compliance failure. The tiered design itself is the cost-control mechanism — it's not "spend more," it's "spend proportionally to risk and change frequency."

8. **"What would you do if a teammate proposed manually overriding a hard-block gate failure to hit a release deadline?"**
   *Model answer:* Decline the override and treat it as a process signal, not a one-off exception — if the deadline pressure is real and recurring, the fix is to improve the pipeline's speed or catch the issue earlier (shift left into Tier 1), not to erode the meaning of "hard block." If overrides become common, the organization has effectively downgraded the hard block to a soft gate without saying so, which reintroduces the exact risk (unsafe/broken output reaching users) the hard block existed to prevent.

---

## 8. Summary, Key Takeaways, and Production Checklist

**Summary.** Evaluation only prevents regressions if it is (a) triggered automatically at the right moments, (b) enforced through gates that block promotion rather than just reporting results, and (c) made legible through a dashboard whose thresholds mirror the gate's thresholds exactly. This module covered all three layers: four trigger points (PR, push, schedule, data-update) organized into a cost-aware three-tier strategy; absolute and relative gates with an explicit hard-block category for zero-tolerance failure modes (format, hallucination); and a four-layer dashboard reading order (headline status, dimension breakdown, trend, regression-vs-baseline) that turns metrics into a fast, defensible ship/hold/rollback decision.

**Key takeaways:**
- Triggers, gates, and dashboards are three separate concerns that must all exist together — a strong evaluation function wired to only one of the three still leaves regressions reaching production.
- Tiering evaluation depth to trigger frequency (cheap/fast on PR, full on push, broad/expensive on schedule) is what makes frequent evaluation economically sustainable.
- Absolute and relative gates protect against different failure modes and are both required; hard blocks are a deliberate zero-tolerance policy choice, not a technical default.
- A gate without a wired action (block, page, rollback) is a report, not a gate — `sys.exit(1)` inside CI is what actually stops a bad release, not a red number on a chart.
- Dashboard thresholds must be the same numbers as gate thresholds — divergence between the two erodes trust in both.

**Production checklist:**
- [ ] PR-triggered fast evaluation exists, runs on every pull request, and blocks merge on failure.
- [ ] Push-to-main triggers a full evaluation before any artifact is registered.
- [ ] A scheduled (weekly or more frequent, based on risk) drift/hallucination check runs against a sample of production or recent traffic.
- [ ] Evaluation re-runs automatically whenever training data, RAG corpus, or fine-tuning set changes.
- [ ] Absolute gates are defined for quality, latency, and safety/hallucination, each with an explicit numeric threshold.
- [ ] A relative gate compares every candidate against the current production baseline, not just a fixed floor.
- [ ] Format-validity and hallucination are explicitly coded as hard blocks with no override path.
- [ ] The evaluation set used for gating is version-pinned/frozen, and changes to it are deliberate and reviewed.
- [ ] Every gate failure produces a machine-readable reason (not just a boolean) surfaced on the PR/dashboard.
- [ ] The CI gate's threshold values and the dashboard's panel threshold values are defined from one shared source of truth.
- [ ] Alert rules exist for hard-block panels (immediate, no debounce) and soft/relative panels (short debounce, tolerant of run-to-run variance).
- [ ] Weekly evaluation cost is itself tracked as a dashboard metric.
- [ ] There is a documented, non-silent process for changing a gate threshold (review required, not a quiet edit under deadline pressure).

---

## 9. Further Reading

Detailed citations, official documentation links, notable open-source repositories (promptfoo, DeepEval, Ragas, Evidently, Arize Phoenix), and supplementary videos/books referenced throughout this module are collected in `references.md`, `videos.md`, `books.md`, and `github.md` in this same module folder — refer to those files for source-level detail rather than repeating them here.
