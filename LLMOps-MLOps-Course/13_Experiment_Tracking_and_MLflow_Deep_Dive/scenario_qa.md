# Experiment Tracking and MLflow Deep Dive — Scenario-Based Q&A

**Situation:** Six weeks into a sprint, someone asks "which exact combination of model, prompt version, and dataset produced the 92% eval score we showed leadership?" and nobody can answer with confidence — the number lives in a screenshot pasted into a shared doc. What would you do and why?

Model answer: Name this as "spreadsheet hell" — the predictable failure mode of tracking experiments by hand instead of through a system of record — and fix the process, not just this one lookup. Stand up MLflow Tracking so every run automatically logs its parameters (model id, prompt version, dataset version, temperature), its metrics, and its artifacts together as one queryable unit, tagged consistently enough to search. For the immediate question, be honest that a screenshot-sourced number can't be reliably reconstructed, and treat that as the concrete cost example that justifies the tracking investment going forward.

---

**Situation:** A new engineer's first PR logs full raw user prompts and full raw model responses to the tracking store "for debuggability," including whatever a user happened to type. What's wrong with this, and what do you have them do instead?

Model answer: Flag this immediately as a privacy-safe telemetry violation — raw prompt and response text can contain PII (names, emails, account numbers, health information) that a user never intended to have persisted in an experiment-tracking system with broad internal access. Have them replace raw content logging with metadata-only logging by default: hashed identifiers, model and prompt version, token counts, latency, and score, with raw content capture only enabled behind an explicit, policy-approved flag for a narrow debugging window. Debuggability doesn't require raw content by default — it requires enough structured metadata (which prompt version, which model, which dataset slice) to reproduce and investigate without needing the actual user text logged everywhere.

---

**Situation:** Your team is still using MLflow's `Staging` → `Production` → `Archived` stage labels to manage model promotion, and a rollback last month required manually re-tagging three separate model versions in the wrong order, causing a brief outage. What would you recommend, and why?

Model answer: Recommend migrating off the deprecated stage-transition model to the alias-based pattern — mutable named pointers like `champion`, `shadow`, and `canary` that point at one specific model version each. The outage happened because stage transitions are a multi-step, order-sensitive operation across versions; aliases fix this by making promotion and rollback a single atomic reassignment (point the `champion` alias at a different version) instead of a sequence of stage moves that can be done out of order under pressure. Production code should always resolve the model to use via the alias, never a hardcoded version number, so rollback is exactly as simple as reassigning the alias back to the last known-good version.

---

**Situation:** A data scientist proudly reports that a new prompt variant improved the eval quality score by 3%, and wants to ship it immediately. You check the experiment tracking dashboard and see the new variant's P95 latency is 40% higher and cost per query nearly doubled. How do you respond?

Model answer: Reframe the decision around the Pareto frontier instead of a single metric win. A run that improves quality while regressing latency and cost isn't strictly better — it's a different point on a multi-metric tradeoff surface, and whether it's worth shipping depends on whether the product can absorb the latency/cost hit for that quality gain. Pull up the full set of tracked runs and check whether any other candidate sits on the Pareto frontier with a smaller cost/latency penalty for a similar quality gain; if this run is genuinely the best available tradeoff, make the decision explicitly with whoever owns the latency SLA and cost budget, rather than shipping on the quality number alone.

---

**Situation:** Your team runs 20 experiments in a single sprint — two models, five prompt variants, two temperatures — and a week later someone wants to know why one specific run scored so much lower than the others. The run's parameters were logged, but nobody tagged which evaluator version scored it, and the evaluator has been updated twice since. What's the gap, and how do you close it?

Model answer: The gap is an incomplete tagging strategy — logging model, prompt, and dataset version but not evaluator version means a run's score can no longer be interpreted correctly once the scoring function itself has changed underneath it, since a low score might reflect a stricter evaluator rather than a worse run. Close it by treating the evaluator (judge prompt version, rubric version) as a first-class tracked parameter on every run, exactly like model and dataset version, so any run's score is always interpretable against the exact scoring function that produced it, and re-scoring old runs under a new evaluator is a deliberate, trackable action rather than an ambiguity.

---

**Situation:** You need to promote a new model version to production, but the team's current process is "whoever finishes testing just updates the alias." Last week two engineers did this within an hour of each other, and it's unclear which model is actually serving traffic right now. What would you fix?

Model answer: The promotion step needs governance metadata and a single source of truth, not just a free-for-all alias reassignment. Require every promotion to first tag the specific model version with governance metadata (who approved it, what evaluation run justified it, when) before reassigning the `champion` alias, and treat the alias reassignment itself as an auditable, logged action, not a quiet update anyone can make unilaterally. Going forward, gate the promotion step behind the deployment quality gate from Module 12 passing, so "finished testing" isn't a personal judgment call but a specific, checkable pass condition tied to one qualifying run.

---

**Situation:** A colleague argues that since MLflow already stores parameters and metrics, there's no need for separate live production monitoring — "we can just keep logging runs against production traffic the same way we do experiments." What's the flaw in that plan?

Model answer: Point out that offline experiment tracking and live production monitoring answer different questions and need different instrumentation. Experiment tracking tells you how a candidate *should* behave against a controlled evaluation set; it doesn't capture the continuous, high-volume, real-time signal of how a live system actually behaves under real traffic — TTFT, P95/P99 latency, cost per query, error rate, and cache-hit rate need to be tracked as an ongoing time series with alerting thresholds, not as periodic "runs." Recommend keeping MLflow for versioned experiments and promotions, and wiring live inference metrics into a dedicated observability stack (Module 14) that feeds back into the same registry when a production regression needs to trace back to an exact run and version.

---

**Situation:** Your org is choosing between MLflow, Weights & Biases, and Comet for a new team, and the deciding factor keeps coming back to "which one has the prettiest dashboards" in every meeting. How do you redirect the conversation?

Model answer: Redirect toward purpose, cost model, and production fit, which matter more than dashboard aesthetics for a decision that will be lived with for years. MLflow is open-source, self-hostable, and integrates tightly with a governed model registry — a strong fit for teams that need full control and already run their own infrastructure. Weights & Biases offers a particularly strong parallel-coordinates view for multi-metric Pareto-style comparison and a polished collaborative UI, often favored by research-heavy teams, but comes with a hosted-service cost model. Comet sits between the two on some axes. The right question is which tool's registry/governance model, hosting constraints, and cost structure actually fit the org's production requirements — not which demo looked nicest in a sales call.

---

**Situation:** A production incident traces back to a bad prompt template, and during the postmortem nobody can determine which exact experiment run first introduced that template into the registry, because the run that logged it didn't tag which PR or commit it came from. What do you add to the tracking discipline going forward?

Model answer: Add source-control provenance as a required tag on every run — commit SHA, PR number, and branch — alongside the existing model/prompt/dataset/evaluator tags, so any registered artifact can be traced back to the exact code change and human decision that produced it. This is the specific gap that breaks incident response: without it, "which run" is answerable but "which change, reviewed by whom, and why" is not, and that second question is what a postmortem actually needs. Treat this as a required field enforced by the logging wrapper, not an optional convention people forget under deadline pressure.

---

**Situation:** Your team wants to compare 15 candidate runs across quality, latency, and cost, and someone starts manually copying numbers into a spreadsheet to sort by quality descending. What's the better approach, and why?

Model answer: Use MLflow's query API (or W&B's parallel-coordinates view) to pull all 15 runs programmatically and reason about them on the Pareto frontier — the set of runs where no metric can be improved without making another worse — rather than sorting by a single column. Sorting by quality alone will surface the highest-quality run even if a nearly-as-good run exists at a fraction of the cost or latency, which is exactly the tradeoff a single-column spreadsheet sort hides. A short script against the tracking API that filters to Pareto-optimal runs turns a subjective "which one looks best" scan into a defensible, reproducible shortlist.

---

**Situation:** Three months after a model was promoted to `champion`, a new engineer asks how they can reproduce the exact evaluation that justified the promotion, and finds that the evaluation dataset referenced in the run's logged parameters no longer exists — it was overwritten in place. What governance gap does this expose?

Model answer: This exposes a missing versioning discipline on the dataset side of the tracked parameters — logging "dataset: eval-set-v1" as a parameter is only meaningful if that exact dataset version is immutably preserved somewhere, the same way a model artifact is. Fix it by treating evaluation datasets as versioned, frozen artifacts (Module 10's frozen evaluation set discipline) stored and referenced the same way model artifacts are, so a run's logged dataset parameter always resolves to something that still exists and matches exactly what was used at run time. Audit other "logged as a parameter but not actually preserved as an artifact" gaps in the tracking setup while fixing this one.

---

**Situation:** An engineer wants to log every single token generated during a long agentic run as individual metric data points to "have maximum visibility," and the tracking backend starts timing out under the volume. What would you recommend instead?

Model answer: Recommend stepping back from maximal granularity to purposeful granularity — logging every token as a discrete metric point is both far more data than is ever queried and expensive enough to degrade the tracking system for everyone. Aggregate to the level that's actually actionable: log per-call metrics (latency, token counts, cost) as scalars or a small time series per run, and reserve full raw content logging (if needed for debugging) for artifacts (a JSONL file attached to the run) rather than as high-cardinality metric time series. This keeps the tracking store fast and keeps the signal-to-noise ratio high enough that someone querying it can actually find what they need.

---

**Situation:** A candidate model version has excellent offline eval metrics logged in MLflow, but a skeptical stakeholder asks "how do we know this will actually behave well in production, not just on the eval set?" What's your answer?

Model answer: Acknowledge the concern is legitimate — offline evaluation tells you how a candidate should behave against a controlled set, not how it behaves under real, messier production traffic. Answer by describing the full loop: promote via alias to a canary slice of live traffic first, track live inference metrics (TTFT, P95/P99 latency, cost per query, error rate, cache-hit rate) against the same alerting thresholds used for the incumbent, and only widen the rollout once the canary's live metrics hold up. The offline eval score earns the right to a canary test, not a direct jump to full production traffic, precisely because eval-set performance and real-world performance can diverge.

---

**Situation:** During a system design interview, you're asked to explain why experiment tracking matters more for LLM applications than it did for classical ML. How do you answer?

Model answer: Explain that the search space and the definition of quality both got harder. Classical ML experiments varied a relatively small set of hyperparameters like learning rate and batch size; LLM-application experiments vary model, prompt, retrieval configuration, temperature, and dataset version simultaneously, producing a combinatorial explosion of runs that's unmanageable without structured tracking. And where classical ML had one closed-form quality number (accuracy, F1), LLM quality is multi-dimensional and probabilistic — correctness, relevance, safety, latency, and cost all need to be tracked and compared together, often via a noisy LLM-judge score, so a system of record that ties parameters, metrics, and artifacts together per run isn't a nice-to-have, it's the only way to make hundreds of experiments comparable and governable at all.
