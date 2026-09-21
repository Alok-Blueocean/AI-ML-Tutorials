# Module 09 — Prompt Lifecycle: Versioning and Statistical Evaluation

> Part of the LLMOps/MLOps senior-engineer curriculum. Builds on Modules 01-06 (CI/CD foundations, versioning/registries/rollback, reproducibility/environments, Docker, Kubernetes). This module assumes you are comfortable with model registries, semantic versioning, immutable artifacts, and rollback patterns from Module 02 — here we apply the same discipline to a new artifact type: **the prompt**.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Explain why a prompt is a first-class, versioned production artifact — not a string literal buried in application code.
2. Design an **immutable prompt registry** with semantic versioning, rich metadata, and alias-based promotion (dev → staging → production), including rollback semantics.
3. Build a **delta-tracking pipeline** that compares a candidate prompt against a baseline on a fixed evaluation set across multiple metrics (correctness, latency, cost, format/behavior).
4. Apply **paired statistical significance testing** (paired t-test) correctly to prompt evaluation data, and explain why an unpaired test or an ad hoc sample size would produce misleading promotion decisions.
5. Perform **power analysis** to choose an evaluation sample size before you run an experiment, instead of guessing.
6. Combine **statistical significance** with a **practical significance threshold** to build an auto-promote/auto-block/manual-review decision gate, and implement it as a CI-style automated check (`delta_check.py` pattern).
7. Critique this design the way a senior engineer would in an interview: know its failure modes, its costs, and when a full statistical framework is overkill.

### Prerequisites

- Module 02 (Versioning, Registries, Rollback) — the concepts of immutable artifact versions, semantic versioning, and alias/tag-based promotion translate almost directly.
- Module 03 (Reproducibility) — you already know why pinning environment and data matters; here we pin the *evaluation dataset* the same way.
- Basic Python: dataclasses, functions, working with NumPy/pandas-like arrays.
- Basic statistics: mean, standard deviation, what a p-value roughly means (we will re-derive the intuition here, so a rusty memory is fine — but if you have never heard of a t-test, expect to reread section 3.4 twice).
- Familiarity with CI/CD gating from Module 01 (a pipeline stage that can pass/fail a build) — we reuse that mental model for prompt promotion gates.

### Key Terminology

| Term | Definition |
|---|---|
| **Prompt version** | An immutable, uniquely identified snapshot of a prompt template plus its generation parameters (temperature, model, top_p, max_tokens, etc.). |
| **Prompt registry** | A system of record that stores prompt versions, their metadata, and the current alias-to-version mapping (e.g., `production → v1.4.0`). |
| **Semantic versioning (for prompts)** | `MAJOR.MINOR.PATCH` applied to prompt changes: PATCH for wording/formatting tweaks with no intended behavior change, MINOR for behavior-affecting changes that are backward-compatible in output contract, MAJOR for changes to the output contract/schema or fundamental task redefinition. |
| **Baseline** | The currently-promoted (usually production) prompt version, used as the point of comparison for a candidate. |
| **Candidate** | A new prompt version being evaluated for promotion. |
| **Fixed evaluation set (eval set)** | A frozen, versioned set of input examples (and, where available, reference answers) used identically for baseline and candidate so the comparison is fair. |
| **Delta** | The measured difference between candidate and baseline on a given metric (correctness, latency, cost, format-error rate, etc.), on the same eval set. |
| **Paired test** | A statistical test that compares two measurements taken on the *same* unit (here: the same eval example scored by both baseline and candidate), which removes per-example variance from the noise term. |
| **Statistical significance** | The observed difference is unlikely to have arisen from random sampling noise alone (conventionally, p < 0.05). |
| **Practical significance** | The observed difference, even if statistically real, is large enough to matter operationally/business-wise (e.g., lift > 3%). |
| **Statistical power** | The probability that a test correctly detects a true effect of a given size, given the sample size and noise level. Conventionally targeted at 80%. |
| **Auto-promote gate** | An automated policy that promotes a candidate to the next environment only if it passes a combination of significance + practical-significance + multi-metric non-regression rules, with no human in the loop. |
| **Prompt drift** | Unintended behavioral change in a prompt's output distribution (tone, structure, verbosity) that may not show up in a single "correctness" number. |
| **LLM-as-judge** | Using a (typically stronger) LLM to score the correctness/quality of another LLM's output, standing in for a metric that classic string-matching (BLEU/exact-match) cannot capture for open-ended generation. |

---

## 2. Why This Topic Matters, and Where It Fits in the Lifecycle

In classical MLOps, the artifact you version, test, and roll back is a trained model binary. In LLMOps, the artifact that changes most often — sometimes multiple times per day — is the **prompt**. And yet, in a great many real codebases, the prompt is still a Python f-string sitting inline in a request handler, edited by whoever is on call that day, with no version, no diff, no test, and no rollback path. That is the single most common "amateur hour" pattern this module exists to eliminate.

Think about what a prompt actually is, functionally: it is the *executable configuration* of a stochastic function (the LLM call). Changing the system prompt, the few-shot examples, the output-format instructions, or the temperature is functionally equivalent to changing a model's hyperparameters or even swapping in a different model checkpoint. If your organization would never let someone hot-patch a fraud-detection model's threshold directly in production without review, versioning, and a rollback plan, there is no principled reason to treat prompt edits any differently. Prompts are code that runs on a model instead of a CPU, and they deserve the same lifecycle discipline: version control, staged promotion, automated regression testing, and statistically defensible go/no-go decisions.

Where this sits in the broader LLMOps lifecycle:

```
 ┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌────────────────┐   ┌─────────────┐
 │  Data /      │   │  Model        │   │  Prompt        │   │  Evaluation /  │   │  Serving /  │
 │  Feature     │──▶│  Training /   │──▶│  Engineering & │──▶│  Statistical   │──▶│  Monitoring │
 │  Pipelines   │   │  Fine-Tuning  │   │  Versioning    │   │  Promotion Gate│   │  (Module 10+)│
 │ (Modules 01-06)  │  (Modules 01-06) │  <<< THIS MODULE│   │  <<< THIS MODULE│  │             │
 └─────────────┘   └──────────────┘   └───────────────┘   └────────────────┘   └─────────────┘
```

For teams building on top of *foundation* models (i.e., not training your own base model — the overwhelmingly common case in 2026), prompt engineering and its lifecycle **is** the primary "model development" loop. The registry, the CI-style delta gate, and the statistical promotion test described in this module are the direct analog of what Modules 01-06 taught you for trained-model artifacts, transplanted onto prompts. If your organization has excellent model CI/CD but prompts are still edited ad hoc in a Slack thread and copy-pasted into production, you have not actually solved LLMOps — you have solved half of it.

This also directly sets up Module 10+ (typically observability/monitoring and continual evaluation in production): the registry and gate built here are what feeds monitoring dashboards with "which prompt version is live, when did it change, and what did we expect vs. observe" — without this module, production monitoring has no baseline to compare against.

---

## 3. Main Concepts

### 3.1 Prompts as Immutable, Versioned Artifacts

#### Theory

**What problem does this solve?** Without versioning, "the prompt" is a moving target. Two engineers debugging a production incident from last Tuesday cannot reconstruct what the model actually saw, because the prompt file has since been edited three more times. Reproducibility (Module 03's core concern) completely breaks down: the same *code* plus the same *input* no longer produces the same *output*, because the third silent dependency — the prompt text and its sampling parameters — was never pinned.

**Why immutability specifically?** Mutable "current prompt" files are the prompt-engineering equivalent of `latest` as a Docker tag (which Module 04 already told you to avoid in production). If version `1.4.0` can be silently edited in place, then any log line, trace, or eval report that says "this used v1.4.0" becomes worthless the moment someone touches the file again. Immutability is what makes a version number a reliable, permanent reference. This is precisely the design MLflow's GenAI Prompt Registry and LangSmith's Prompt Hub both converge on: every save creates a new, permanent, content-addressed or sequentially-numbered version; nothing already published is ever edited in place (see references.md for the exact APIs).

**Why semantic versioning?** A bare incrementing integer (`v37`) tells a reviewer nothing about the *blast radius* of a change. Borrowing MAJOR.MINOR.PATCH from software gives every stakeholder — including non-engineers reviewing a changelog — an immediate sense of risk:

| Segment | Prompt-engineering meaning | Example |
|---|---|---|
| **PATCH** (`1.3.2 → 1.3.3`) | Cosmetic/wording tweak, no intended behavior change (typo fix, whitespace, punctuation) | Fixing "recieve" → "receive" in a static instruction |
| **MINOR** (`1.3.x → 1.4.0`) | Behavior-affecting change that keeps the same input/output contract (new few-shot example, reworded instruction, temperature tweak) | Adding a clarifying instruction that measurably improves correctness |
| **MAJOR** (`1.x.x → 2.0.0`) | Breaking change to the output contract, task redefinition, or model swap that downstream code must adapt to | Switching output from free text to strict JSON schema; changing target model family |

**Tradeoffs / when NOT to bother with the full machinery:**
- For a prototype or a single-developer side project with no production traffic, a full registry is overkill — a Git-tracked prompt file with commit messages may be enough (this is itself "version 0" of a registry; do not skip straight to a database-backed system before you need one).
- Full statistical promotion gating (section 3.4) is expensive to run for every trivial PATCH-level wording fix — most registries let PATCH-level changes skip the full statistical gate and instead require only a smoke test, reserving the expensive paired-significance pipeline for MINOR/MAJOR candidate promotions.

#### Architecture

```
                         PROMPT REGISTRY (immutable store)
        ┌───────────────────────────────────────────────────────────────┐
        │                                                               │
        │   version: 1.3.2  (immutable)      version: 1.4.0 (immutable)│
        │   ┌─────────────────────────┐       ┌─────────────────────────┐
        │   │ system_prompt: "..."     │       │ system_prompt: "..."     │
        │   │ temperature: 0.2         │       │ temperature: 0.2         │
        │   │ model: gpt-4o-2024-08-06 │       │ model: gpt-4o-2024-08-06 │
        │   │ eval_dataset_version: 7  │       │ eval_dataset_version: 7  │
        │   │ author: a.ranjan         │       │ author: a.ranjan         │
        │   │ commit_message: "..."    │       │ commit_message: "..."    │
        │   │ created_at: 2026-06-01   │       │ created_at: 2026-07-20   │
        │   └─────────────────────────┘       └─────────────────────────┘
        │              ▲                                  ▲
        │              │                                  │
        │       alias: production                  alias: staging
        │       (mutable pointer)                  (mutable pointer)
        └───────────────────────────────────────────────────────────────┘
                              │                            │
                              ▼                            ▼
                     ┌─────────────────┐         ┌───────────────────┐
                     │  Serving layer   │         │  Delta / stat-test │
                     │  resolves alias  │         │  pipeline runs     │
                     │  → version each  │         │  candidate vs.     │
                     │  request (or on  │         │  baseline (§3.2/3.4)│
                     │  TTL cache)      │         │                    │
                     └─────────────────┘         └───────────────────┘
```

Key architectural point: **versions are immutable, aliases are mutable pointers.** Promotion is never "edit the production prompt" — it is always "repoint the `production` alias at a different, already-existing, already-tested version number." This is identical in spirit to a Kubernetes Deployment repointing at a new immutable container image tag (Module 04/05), and to a model registry's `Production` stage tag in Module 02. Rollback becomes trivial and instantaneous: repoint the alias back to the previous version. No re-deploy of application code is required if the application resolves the alias at request time (with the caching caveat below).

#### Examples

**Beginner — a hand-rolled JSON-lines immutable log**

```python
import json
import hashlib
import time
from pathlib import Path

REGISTRY_PATH = Path("prompt_registry.jsonl")

def log_prompt_version(version: str, system_prompt: str, temperature: float,
                        model: str, eval_dataset_version: str, author: str,
                        commit_message: str) -> dict:
    """Append-only write. Never edits an existing line -- that is the
    immutability guarantee at the simplest possible level."""
    entry = {
        "version": version,
        "system_prompt": system_prompt,
        "temperature": temperature,
        "model": model,
        "eval_dataset_version": eval_dataset_version,
        "author": author,
        "commit_message": commit_message,
        "created_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
    }
    # content hash makes tampering/edits after the fact detectable
    entry["content_hash"] = hashlib.sha256(
        json.dumps(entry, sort_keys=True).encode()
    ).hexdigest()[:12]

    with REGISTRY_PATH.open("a", encoding="utf-8") as f:
        f.write(json.dumps(entry) + "\n")
    return entry
```

This is "version 0" of a registry: append-only, auditable, diffable with `git diff` on the JSONL file. It has no query API, no alias mechanism, and no access control — but it already buys you the two rules the source material insists on: immutability and full metadata capture.

**Intermediate — a minimal registry class with alias-based promotion**

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional
import copy

@dataclass(frozen=True)
class PromptVersion:
    version: str                  # semantic version, e.g. "1.4.0"
    system_prompt: str
    temperature: float
    model: str
    eval_dataset_version: str
    author: str
    commit_message: str
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())

class PromptRegistry:
    """In-memory reference implementation. Swap the dict for a real
    database (Postgres table + a KV store for aliases) in production."""

    def __init__(self):
        self._versions: dict[str, PromptVersion] = {}
        self._aliases: dict[str, str] = {}   # alias_name -> version
        self._alias_history: dict[str, list[tuple[str, str]]] = {}  # audit trail

    def register(self, pv: PromptVersion) -> None:
        if pv.version in self._versions:
            raise ValueError(
                f"Version {pv.version} already exists and is immutable; "
                f"bump the version instead of overwriting."
            )
        self._versions[pv.version] = pv

    def get(self, version: str) -> PromptVersion:
        return self._versions[version]

    def promote(self, alias: str, version: str, actor: str) -> None:
        if version not in self._versions:
            raise ValueError(f"Cannot promote unknown version {version}")
        self._alias_history.setdefault(alias, []).append((version, actor))
        self._aliases[alias] = version

    def resolve(self, alias: str) -> PromptVersion:
        return self._versions[self._aliases[alias]]

    def rollback(self, alias: str, actor: str) -> str:
        """Roll back to the version immediately before the current one
        for this alias. O(1) -- no re-run of anything needed because the
        prior version object is fully intact and immutable."""
        history = self._alias_history.get(alias, [])
        if len(history) < 2:
            raise RuntimeError(f"No prior version to roll back to for alias '{alias}'")
        previous_version, _ = history[-2]
        self.promote(alias, previous_version, actor=f"{actor}(rollback)")
        return previous_version
```

**Production-grade — MLflow's GenAI Prompt Registry (real API)**

MLflow 3.x ships this exact pattern as a first-class citizen (see references.md for the doc link). Production code looks like:

```python
import mlflow

# Register a new, immutable version. MLflow assigns the version number
# and stores commit message + arbitrary tags (author, temperature, model, etc.)
prompt = mlflow.genai.register_prompt(
    name="support-ticket-classifier",
    template="You are a support triage assistant.\n\n{{ticket_text}}\n\nRespond with one of: {{categories}}.",
    commit_message="Add explicit category enumeration to reduce free-text drift",
    tags={
        "author": "a.ranjan@castsoftware.com",
        "temperature": "0.0",
        "model": "gpt-4o-2024-08-06",
        "eval_dataset_version": "v7",
    },
)

# Alias-based promotion -- zero application redeploy required
mlflow.genai.set_prompt_alias(
    name="support-ticket-classifier", alias="production", version=prompt.version
)

# Application code always resolves by alias, never by hardcoded version
live_prompt = mlflow.genai.load_prompt(
    "prompts:/support-ticket-classifier@production"
)
```

Rollback is a single call: repoint `production` at the prior version number — no re-registration, because the old version was never deleted or mutated.

#### Comparison Table — Registry Approaches

| Approach | Immutability guarantee | Alias/rollback support | Query/audit | Best for |
|---|---|---|---|---|
| Prompt string in app code, edited in place | None | None (requires code redeploy) | Only via `git blame` on app repo | Never, in production |
| Git-tracked prompt files (YAML/JSON per version) | Strong, via Git history | Manual (checkout a commit) | Full via `git log` | Small teams, low prompt-change velocity |
| Hand-rolled append-only JSONL/DB table (examples above) | Enforced in code | Custom-built | Custom queries | Teams needing a lightweight in-house solution before adopting a platform |
| MLflow GenAI Prompt Registry | Enforced by platform | Native (aliases + history) | Native UI + API, linked to eval runs/traces | Teams already on MLflow for model tracking |
| LangSmith Prompt Hub | Enforced by platform (commit-based) | Native (tags per environment) | Native UI, diff view | Teams already on LangSmith/LangChain tracing |
| Commercial (e.g., PromptLayer) | Enforced by platform | Native, plus non-engineer UI | Native, with release labels | Teams wanting non-engineers to iterate without redeploys |

#### The rollback-caching gotcha (an important operational nuance)

An alias repoint is instantaneous at the registry level, but **already-running application instances may have a cached copy of the resolved prompt version**. LangSmith's Prompt Hub documents this explicitly: a default cache TTL (on the order of ~5 minutes) means a rollback can take that long to fully propagate unless the application forces a cache invalidation or reads through on every request. This is exactly the same class of problem as DNS TTLs or Kubernetes ConfigMap propagation delays from Module 05 — **know your cache invalidation story before you need it during an incident.** During an active incident, "I already rolled back" and "the rollback has actually reached every pod" are two different facts, and confusing them costs minutes you don't have.

---

### 3.2 Delta Tracking: Multi-Metric Comparison on a Fixed Eval Set

#### Theory

**What problem does this solve?** A single "it feels better" or even a single aggregate accuracy number hides tradeoffs. A prompt change that improves correctness by asking the model to "think step by step and double-check your answer" will very plausibly also increase latency and token cost. A change that tightens the output format instructions might improve parseability but subtly shift tone in a way that a brand/style reviewer would flag but a correctness metric would not. The core insight from the source material is: **a prompt version is only meaningful in reference to a measured behavioral delta against its predecessor — not in isolation.**

**Why a *fixed* evaluation set specifically?** If baseline and candidate are evaluated on different examples (or the same examples resampled/reshuffled), any observed difference is confounded by *which examples happened to be easy or hard this time*, not just by the prompt change itself. This is the same reproducibility principle from Module 03 (pin your data) applied to evaluation: freeze and version the eval dataset, and never let it silently drift between comparisons.

**Why multiple metrics?** Because a production LLM system's cost function is genuinely multi-dimensional: correctness, latency, $ cost per call, and format/behavioral stability (tone, structure, refusal rate) all matter, and they trade off against each other. Optimizing only for correctness is how teams end up with a "better" prompt that quietly triples their inference bill or introduces a subtle tone regression that damages user trust — a class of failure a single-metric gate is structurally blind to.

**Tradeoffs:**
- More metrics = more thresholds to tune and more false-positive blocks from noisy secondary metrics. Over-gating on a jittery p95-latency measurement can block genuinely good prompt changes.
- Fully automated blocking on subjective metrics (tone, brand voice) is risky — the source material is explicit that some deltas (e.g., tone-shift score above a threshold) should route to **human review**, not auto-block or auto-promote. Automating away human judgment entirely on qualitative dimensions is itself a common mistake (see section 5).

#### Architecture

```
   Eval Set (frozen, versioned: v7, N=200 examples)
        │
        ├────────────────────────────┬───────────────────────────────┐
        ▼                            ▼                                │
 ┌────────────────┐          ┌────────────────┐                       │
 │ Baseline v1.3.2 │          │ Candidate v1.4.0│                      │
 │ run on all N     │          │ run on all N     │                     │
 └────────────────┘          └────────────────┘                       │
        │                            │                                │
        ▼                            ▼                                │
 per-example: {correctness, latency_ms, cost_usd, format_ok}   (same schema)
        │                            │                                │
        └─────────────┬──────────────┘                                │
                       ▼                                               │
              ┌─────────────────────┐                                  │
              │   delta_check.py     │◀─────────────────────────────────┘
              │  compute deltas per   │      (thresholds config,
              │  metric, aggregate    │       versioned alongside code)
              └─────────────────────┘
                       │
        ┌──────────────┼───────────────┬─────────────────────┐
        ▼               ▼                ▼                     ▼
   All deltas OK   Any hard metric   Tone/format soft      Any hard metric
   → AUTO-PROMOTE  regresses beyond  metric drifts beyond   passes but is
   to staging       threshold →      threshold →             borderline →
                    AUTO-BLOCK       ROUTE TO HUMAN REVIEW    ROUTE TO HUMAN
                                                               REVIEW
```

#### Examples

**Beginner — computing deltas by hand for one metric**

```python
baseline_correctness = 0.880   # v1.3.2, N=200
candidate_correctness = 0.914  # v1.4.0, N=200

delta = candidate_correctness - baseline_correctness
pct_change = delta / baseline_correctness

print(f"correctness delta: {delta:+.3f} ({pct_change:+.1%})")
# correctness delta: +0.034 (+3.9%)
```

**Intermediate — multi-metric delta report matching the source transcript's numbers**

```python
from dataclasses import dataclass

@dataclass
class VersionMetrics:
    version: str
    correctness: float      # mean over eval set, e.g. LLM-judge score in [0,1]
    p95_latency_ms: float
    cost_per_call_usd: float
    format_error_rate: float = 0.0

baseline = VersionMetrics(version="1.3.2", correctness=0.880,
                           p95_latency_ms=1820, cost_per_call_usd=0.00727)
candidate = VersionMetrics(version="1.4.0", correctness=0.914,
                            p95_latency_ms=1640, cost_per_call_usd=0.00712)

def delta_report(baseline: VersionMetrics, candidate: VersionMetrics) -> dict:
    return {
        "correctness_delta_pct": (candidate.correctness - baseline.correctness) / baseline.correctness,
        "latency_delta_pct": (candidate.p95_latency_ms - baseline.p95_latency_ms) / baseline.p95_latency_ms,
        "cost_delta_pct": (candidate.cost_per_call_usd - baseline.cost_per_call_usd) / baseline.cost_per_call_usd,
    }

report = delta_report(baseline, candidate)
# correctness_delta_pct: +0.039   latency_delta_pct: -0.099   cost_delta_pct: -0.021
```

Note: correctness up, latency *down* (faster, negative is good), cost *down* (cheaper, negative is good) — all three deltas favor the candidate, matching the "auto-promote" narrative from the transcript.

**Production-grade — a `delta_check.py` CI gate with mixed auto-block / manual-review outcomes**

```python
"""delta_check.py -- run as a CI/CD gate step (Module 01 style) whenever a
new candidate prompt version is registered. Exit code drives the pipeline:
  0 = auto-promote allowed
  1 = auto-blocked, promotion refused
  2 = passed hard gates but flagged for manual review
"""
import sys
import json
from dataclasses import dataclass

@dataclass
class Thresholds:
    max_correctness_drop: float = 0.02      # block if correctness drops > 2%
    max_latency_regression_pct: float = 0.15  # block if p95 latency worsens > 15%
    max_cost_regression_pct: float = 0.20     # block if cost worsens > 20%
    max_format_error_rate: float = 0.01       # block if >1% outputs fail format validation
    tone_shift_review_threshold: float = 0.15  # route to human review, do not auto-block

THRESHOLDS = Thresholds()

def evaluate_gate(baseline: dict, candidate: dict, tone_shift_score: float) -> dict:
    correctness_delta = candidate["correctness"] - baseline["correctness"]
    latency_delta_pct = (candidate["p95_latency_ms"] - baseline["p95_latency_ms"]) / baseline["p95_latency_ms"]
    cost_delta_pct = (candidate["cost_per_call_usd"] - baseline["cost_per_call_usd"]) / baseline["cost_per_call_usd"]

    hard_blocks = []
    if correctness_delta < -THRESHOLDS.max_correctness_drop:
        hard_blocks.append(f"correctness dropped {correctness_delta:.1%}")
    if latency_delta_pct > THRESHOLDS.max_latency_regression_pct:
        hard_blocks.append(f"p95 latency regressed {latency_delta_pct:.1%}")
    if cost_delta_pct > THRESHOLDS.max_cost_regression_pct:
        hard_blocks.append(f"cost regressed {cost_delta_pct:.1%}")
    if candidate["format_error_rate"] > THRESHOLDS.max_format_error_rate:
        hard_blocks.append(f"format error rate {candidate['format_error_rate']:.1%} exceeds cap")

    needs_review = tone_shift_score > THRESHOLDS.tone_shift_review_threshold

    if hard_blocks:
        decision = "BLOCK"
    elif needs_review:
        decision = "MANUAL_REVIEW"
    else:
        decision = "AUTO_PROMOTE"

    return {
        "decision": decision,
        "hard_blocks": hard_blocks,
        "tone_shift_score": tone_shift_score,
        "needs_review": needs_review,
        "deltas": {
            "correctness": correctness_delta,
            "latency_pct": latency_delta_pct,
            "cost_pct": cost_delta_pct,
        },
    }

if __name__ == "__main__":
    baseline = json.loads(sys.stdin.readline())
    candidate = json.loads(sys.stdin.readline())
    tone_shift_score = float(sys.argv[1]) if len(sys.argv) > 1 else 0.0

    result = evaluate_gate(baseline, candidate, tone_shift_score)
    print(json.dumps(result, indent=2))

    exit_code = {"AUTO_PROMOTE": 0, "BLOCK": 1, "MANUAL_REVIEW": 2}[result["decision"]]
    sys.exit(exit_code)
```

This is the direct code realization of the transcript's `delta_check.py` narrative: "if accuracy drops by 2%, block; if tone shift rises above 0.15, send to manual review; log all deltas." Wired into a GitHub Actions workflow (Module 01 patterns), exit code `1` fails the build (blocking a merge/deploy), exit code `2` can post a comment tagging a human reviewer instead of failing the pipeline outright.

```yaml
# .github/workflows/prompt-promotion-gate.yml (excerpt)
- name: Run delta check against production baseline
  run: |
    python delta_check.py "${{ steps.tone.outputs.score }}" \
      < <(cat baseline_metrics.json candidate_metrics.json)
  continue-on-error: true   # capture exit code 2 (manual review) without failing the job outright
  id: gate
- name: Fail build on hard block
  if: steps.gate.outputs.exit_code == '1'
  run: exit 1
- name: Request manual review
  if: steps.gate.outputs.exit_code == '2'
  run: gh pr comment "$PR_NUMBER" --body "Prompt candidate flagged for manual review: tone-shift score exceeded threshold."
```

---

### 3.3 Statistical Significance Testing for Prompt Promotion

#### Theory

**What problem does this solve?** Delta tracking (3.2) tells you *what* changed. It does not tell you whether that change is *real* or just noise. Evaluation scores — especially LLM-judge-based correctness scores — are themselves stochastic: rerun the exact same prompt version on the exact same eval set with a nonzero temperature (or even at temperature 0, against a judge model with its own sampling variance) and you will not get bit-for-bit identical aggregate scores. A candidate showing +3.4% correctness could be a genuine improvement, or it could be that this particular batch of 200 examples happened to contain slightly more questions the candidate's phrasing handles well, by chance.

**Why a paired test, specifically?** Because baseline and candidate are evaluated on the *same* set of examples, each example's two scores (baseline score, candidate score) are naturally paired — some examples are simply harder than others for *both* models, and that per-example difficulty is a large source of variance that has nothing to do with which prompt is better. A **paired** test (e.g., a paired t-test on per-example score differences) cancels out that shared per-example variance, leaving only the variance attributable to the *prompt difference itself*. An **unpaired** test (treating the two score arrays as independent samples) throws that cancellation away, inflates the estimated noise, and makes it much harder to detect a real but modest improvement — or, applied incorrectly to genuinely paired data, can also invalidate the test's assumptions (see the Dror et al. ACL 2018 paper in references.md, which is explicitly about avoiding exactly these misapplications).

**Why n ≥ 200, and where does that number come from?** It is not an arbitrary round number — it is the output of a **power analysis**: given the expected effect size (here, ~3% quality difference) and the observed variance in per-example scores, how large a sample is needed to have an acceptable (conventionally 80%) probability of detecting that effect if it is real, at a chosen significance level (conventionally α = 0.05)? Section 3.5 below shows exactly how to derive this number yourself rather than take it on faith — a senior engineer should be able to justify *why* 200 and not 80 or 500.

**The promotion rule, precisely, from the source material:**
1. Same fixed, versioned eval dataset for both versions — no reruns with different data.
2. n ≥ 200 examples (sized via power analysis for the effect size you care about detecting).
3. Log per-sample metrics: score, latency_ms, cost_usd, format_ok — not just aggregates, because you need per-example pairs for the paired test and you need the distribution, not just the mean, to catch regressions hiding in the tail (e.g., p95 latency).
4. Statistical significance: paired t-test, p < 0.05.
5. **Practical significance combined with statistical significance**: promote only if correctness does not decrease and no other tracked metric regresses by more than its own threshold (e.g., 1%) — even when p < 0.05.

**Why you need practical significance *on top of* statistical significance — this is the single most important nuance in this entire module.** With n = 200 (or larger), a t-test can return p < 0.05 for a correctness improvement of, say, 0.3% — a difference that is "real" in the sense of not being pure noise, but utterly irrelevant to users or the business, and possibly not worth the deployment risk, review overhead, or a confounding cost regression. Statistical significance answers "is this difference probably real?" Practical significance answers "do I care?" You need both answers before you spend engineering trust on a promotion. This is precisely why the transcript's A/B example gates on **both** `p < 0.05` **and** `lift > 0.03` (a minimum 3% practical improvement threshold) before promoting.

**When NOT to use a full paired-significance-test gate:**
- PATCH-level wording/typo fixes with no intended behavior change — a smoke test (does it still produce valid output on a handful of examples) is proportionate; running a 200-example statistical experiment for every comma fix is wasteful bureaucracy that will train engineers to route around your process.
- Extremely low-traffic or early-stage products where you cannot realistically collect 200 comparable eval examples — here, favor a smaller eval set with qualitative human review and accept weaker statistical guarantees, but be explicit that you are doing so (do not silently apply a t-test to n=15 and trust the p-value; small-sample paired t-tests are far more sensitive to violations of the normality assumption).
- Safety-critical MAJOR changes (e.g., a medical or legal-advice system prompt) where even a statistically and practically significant *average* improvement might mask a worse *tail* behavior — here, statistical promotion gates should be a necessary but not sufficient condition; add explicit tail/worst-case and adversarial-example review.

#### Architecture — Statistical Promotion Gate

```
        Frozen Eval Set (v7, N=200)
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
 Baseline v1.3.2            Candidate v1.4.0
 scores[200]                scores[200]
      │                           │
      └─────────────┬─────────────┘
                     ▼
        per-example paired differences
           d_i = candidate_i - baseline_i     (i = 1..200)
                     │
                     ▼
        ┌─────────────────────────────┐
        │  paired t-test on d_i         │
        │  scipy.stats.ttest_rel(...)    │
        │  → statistic, p-value          │
        └─────────────────────────────┘
                     │
        p-value < 0.05?  ──No──▶  NOT statistically significant
                     │                       │
                    Yes                       ▼
                     │              → hold at current version,
                     ▼                 log result, do not promote
        lift = mean(d_i) practically
        meaningful? (e.g. > 3%)
                     │
       ┌─────────────┼─────────────┐
      Yes                          No
       │                            │
       ▼                            ▼
  other metrics (latency,   Statistically real but
  cost, format) also within  too small to matter →
  non-regression thresholds? hold, log, do not promote
       │
  ┌────┴────┐
 Yes        No
  │          │
  ▼          ▼
AUTO-      BLOCK / manual review
PROMOTE    (secondary metric regressed)
```

#### Examples

**Beginner — the mechanics of a paired t-test, conceptually, with a tiny toy array**

```python
import numpy as np
from scipy import stats

# Toy example: 10 paired examples, scores in [0,1]
baseline_scores  = np.array([0.7, 0.8, 0.6, 0.9, 0.75, 0.65, 0.85, 0.7, 0.8, 0.9])
candidate_scores = np.array([0.75, 0.82, 0.65, 0.90, 0.80, 0.70, 0.88, 0.74, 0.83, 0.91])

t_stat, p_value = stats.ttest_rel(candidate_scores, baseline_scores)
print(f"mean lift: {(candidate_scores - baseline_scores).mean():+.3f}")
print(f"t-statistic: {t_stat:.3f}, p-value: {p_value:.4f}")
# With only 10 samples, even a consistent small lift often fails to reach
# p < 0.05 -- this is the intuition for why n matters (see section 3.5).
```

**Intermediate — reproducing the transcript's exact experiment (N=200, p=0.0023)**

```python
import numpy as np
from scipy import stats

def paired_significance_test(baseline_scores: np.ndarray,
                              candidate_scores: np.ndarray,
                              alpha: float = 0.05) -> dict:
    assert len(baseline_scores) == len(candidate_scores), "must be paired on the same eval set"
    diffs = candidate_scores - baseline_scores
    t_stat, p_value = stats.ttest_rel(candidate_scores, baseline_scores)
    return {
        "n": len(baseline_scores),
        "mean_lift": diffs.mean(),
        "std_diff": diffs.std(ddof=1),
        "t_statistic": t_stat,
        "p_value": p_value,
        "significant_at_0.05": p_value < alpha,
    }

# Simulated data consistent with the transcript's reported result
rng = np.random.default_rng(seed=7)
baseline_scores  = rng.normal(loc=0.880, scale=0.14, size=200).clip(0, 1)
candidate_scores = baseline_scores + rng.normal(loc=0.034, scale=0.10, size=200)
candidate_scores = candidate_scores.clip(0, 1)

result = paired_significance_test(baseline_scores, candidate_scores)
print(result)
# {'n': 200, 'mean_lift': ~0.03x, ..., 'p_value': ~0.00xx, 'significant_at_0.05': True}
```

**Production-grade — full A/B promotion decision combining statistical *and* practical significance**

```python
from dataclasses import dataclass
import numpy as np
from scipy import stats

@dataclass
class PromotionDecision:
    promote: bool
    reason: str
    lift: float
    p_value: float

def ab_promotion_check(
    variant_a_scores: np.ndarray,   # baseline
    variant_b_scores: np.ndarray,   # candidate
    min_practical_lift: float = 0.03,   # 3% minimum meaningful improvement
    alpha: float = 0.05,
    secondary_metrics_ok: bool = True,  # result of the delta_check.py gate (3.2)
) -> PromotionDecision:
    lift = variant_b_scores.mean() - variant_a_scores.mean()
    _, p_value = stats.ttest_rel(variant_b_scores, variant_a_scores)

    if p_value >= alpha:
        return PromotionDecision(False, "not statistically significant", lift, p_value)
    if lift <= min_practical_lift:
        return PromotionDecision(
            False,
            f"statistically significant (p={p_value:.4f}) but lift {lift:.1%} "
            f"below practical threshold {min_practical_lift:.0%}",
            lift, p_value,
        )
    if not secondary_metrics_ok:
        return PromotionDecision(False, "secondary metric (latency/cost/format) regressed", lift, p_value)

    return PromotionDecision(True, "significant and practically meaningful improvement", lift, p_value)
```

This function is the concrete implementation of the transcript's closing lesson: *"statistical testing is most useful when combined with a minimum practical improvement threshold."* Note that the function returns `False` — correctly — for a scientifically "real" but tiny lift (e.g., p=0.001, lift=0.8%). A junior engineer's version of this function often only checks `p_value < alpha`; the senior version always also checks the lift against a business-meaningful floor.

---

### 3.4 Power Analysis: Choosing Your Sample Size Before You Run the Experiment

#### Theory

Power analysis answers the question you should ask *before* running any comparison: **"how many eval examples do I need to reliably detect an improvement of the size I actually care about?"** Running an experiment with too few examples wastes engineering time on a test that had almost no chance of detecting a real effect even if one exists (an underpowered test) — and worse, if it happens to return p < 0.05 anyway on a small sample, that result is disproportionately likely to be an overestimate of the true effect (a well-documented statistical phenomenon sometimes called the "winner's curse" in small underpowered studies).

Four quantities are linked by power analysis, and fixing any three lets you solve for the fourth:

| Quantity | Symbol | Meaning |
|---|---|---|
| Effect size | *d* (Cohen's d for a paired test: mean difference ÷ std of differences) | The magnitude of improvement you want to be able to detect |
| Significance level | α | Your false-positive tolerance (probability of concluding "improved" when it isn't) — conventionally 0.05 |
| Power | 1 − β | Your true-positive detection rate (probability of concluding "improved" when it truly is) — conventionally 0.80 |
| Sample size | n | Number of paired eval examples |

The workflow in practice is almost always: "I know roughly how variable my per-example scores are (from historical data), I know the smallest improvement I actually care about detecting (e.g., 3% correctness), and I've fixed α=0.05 and power=0.80 by convention — solve for n."

```python
from statsmodels.stats.power import TTestPower

# Historical data tells you: individual example scores (LLM-judge, 0-1 scale)
# have roughly this standard deviation of paired differences
std_of_differences = 0.14
minimum_detectable_lift = 0.03    # the smallest improvement worth detecting
cohens_d = minimum_detectable_lift / std_of_differences

analysis = TTestPower()
required_n = analysis.solve_power(
    effect_size=cohens_d,
    alpha=0.05,
    power=0.80,
    alternative="larger",   # one-sided: we only care if candidate is BETTER
)
print(f"Cohen's d: {cohens_d:.3f}")
print(f"required sample size: {required_n:.0f}")
```

With `std_of_differences = 0.14` and `minimum_detectable_lift = 0.03`, Cohen's d ≈ 0.214 (a "small" effect by conventional benchmarks), and solving for n at α=0.05/power=0.80 lands in the same neighborhood as the transcript's stated n ≥ 200 — which is exactly why that number appears in the source material rather than a round 100 or 500: it is what the underlying variance and target effect size actually demand. **Do not treat n=200 as a universal constant** — it is the answer for *this* variance and *this* target lift. A noisier metric (higher variance) or a smaller improvement you want to be able to detect will require a larger n; a cleaner metric (e.g., a deterministic format-validity check instead of a noisy LLM-judge score) or a larger improvement you're targeting will need less.

**Practical guidance for choosing your inputs:**
- Estimate `std_of_differences` from historical A/B runs or from a pilot experiment — never guess it. If you have no historical data yet, run a modest pilot (e.g., n=50) purely to estimate variance, then power-analyze the *real* experiment from that.
- Choose `minimum_detectable_lift` based on what actually matters to the business/product, not what you hope to see. If a 1% correctness gain would not change any downstream decision, don't power your experiment to detect 1% — you'll pay for a much larger n than you need.
- Recompute power analysis whenever your eval metric changes (e.g., switching your correctness scorer from exact-match to an LLM judge changes the noise profile entirely).

#### Comparison Table — Choosing Your Statistical Test

| Scenario | Recommended test | Why |
|---|---|---|
| Same eval examples scored by both baseline and candidate (the standard case in this module) | Paired t-test (`scipy.stats.ttest_rel`) | Cancels shared per-example variance; matches this module's design |
| Different, non-overlapping eval example pools for baseline vs. candidate | Independent/unpaired t-test (`scipy.stats.ttest_ind`) or Mann-Whitney U | No natural pairing exists; expect to need a larger n for the same power |
| Score distribution is heavily skewed / has outliers (e.g., cost or latency, not a bounded [0,1] score) | Wilcoxon signed-rank test (paired, non-parametric) | Does not assume normally-distributed differences |
| Binary outcome (pass/fail per example) rather than a continuous score | McNemar's test (paired proportions) | Correct test family for paired binary data |
| Comparing more than two prompt variants simultaneously | Repeated-measures ANOVA, with post-hoc corrected pairwise tests | Controls family-wise error rate across multiple comparisons |
| Need continuously updated online monitoring (traffic-split, not offline eval set) | Sequential testing / always-valid p-values (as implemented in platforms like GrowthBook) | Classic fixed-n t-tests are invalid if you peek at results repeatedly before n is reached |

---

## 4. Real-World Case Studies (Reasoned Inference)

> These describe how organizations *would plausibly* architect such systems, based on publicly documented engineering-blog patterns and industry norms — not confirmed internal specifics.

**OpenAI / Anthropic (foundation-model providers running their own product surfaces, e.g., chat assistants and API-facing system prompts).** A provider operating a chat product at massive scale would almost certainly maintain an internal prompt/system-message registry analogous to what's described in section 3.1, versioned independently of model checkpoint releases, since system-prompt changes ship far more frequently than new model weights. Given the scale of daily active traffic, they would very plausibly run large-scale, statistically rigorous online A/B tests (not just offline eval-set comparisons) for user-facing system-prompt or tool-use-instruction changes, likely with sequential/always-valid testing methodology (as adopted broadly across the industry, e.g., in platforms like GrowthBook) rather than fixed-n batch t-tests, precisely because traffic volume makes continuous online monitoring both feasible and preferable to a one-shot offline evaluation. Given both companies publish extensively on evaluation methodology and safety review, a plausible architecture would combine an automated multi-metric gate (correctness/helpfulness, refusal-rate, safety-classifier scores, latency, cost) with mandatory human review for any change touching safety-relevant instructions — mirroring this module's "auto-block hard metrics, route soft/qualitative metrics to human review" pattern, but with safety review elevated to non-negotiable for a wider class of changes than a typical commercial product would require.

**Netflix (large-scale, self-service experimentation culture).** Netflix's publicly documented experimentation platform ("It's All A/Bout Testing," see references.md) is built around the principle that *any* team should be able to self-serve an experiment against a shared, centrally-computed metrics pipeline, with company-wide statistical rigor standards enforced by the platform rather than left to each team to reinvent. Applied to prompts in an LLM-touching product surface (e.g., a conversational recommendation or support-summarization feature), a Netflix-like architecture would plausibly treat a new prompt version exactly like any other experiment variant in their central platform: define the eval/success metrics once, register the candidate as a new experiment arm, and let the shared statistical engine (handling sample-size/power calculations and significance testing consistently) make the call — rather than a bespoke one-off script per team, which is the trap this module's `delta_check.py` pattern is a lightweight, single-team analog of.

**Databricks (an MLOps/data platform vendor, also an evaluation-tooling provider).** Databricks ships the MLflow GenAI Prompt Registry with Unity Catalog governance layered on top (access control over who may register/promote prompt versions, plus an audit trail) — a plausible reflection of enterprise customer demand for the same governance guarantees they already expect for trained-model artifacts (Module 02) applied to prompts. A Databricks-style enterprise deployment would likely tie prompt-version promotion approvals to the same RBAC/audit system used for model-stage transitions, so that "who approved promoting this prompt to production" is answerable from the same governance surface as "who approved this model for production."

**Uber / Spotify / Airbnb (product companies with mature internal experimentation platforms, generalized to LLM features).** These companies have historically built (and published about) sophisticated internal experimentation platforms for classic product A/B testing; extending such a platform to cover LLM prompt variants would plausibly mean *reusing* the existing metrics-pipeline and statistical-engine investment rather than building a parallel bespoke system — treating a prompt version as just another "variant" type in an existing framework, with LLM-specific metrics (correctness via an LLM judge, format validity, cost per call) added as new metric definitions inside that framework. This is a strong argument, in general, for *not* building this module's tooling as a silo: if your organization already has a mature experimentation platform, integrate prompt evaluation as a new metric/variant type there instead of building a second, parallel statistical testing system.

**Amazon / Meta (very high request volume, cost-sensitive at scale).** At extreme request volumes, even a "small" percentage cost regression per call translates into a large absolute dollar impact, so a plausible architecture would weight the cost-delta gate more aggressively (tighter regression thresholds) than a lower-traffic product would need to, and would very likely require the practical-significance lift threshold to clear a higher bar specifically to justify the operational and cost risk of any change — consistent with this module's point that statistical significance alone is an insufficient promotion criterion at scale.

---

## 5. Common Mistakes

1. **Editing a "live" prompt file in place instead of creating a new version.** Destroys reproducibility and makes every past log entry that references a version number silently wrong.
2. **Comparing candidate and baseline on different eval examples (or a reshuffled/resampled eval set).** Any observed delta is now confounded with which examples happened to be easier or harder — this single mistake invalidates the entire comparison, no matter how sophisticated the statistics afterward.
3. **Using an unpaired test on paired data (or vice versa).** Wastes statistical power at best, produces an outright invalid p-value at worst.
4. **Trusting p < 0.05 alone as the promotion criterion, with no practical-significance floor.** With a large enough n, trivially small and operationally meaningless improvements will reach statistical significance.
5. **Optimizing (and gating) on a single metric — usually correctness — while ignoring latency, cost, and format/behavioral stability.** Ships a prompt that is "better" on paper but worse in production economics or user experience.
6. **Choosing sample size arbitrarily** ("let's just use 50 examples, that feels like enough") instead of via power analysis tied to the effect size that actually matters.
7. **Fully automating decisions on qualitative/subjective metrics** (tone, brand voice, style) with no human-review escape hatch — some deltas genuinely require human judgment, and an all-automated gate either false-blocks good changes or lets bad ones through silently.
8. **Ignoring the alias-cache-propagation gap during rollback.** Believing a rollback is complete the instant the alias is repointed, when running instances may still be serving a cached prior resolution for several minutes.
9. **Not versioning the eval dataset itself.** If the eval set can silently change (new examples added, old ones edited) between two comparisons, you've broken the exact same fixed-comparison guarantee that made the statistical test valid in the first place.
10. **Re-running/peeking at results repeatedly before your pre-computed n is reached, using a fixed-n t-test.** Classic "p-hacking via optional stopping" — if you need to monitor results continuously as traffic accrues, use a sequential testing method designed for repeated looks, not a one-shot t-test evaluated early.
11. **Conflating a MAJOR breaking change (e.g., new output schema) with a MINOR or PATCH bump**, breaking downstream consumers who assumed backward compatibility within a minor version.
12. **Treating the "correctness" score as ground truth when it's actually an LLM-judge output** with its own known biases (position bias, verbosity bias, self-enhancement bias — see Zheng et al. in references.md) — a statistically significant lift in an LLM-judge score is only as trustworthy as the judge itself.

---

## 6. Best Practices and Production Tips

**When to use the full statistical-promotion machinery:**
- MINOR/MAJOR prompt version changes headed for production, on any system with meaningful traffic/business impact.
- Any change to a metric-sensitive dimension: correctness-affecting wording, few-shot examples, temperature/sampling parameters, or model swaps.

**When NOT to (lighter-weight alternatives):**
- PATCH-level cosmetic fixes → smoke test only.
- Early-prototype/pre-launch products with no meaningful traffic yet → qualitative human review plus a small fixed eval set; be explicit that statistical rigor is being deferred, and revisit once traffic/data volume supports it.
- If your organization already runs a mature, general-purpose experimentation platform (Netflix/Uber/Spotify-style) → integrate as a new metric/variant type there instead of building a parallel bespoke statistical pipeline.

**Cost and scaling:**
- Every candidate evaluation run costs real money (n inference calls × candidate, plus n calls × baseline for re-verification if you don't cache baseline scores — cache them). At n=200 and typical foundation-model pricing, a single comparison is inexpensive; the cost adds up when many teams each run frequent comparisons — this is the argument for a shared evaluation service rather than per-team ad hoc scripts.
- If correctness is scored by an LLM judge, the judge call itself is an additional cost and latency line item per eval example — budget for 2× (or more, with e.g. self-consistency/multiple-judge-sample strategies) the raw candidate-inference cost.
- Cache baseline scores per eval-dataset-version + prompt-version pair; never re-run the (unchanged) baseline from scratch for every new candidate comparison.

**Monitoring:**
- Emit every delta report and every statistical-test result to your observability stack (a natural next step into Module 10+ monitoring) — dashboards should show "which prompt version is live in production right now," "when did it last change," and "what was the promotion evidence."
- Alert on divergence between offline eval-set results and observed online production metrics after promotion — a candidate that won offline can still underperform online due to eval-set/production distribution shift; this is why many mature setups follow an offline statistical gate with a small, monitored canary/shadow-traffic rollout before full promotion (directly analogous to the canary deployment patterns from Module 05, just applied to prompt versions instead of container images).

**Security and governance:**
- Restrict who can register and, especially, who can *promote* prompt versions to production — a malicious or careless prompt change is a real attack surface (prompt injection via a compromised registry, or a well-meaning but unreviewed change that leaks system instructions). Apply the same RBAC/audit discipline used for model-registry promotions (Module 02).
- Treat prompt content itself as potentially sensitive — system prompts can encode business logic, guardrail instructions, or proprietary strategy; the registry's access controls and audit trail should reflect that.

**Performance tradeoffs:**
- Resolving a prompt alias on every request (read-through, no cache) guarantees instant rollback propagation but adds a registry-lookup round trip to every request's latency budget; a short-TTL cache trades a small, bounded rollback-propagation delay for materially better steady-state latency — choose deliberately and document the tradeoff (this is precisely the nuance LangSmith's Prompt Hub documentation calls out).

---

## 7. Interview Questions

**Q1: Why should prompts be versioned and treated as immutable artifacts rather than editable config?**
*Model answer:* Because prompts directly determine production model behavior, the same way hyperparameters or model weights do. Immutability guarantees that a version number is a permanent, trustworthy reference — any log, trace, or eval report citing "v1.4.0" remains meaningful forever, which is essential for reproducibility and incident debugging. Mutable "current prompt" files are the prompt-engineering equivalent of the `latest` Docker tag antipattern.

**Q2: Why must you use a paired test (not an unpaired test) when comparing a baseline and candidate prompt on the same eval set?**
*Model answer:* Because each eval example is scored by both baseline and candidate, the two score arrays are not independent — some examples are inherently harder for both prompts. A paired test (e.g., paired t-test on per-example differences) removes that shared per-example variance from the noise estimate, isolating the variance attributable specifically to the prompt difference. Using an unpaired test throws away that cancellation, understating your statistical power (or, in other data configurations, violating test assumptions outright) and making real improvements harder to detect.

**Q3: A candidate prompt shows p = 0.001 and a 0.4% correctness lift. Should you promote it? Why or why not?**
*Model answer:* Not automatically. p = 0.001 says the lift is very unlikely to be pure sampling noise — it's statistically significant. But 0.4% may fall below any meaningful practical-significance threshold (e.g., a 3% minimum lift), meaning the improvement, while "real," is too small to justify the review overhead, deployment risk, or any confounding regression in cost/latency. Promotion should require both statistical *and* practical significance.

**Q4: How do you decide how many eval examples (n) you need before running a prompt comparison?**
*Model answer:* Via a power analysis: given your target minimum detectable effect size (e.g., 3% quality lift), your estimated variance in per-example score differences (from historical data or a small pilot), and conventional α=0.05/power=0.80, solve for n using a tool like `statsmodels.stats.power.TTestPower.solve_power`. n should scale with variance and inversely with the target effect size — larger n is needed to reliably detect smaller or noisier improvements. Do not treat any specific n (e.g., 200) as a universal constant; it's derived from your specific metric's noise and your target lift.

**Q5: Why should prompt evaluation be multi-metric rather than gating only on correctness?**
*Model answer:* Because a prompt change's effects on latency, cost, and output format/behavior are largely independent of its effect on correctness — a prompt engineered to be more accurate (e.g., via chain-of-thought instructions) very plausibly becomes slower and more expensive, and a prompt that tightens output formatting can shift tone in ways a correctness metric can't detect. Gating on correctness alone is structurally blind to regressions in every other dimension that matters operationally and to users.

**Q6: What's the operational risk in an alias-based rollback design, and how do you mitigate it?**
*Model answer:* Repointing an alias (e.g., `production`) is instantaneous at the registry level, but already-running application instances may hold a cached resolution of the old version for up to their cache TTL (commonly a few minutes in systems like LangSmith's Prompt Hub). During an incident, teams can mistakenly believe a rollback is fully propagated when it isn't. Mitigations: keep TTLs short and known, support forced cache invalidation/read-through on rollback, and make rollback propagation status observable (e.g., via per-instance version metrics) rather than assumed.

**Q7: Why route some prompt-delta metrics (e.g., tone shift) to human review instead of fully automating the gate?**
*Model answer:* Because some dimensions — tone, brand voice, style consistency — are inherently qualitative and context-dependent; a numeric "tone-shift score" crossing a threshold is a useful *signal* that something changed, but whether that change is acceptable often requires human judgment that a scalar threshold can't safely encode. Fully automating decisions on subjective metrics either blocks good changes unnecessarily or silently ships bad ones that happen to satisfy the metric's blind spots.

**Q8: How would you scale this module's single-team `delta_check.py` pattern to an organization with dozens of teams each shipping prompt changes?**
*Model answer:* Rather than each team maintaining its own bespoke gate script and eval set, invest in a shared evaluation/registry service (analogous to a company-wide experimentation platform, as Netflix and similar companies have built for classic product A/B testing): centralized eval-dataset versioning, a shared statistical engine implementing power analysis and paired significance testing consistently, and a shared registry with uniform RBAC/audit — with teams supplying only their task-specific eval examples and metric definitions. This avoids each team reinventing (and subtly misimplementing) the statistics, and gives the organization a single, auditable source of truth for "what prompt is live where, and what evidence justified it."

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

A production prompt is a versioned, immutable configuration artifact — not a string literal. Every meaningful change should produce a new semantic version with full metadata (system prompt text, temperature, model, eval-dataset version, author, commit message), stored in a registry where promotion means repointing a mutable alias at an immutable version, never editing history. A version number alone is not evidence of improvement: only a **delta report** — measured on the *same, frozen, versioned* eval set, across *multiple metrics* (correctness, latency, cost, format/behavior) — tells you what actually changed. And an observed delta is only trustworthy as a promotion signal once it clears **both** a statistical-significance bar (a properly paired test, sized via power analysis to the effect you care about) **and** a practical-significance floor (the improvement is large enough to matter operationally). Automate what can be automated (hard-metric regressions, statistical gating); route what genuinely needs judgment (qualitative/tone shifts, safety-relevant changes) to human review.

### Key Takeaways

- Immutable versions + mutable aliases = safe, instant, auditable promotion and rollback.
- No comparison is trustworthy unless baseline and candidate are evaluated on the identical, frozen eval set.
- Prompt evaluation must be multi-metric: correctness alone hides cost/latency/format regressions.
- Use a **paired** significance test (per-example differences), not an unpaired one.
- Statistical significance (p < 0.05) is necessary but not sufficient — always pair it with a practical-significance/minimum-lift threshold.
- Choose sample size via power analysis tied to your actual target effect size and measured variance — don't guess n.
- Automate hard-metric gates; route qualitative/subjective deltas to human review.
- Know your rollback's cache-propagation delay before an incident forces you to find out.

### Production Checklist

- [ ] Every prompt change creates a new immutable version with a semantic version bump appropriate to its risk (PATCH/MINOR/MAJOR).
- [ ] Version metadata captured: system prompt, temperature, model, eval_dataset_version, author, commit message, timestamp.
- [ ] Promotion is alias-repoint only; no in-place edits to published versions, ever.
- [ ] Rollback path tested and its cache-propagation delay is known and monitored.
- [ ] Eval dataset is frozen, versioned, and never silently changes between a baseline and candidate comparison.
- [ ] Delta report computed across correctness, latency, cost, and format/behavior — not correctness alone.
- [ ] Sample size (n) for any statistical comparison is derived via power analysis, not guessed.
- [ ] Paired significance test used (not unpaired) when baseline/candidate share the same eval examples.
- [ ] Promotion rule requires both statistical significance (p < 0.05) AND a practical-significance/minimum-lift threshold.
- [ ] No other tracked metric is allowed to regress beyond its own threshold, even when the primary metric improves.
- [ ] Hard-metric regressions auto-block; qualitative/subjective-metric drift routes to human review, not silent auto-decision.
- [ ] All deltas, test statistics, p-values, and promotion decisions are logged for audit.
- [ ] RBAC/access control restricts who can register and, especially, who can promote prompt versions to production.
- [ ] Post-promotion online monitoring compares production metrics against offline eval-set expectations to catch distribution shift.

---

## 9. Further Reading

Full citations, official docs, papers, GitHub repos, and video references for this module are maintained separately (and kept current independent of this file) in this same folder:

- `references.md` — official documentation, academic papers, and engineering blog posts.
- `videos.md` — supplementary video resources.
- `books.md` — book chapters and further textbook treatments.
- `github.md` — reference repositories (MLflow, promptfoo, deepeval, openai/evals, GrowthBook, scipy/statsmodels).

See `architecture.md` in this folder for expanded architecture diagrams, a full sequence diagram of the promotion flow, and a decision tree for choosing between the tools and statistical approaches covered above.
