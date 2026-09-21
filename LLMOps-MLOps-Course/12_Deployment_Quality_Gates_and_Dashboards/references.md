# References — Deployment Quality Gates and Release Dashboards

Official documentation, papers, and engineering blog posts, verified via direct fetch
where noted. Grouped by theme to match the module's three-part structure plus the two
expansion topics (GitHub Actions integration, Grafana dashboard design).

---

## 1. Evaluation triggers and CI/CD integration

**GitHub Actions — Events that trigger workflows**
https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
What it teaches: Official syntax and semantics for `pull_request`, `push`, `schedule`
(cron, 5-minute minimum interval, runs only on the default branch), `workflow_dispatch`
(manual trigger with typed inputs), and `repository_dispatch` (externally triggered,
e.g. from a data-pipeline completion webhook). This is the primary reference for
implementing this module's four trigger points (PR, push, schedule, data-update) as
literal workflow YAML.
Difficulty: Beginner. Reading time: 20–30 minutes.

**Promptfoo — CI/CD Integration for LLM Eval and Security**
https://www.promptfoo.dev/docs/integrations/ci-cd/
What it teaches: How to wire promptfoo evaluation runs into GitHub Actions, GitLab CI,
and other CI systems, including PR-comment reporting and threshold-based failure.
Difficulty: Intermediate. Reading time: 20 minutes.

**Promptfoo GitHub Action reference (action.yml)**
https://github.com/promptfoo/promptfoo-action/blob/main/action.yml
What it teaches: The literal input/output contract of the action — `fail-on-threshold`,
`repeat`, `repeat-min-pass`, cache configuration — useful as a copy-paste starting point
for the module's GitHub Actions release-gate example.
Difficulty: Intermediate. Reading time: 15 minutes.

**DeepEval — Unit Testing in CI/CD**
https://deepeval.com/docs/evaluation-unit-testing-in-ci-cd
What it teaches: The `deepeval test run` workflow — loading an `EvaluationDataset`,
constructing `LLMTestCase`/`ConversationalTestCase` objects, running pytest-style
assertions with `assert_test()`, and a complete GitHub Actions YAML example. Explicitly
confirms the same command works unmodified in GitLab CI, CircleCI, and Jenkins.
Difficulty: Intermediate. Reading time: 25–30 minutes.

**Evidently AI — CI/CD for LLM apps: Run tests with Evidently and GitHub Actions**
https://www.evidentlyai.com/blog/llm-unit-testing-ci-cd-github-actions
What it teaches: A five-step pattern (load test data → run inference → evaluate with
LLM judges/Python functions/metrics → generate a pass/fail Test Suite report → gate the
CI workflow), plus optional cloud dashboarding for tracking regressions and trends over
time. This is close to a direct reference implementation of the module's three-tier
evaluation strategy.
Difficulty: Intermediate. Reading time: 20 minutes.

**Arize — How to Add LLM Evaluations to CI/CD Pipelines**
https://arize.com/blog/how-to-add-llm-evaluations-to-ci-cd-pipelines/
What it teaches: Using the Phoenix experiments API to define a dataset, task, and
evaluator, then gating a GitHub Actions job on
`experiment.get_evaluations()["score"].mean() > 0.8` — a concrete, literal release-gate
code pattern very close to what this module's "release-gate code example" should look
like.
Difficulty: Intermediate. Reading time: 15–20 minutes.

**Langfuse — Prompt CI/CD: version, gate, and roll out prompts like code**
https://langfuse.com/resources/engineering/prompt-cicd
What it teaches: Treating prompts as versioned artifacts with their own gated
release/rollout pipeline, distinct from application code deploys — relevant background
for why LLM systems need release gates in more places than a traditional service does
(prompt changes, model-version changes, and retrieval-config changes are all separate
deploy surfaces).
Difficulty: Intermediate. Reading time: 15 minutes.

---

## 2. Quality gates, thresholds, and release-readiness research

**"Automated Self-Testing as a Quality Gate: Evidence-Driven Release Management for
LLM Applications"** — Alexandre Cristovão Maiorano (arXiv, 2026)
https://arxiv.org/html/2603.15676v1
What it teaches: A five-dimension quality gate (task success rate, context
preservation, P95 latency, safety pass rate, evidence coverage) producing a
deterministic PROMOTE / HOLD / ROLLBACK decision, validated across 38 evaluation runs
over 20+ releases of a production multi-agent conversational system. Directly relevant
to this module's "hard blocks" and "release-gate code example" sections — it's one of
the few papers that treats the release decision itself, not just the metric, as the
object of study.
Difficulty: Advanced. Reading time: 45–60 minutes.

**"LLM Readiness Harness: Evaluation, Observability, and CI Gates for LLM/RAG
Applications"** — Alexandre Cristovão Maiorano, Lumytics (arXiv, 2026)
https://arxiv.org/html/2603.27355
What it teaches: Combines automated benchmarking (including BEIR retrieval
benchmarks), OpenTelemetry-based observability, and promptfoo-driven CI quality gates
into "scenario-weighted readiness scores" aggregating workflow success, policy
compliance, groundedness, retrieval hit rate, cost, and P95 latency. Notably
demonstrates that a smaller model can beat a larger one on readiness/faithfulness for
a specific domain (FiQA finance questions) — a good concrete example for teaching why
relative, per-dimension gates matter more than a single aggregate score.
Difficulty: Advanced. Reading time: 45–60 minutes.

---

## 3. Reading evaluation dashboards / LLM observability

**Grafana Labs — "How to monitor LLMs in production with Grafana Cloud, OpenLIT, and
OpenTelemetry"**
https://grafana.com/blog/ai-observability-llms-in-production/
What it teaches: The architecture and five prebuilt dashboards Grafana Cloud ships for
GenAI observability: GenAI Observability (request rate, latency percentiles, cost,
time-to-first-token), GenAI Evaluations (hallucination/bias/toxicity event summaries),
Vector Database Observability, MCP Observability, and GPU Monitoring. Also documents
the specific OpenTelemetry metric names used (e.g. `gen_ai_usage_cost_USD_sum`,
`gen_ai_usage_input_tokens_total`) and confirms that Grafana Alerting can fire when
"evaluation scores cross your quality gates" — directly relevant to this module's
Grafana dashboard-design expansion.
Difficulty: Intermediate. Reading time: 20–25 minutes.

**Grafana Labs — "A complete guide to LLM observability with OpenTelemetry and Grafana
Cloud"**
https://grafana.com/blog/a-complete-guide-to-llm-observability-with-opentelemetry-and-grafana-cloud/
What it teaches: A deeper walkthrough of the OTel-based instrumentation pipeline
feeding Grafana's GenAI dashboards, useful alongside the post above for the panel/alert
design section.
Difficulty: Intermediate. Reading time: 20 minutes.

**Grafana documentation — Alerting**
https://grafana.com/docs/grafana/latest/alerting/
What it teaches: Core concepts for alert rules, evaluation intervals, thresholds, and
notification policies — the primitives this module's Grafana section uses to define
release-readiness alerts (e.g., "page release manager if hallucination rate exceeds
threshold for two consecutive evaluation windows").
Difficulty: Beginner to intermediate. Reading time: 20–30 minutes.

**OpenTelemetry — Gen AI semantic conventions (attribute registry)**
https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/
What it teaches: The standardized `gen_ai.*` attributes (model name, token counts,
operation name, etc.) that vendor-neutral dashboards should be built against. As of
mid-2026 these conventions remain marked experimental, and there is no stabilized
attribute set yet for embedding evaluation/quality scores directly into spans — worth
flagging explicitly to students as a "the standard hasn't caught up to the practice
yet" gap.
Difficulty: Intermediate. Reading time: 20 minutes (registry is a reference, not a
narrative read).

**OpenTelemetry — Metrics semantic conventions (general)**
https://opentelemetry.io/docs/specs/semconv/general/metrics/
What it teaches: General rules for naming and structuring metrics that the GenAI
conventions build on top of — useful grounding if students are defining their own
custom evaluation-score metrics for a dashboard.
Difficulty: Intermediate. Reading time: 15 minutes.

**Google Cloud — Vertex AI Gen AI evaluation service**
https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/evaluate-models
What it teaches: Google's managed evaluation offering — rubric-based metrics, dataset
preparation, and the AutoSxS (side-by-side) comparison pipeline. Note: as of this
writing the official docs do not give explicit CI/CD or release-gating guidance, so use
this primarily as a reference for available managed metrics rather than a gating
recipe — pair it with the promptfoo/DeepEval/Evidently CI docs above for the actual
gating mechanics.
Difficulty: Intermediate. Reading time: 20 minutes.

---

## 4. Broader release-engineering grounding (non-LLM-specific but load-bearing)

**Google SRE Book — free online**
https://sre.google/sre-book/monitoring-distributed-systems/
What it teaches: The four golden signals (latency, traffic, errors, saturation) and
symptom-vs-cause monitoring philosophy that this module's green/yellow/red readiness
framing is descended from.
Difficulty: Intermediate. Reading time: 30–40 minutes.

---

## Currency notes for mid-2026

- Promptfoo is now part of OpenAI while remaining MIT-licensed and open source — worth
  mentioning as a "who owns the eval tooling landscape" fact when teaching this module,
  since it changes the vendor-neutrality calculus for teams choosing a CI eval tool.
- OpenTelemetry's GenAI semantic conventions are still experimental and explicitly do
  not yet standardize how evaluation/quality scores are attached to spans — teams
  building the Grafana dashboard in this module are currently using custom attributes
  for that, not a blessed standard. Expect this to stabilize sometime after mid-2026;
  check the OpenTelemetry GenAI SIG's status page before teaching this as settled.
- Vendor dashboards (Grafana's GenAI Observability/Evaluations pair, Arize Phoenix,
  Evidently Cloud, Confident AI) have converged on essentially the same five-signal
  shape this module teaches: request volume/latency/cost, and a separate
  hallucination/bias/toxicity/safety evaluation summary — reinforcing that this
  module's dashboard design is representative of current industry practice, not a
  bespoke invention.
