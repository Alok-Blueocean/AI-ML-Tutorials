# GitHub Repositories — Deployment Quality Gates and Release Dashboards

All entries below were verified to exist via direct repository fetch. Star counts are
approximate snapshots taken mid-2026 and will drift — treat the "popularity tier" label
as the durable signal, not the exact number.

---

### 1. promptfoo/promptfoo
**URL:** https://github.com/promptfoo/promptfoo
**Popularity tier:** Very popular / widely adopted (~23.8k stars, ~2.1k forks at time
of writing — one of the most-starred LLM eval tools on GitHub).
**Purpose:** CLI and library for evaluating, red-teaming, and regression-testing LLM
prompts, agents, and RAG pipelines. Supports declarative YAML test configs, comparison
across providers (GPT, Claude, Gemini, Llama, DeepSeek, etc.), and a red-team mode for
vulnerability/prompt-injection scanning.
**Notable:** As of mid-2026 promptfoo is part of OpenAI while remaining MIT-licensed
and open source — a useful current fact for interview-level "who owns what" awareness.
**How it relates to this module:** Its native GitHub Actions integration
(`promptfoo/promptfoo-action`, below) is the most directly reusable template for the
module's "gate a deployment on evaluation results" GitHub Actions example — it already
implements PR-comment reporting, push-triggered summaries, and a `fail-on-threshold`
gate parameter that maps almost one-to-one onto the module's release-threshold logic.

---

### 2. promptfoo/promptfoo-action
**URL:** https://github.com/promptfoo/promptfoo-action
**Popularity tier:** Niche / emerging (modest star count, but it is the official,
actively maintained action for the very popular promptfoo project).
**Purpose:** The official GitHub Action wrapper for promptfoo. Supports `pull_request`
(posts eval results as a PR comment), `push` (writes to the workflow summary), and
`workflow_dispatch` (manual/on-demand runs) — i.e., three of this module's four
trigger points natively, out of the box.
**How it relates to this module:** Directly demonstrates `fail-on-threshold` (a
required pass-percentage gate) and `repeat`/`repeat-min-pass` settings for handling
flaky, non-deterministic LLM outputs — exactly the "how do you gate on something
non-deterministic" problem this module's release-gate code example has to solve.

---

### 3. confident-ai/deepeval
**URL:** https://github.com/confident-ai/deepeval
**Popularity tier:** Very popular / widely adopted.
**Purpose:** "The LLM evaluation framework" — a pytest-flavored library
(`deepeval test run` replaces `pytest`) with built-in metrics (answer relevancy,
faithfulness/hallucination, contextual precision/recall, bias, toxicity, and more) and
first-class CI/CD documentation showing GitHub Actions, GitLab CI, CircleCI, and
Jenkins integration.
**How it relates to this module:** Its CI/CD docs are close to a reference
implementation of the module's "PR trigger runs tier-2 evaluation and blocks the merge
on regression" pattern, including optional integration with Confident AI's cloud
dashboard for tracking pass-rate trends over time — directly relevant to the
30-day-trend dashboard section.

---

### 4. explodinggradients/ragas
**URL:** https://github.com/explodinggradients/ragas
**Popularity tier:** Very popular / widely adopted (~15k stars).
**Purpose:** Evaluation toolkit specifically for RAG and LLM applications — faithfulness,
answer relevancy, context precision/recall, plus synthetic test-set generation
("intelligent test generation") so teams don't have to hand-label a full golden
dataset from scratch.
**How it relates to this module:** Useful for the "data-update trigger" discussion —
Ragas's test-generation tooling is a practical answer to "how do you keep your golden
dataset current as production data drifts," which is exactly the fourth trigger point
(data-update) this module covers.

---

### 5. evidentlyai/evidently
**URL:** https://github.com/evidentlyai/evidently
**Popularity tier:** Very popular / widely adopted (one of the most established
ML/LLM monitoring and drift-detection libraries, predating the current LLM-eval wave).
**Purpose:** Open-source library for evaluating, testing, and monitoring ML and LLM
systems — data drift, model quality, and a "Test Suite" abstraction (pass/fail checks
with explicit thresholds) that generates HTML/JSON reports and can gate CI builds.
**How it relates to this module:** Its documented GitHub Actions pattern (load test
data → run inference → evaluate with Test Suite → fail the workflow on any failing
check) is essentially the module's three-tier evaluation strategy expressed as a
concrete, runnable pipeline. Its optional cloud dashboards are a second reference point
(alongside Grafana) for the "reading evaluation dashboards" section.

---

### 6. Arize-ai/phoenix
**URL:** https://github.com/Arize-ai/phoenix
**Popularity tier:** Very popular / widely adopted.
**Purpose:** Open-source LLM observability and evaluation library (OpenTelemetry-based
tracing plus an "Experiments" API for running evaluators against datasets and tracking
scores over time).
**How it relates to this module:** Arize's public engineering content demonstrates
wiring an experiment's mean score directly into a CI gate condition
(`experiment.get_evaluations()["score"].mean() > 0.8`), which is close to a literal
implementation of this module's release-gate code example, built on OpenTelemetry
traces rather than a bespoke logging format — useful for connecting this module back
to modules on observability instrumentation.

---

### 7. open-telemetry/semantic-conventions
**URL:** https://github.com/open-telemetry/semantic-conventions
**Popularity tier:** Very popular / widely adopted (official OpenTelemetry project).
**Purpose:** Source of truth for the GenAI semantic conventions (`gen_ai.*` span and
metric attributes: token counts, model name, latency, cost) that vendor dashboards
(Grafana, Datadog, and others) build their GenAI panels on top of.
**How it relates to this module:** This is the underlying data contract that makes a
vendor-neutral Grafana dashboard for LLM release readiness possible — panels for
latency, cost, and token usage in this module's Grafana design section should be built
against these attribute names so the dashboard isn't locked to one vendor's proprietary
schema. Note: as of early-to-mid 2026 the GenAI semantic conventions are still marked
experimental, and there is no stable convention yet for embedding evaluation/quality
scores directly into spans — teams currently attach those as custom attributes.

---

### 8. chiphuyen/dmls-book
**URL:** https://github.com/chiphuyen/dmls-book
**Popularity tier:** Very popular / widely adopted (~5.1k stars).
**Purpose:** Official companion repository for *Designing Machine Learning Systems*
— chapter summaries, curated MLOps tooling lists, and supplementary learning
resources (no runnable code, since the book itself is design-principles-focused).
**How it relates to this module:** A free, fast way to skim the monitoring and
deployment chapters' key ideas (distribution shift, evaluation slicing, canary/shadow
deployment) that underpin this module's dashboard and gating philosophy, without
requiring the book itself.

---

## How to use this list

For hands-on lab exercises in this module, `promptfoo` + `promptfoo-action` or
`deepeval` are the fastest path to a working, gated GitHub Actions pipeline — both
have first-party actions/CLI commands and documented CI examples. For the
observability/dashboard half of the module, pair either of those with `Arize-ai/phoenix`
or Grafana's OpenTelemetry-based GenAI dashboards (see references.md) so that trace
data and evaluation scores live in the same OpenTelemetry-based data model.
