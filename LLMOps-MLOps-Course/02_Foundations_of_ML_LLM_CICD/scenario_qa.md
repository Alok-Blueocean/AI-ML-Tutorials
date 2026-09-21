# Module 02 — Scenario-Based Q&A

**Situation:** A nightly retrain pipeline has 100% green CI (unit tests, integration tests, deploy step all pass), but the newly deployed model's real-world accuracy has quietly dropped 8 points over the past week. What would you do and why?

Model answer: Diagnose this as the module's central "silent failure" pattern — infrastructure health (green pipeline, service up) is orthogonal to model quality, because unit tests validate code paths, not statistical correctness. Check whether a model-evaluation gate (Gate 2) even exists in this pipeline; if it doesn't, that's the root cause, not a bug in an existing gate. Add an evaluation gate that compares the candidate model against a versioned baseline metric (e.g., AUC ≥ baseline × 0.99) before any registration/promotion, and retroactively check whether the past week's silently-bad model can be identified and rolled back using registry history.

---

**Situation:** An engineer proposes adding a data validation gate that, on failure, logs `"WARNING: schema mismatch, proceeding anyway"` and lets the pipeline continue, reasoning that "we don't want a hiccup in the data to block the whole nightly retrain."

Model answer: Reject this design as not actually a gate — the exit code is the entire enforcement mechanism a CI/CD system reacts to, and a script that logs a warning and returns 0 is invisible to the orchestrator regardless of what it printed. Explain the correct failure mode instead: a validation failure should call `sys.exit(1)`, which blocks training compute from being spent on bad data and keeps the previous model serving. If the concern is that occasional legitimate data hiccups shouldn't halt the pipeline for days, address that with an on-call alert plus a fast manual override path (a human explicitly re-running with `workflow_dispatch`), not a silently-continuing check.

---

**Situation:** Your team's retraining pipeline currently trains the model first (a 40-minute GPU job) and only checks the data schema afterward, right before registration. A teammate asks whether the order matters, since "the checks happen either way."

Model answer: Yes, order matters — this violates fail-fast ordering. Schema and row-count checks cost milliseconds; training costs 40 minutes of GPU compute. Running the expensive step before the cheap validation means every bad-data incident wastes 40 minutes of compute and delays the failure signal by that same 40 minutes, compared to catching it up front. Reorder to: row-count check → schema check → (optionally sampled) distribution/drift check → train → model evaluation gate — cheapest and most-likely-to-catch-a-problem first, most expensive last.

---

**Situation:** A data batch passes schema validation cleanly (every column present, every type correct, every value within declared range) but the model trained on it performs unexpectedly poorly in evaluation. The data engineer insists "the data passed validation, so the data is fine — it must be a training bug."

Model answer: Explain that schema validation and distribution validation are not redundant, and this is the textbook case where schema-only validation misses a real problem: a batch can be structurally identical to expected (same columns, types, ranges) while representing a meaningfully different population than the training baseline — for example, a marketing campaign shifting the customer geography mix. Recommend running a KS-test or PSI check comparing the new batch's feature distributions against the baseline before concluding it's a training bug; if PSI exceeds roughly 0.25 on a key feature, that's the actual root cause, and adding a distribution-validation gate prevents this exact ambiguity next time.

---

**Situation:** A retraining pipeline fails partway through due to a transient network error while pulling data. On retry, it produces what looks like a slightly different model even though the same 30-day data window was requested.

Model answer: Diagnose this as an idempotency gap. If the pipeline doesn't hash the input dataset and tag that hash on the registered model, a retry after a transient failure can silently produce a different "same-day" model version — because of subtle non-determinism in data ordering, sampling, or hardware kernel behavior — without anyone noticing the two runs weren't actually identical. Fix by writing a dataset content hash alongside the trained model and using it as an MLflow tag; if a retry against the same window produces the same hash, registration becomes a safe no-op instead of a confusing duplicate "new" version, and postmortems and rollback stay unambiguous.

---

**Situation:** Your CI pipeline for an LLM-backed feature currently only runs a model-evaluation-style AUC comparison, inherited unmodified from a classical ML pipeline template. A new PR changes the system prompt.

Model answer: Point out the template doesn't fit — AUC-style evaluation assumes a fixed-label classifier; a prompt change produces open-ended text output that a scalar metric comparison can't meaningfully assess. Recommend adding a Gate 3 (response/prompt quality) that runs the new prompt against a fixed, versioned golden eval set and scores coherence, groundedness, and safety via LLM-as-judge (or rubric-based scoring), gating promotion on a minimum score threshold — not reusing Gate 2's classical-model comparison logic. This is exactly why LLM eval gates are described as qualitatively, not just quantitatively, harder than classical ML eval gates.

---

**Situation:** A teammate wants to set the model-evaluation baseline threshold once, based on the best AUC ever achieved during an unusually strong training run six months ago, and never revisit it.

Model answer: Warn against a static, never-revisited threshold — it will either rot (if the underlying problem naturally gets harder over time, a stale threshold either blocks every legitimate retrain forever) or, if set too loosely from a lucky run, let mediocre models slip through. Recommend assigning a named owner and a review cadence to the threshold, and treating the baseline itself as a value that should track the current production model's actual measured performance (e.g., `baseline * 0.99` relative to whatever is live now), not a fixed historical high-water mark.

---

**Situation:** During a compute-cost review, someone proposes running the full LLM-judge-based response-quality evaluation (Gate 3) on every single commit to the prompt repository, including trivial whitespace and formatting-only edits, arguing "we should never skip evaluation."

Model answer: Push back on blanket "never skip" as the wrong lens — the actual engineering judgment call is where to place each gate and how strict/frequent it needs to be, matching validation depth to risk and cost. A whitespace-only prompt edit is extremely unlikely to change model behavior meaningfully; running full inference-time LLM-judge evaluation on every such commit burns real compute for near-zero signal. Recommend a lightweight pre-check (e.g., a semantic diff that flags whether the actual instruction text changed, not just formatting) that decides whether the expensive Gate 3 needs to run at all — reserving full evaluation for commits that materially change the prompt's content.

---

**Situation:** After a gate-blocked deployment, an on-call engineer under time pressure manually edits the registry to promote the rejected model anyway, reasoning "the eval score was close enough and we need this shipped tonight."

Model answer: Call this out as defeating the entire purpose of the gate — a threshold exists precisely to remove subjective, under-pressure judgment calls from release decisions, and a manual override that bypasses it reintroduces exactly the risk the gate was built to prevent. If the business genuinely needs an exception, the right process is a reviewed, logged, and time-boxed manual override with an explicit approver and reason recorded (similar to the regulated-domain human-sign-off pattern), not a silent registry edit that leaves no audit trail and looks, to future incident responders, identical to a model that legitimately passed its gate.

---

**Situation:** A recommendation system's Gate 2 checks NDCG@10 against a held-out slice, and a new model change also affects the system's generated natural-language explanations ("Because you watched..."). The team wants to ship as soon as the NDCG check passes.

Model answer: Point out that NDCG@10 only validates the ranking quality, not the explanation quality — these are two independently gate-worthy outputs of the same release. Recommend a second, LLM-judge-scored coherence gate specifically for the explanations (e.g., coherence ≥ 0.85 on a fixed prompt set) before shipping, mirroring the production-grade case study in this module. Shipping on NDCG alone risks a release where recommendations improved but explanations became incoherent or factually inconsistent with the actual recommendation — a regression the ranking metric is structurally blind to.

---

**Situation:** A platform team wants every product team at the company to hand-roll their own schema validation, drift detection, and model-evaluation gate code independently, arguing "each team understands their own data best."

Model answer: Push back using the Uber/Michelangelo-style reasoning from this module's case studies — at multi-team scale, hand-rolled gates independently reinvented per team tend to be under-implemented inconsistently (some teams get PSI thresholds right, others skip distribution checks entirely), and a shared incident later becomes harder to debug because every team's gate logic and structured-log format differs. Recommend a shared library providing schema validation, PSI/KS drift checks, and a common structured JSON log format that every team's pipeline calls into — teams still supply their own schema definitions and thresholds (since they do know their own data best), but the enforcement mechanism and logging contract stay uniform across the org.

---

**Situation:** Your team's `validate_data.py` uses Pandera with `lazy=False` (the default in older code), so it raises and exits on the very first schema violation it finds, even when a batch has five separate structural problems.

Model answer: Recommend switching to `lazy=True`, which collects all schema failures in one validation pass instead of stopping at the first. The extra compute cost is small, and the payoff is concrete: a single validation run that reports all five things wrong with a batch saves four subsequent round trips compared to a validator that surfaces one error, gets fixed, re-runs, and immediately hits the next one — meaningfully faster incident resolution when a vendor feed has multiple simultaneous issues.

---

**Situation:** A security review flags that your CI/CD pipeline's data-validation step currently has the same broad credentials (including production model-registry write access) as the deployment step, even though validation only needs read access to a data file.

Model answer: Agree this is a real gap and fix it by scoping credentials per gate, not per pipeline — validation/eval scripts should have narrowly scoped, typically read-only access to the data they check, and only the final promotion step should hold write credentials to the registry or serving environment. This isn't just defense-in-depth in the abstract: a compromised or buggy validation script with registry-write access could itself become the mechanism for a bad model reaching production, which is precisely the "promotion should be a separate, more tightly controlled step than the eval script said pass" principle this module recommends.

---

**Situation:** A teammate argues that since their internal research notebook model "will never touch production traffic," it doesn't need any of the three-gate CI/CD machinery from this module, and pushes back when you suggest adding even a minimal check.

Model answer: Agree with the core judgment, but clarify the boundary precisely: a one-off exploratory model with no production path and no recurring retrain cadence genuinely doesn't need the full three-gate pipeline — gates are an investment that pays off across repeated runs, and over-gating exploratory work slows iteration for no safety benefit. The trigger for adding gates isn't "is it currently research" but "will this be retrained on a recurring cadence, run in CI, or ever have a real production consumer" — recommend revisiting the decision the moment any of those becomes true, rather than defaulting to heavyweight tooling now or permanently deferring it.

---

**Situation:** During an incident postmortem, the team discovers that a validation script correctly detected a schema violation, correctly logged a structured JSON error message describing it — but the pipeline still proceeded to train and promote a bad model.

Model answer: Investigate the exit code specifically, since structured logging and non-zero exit are two separate, both-mandatory operating rules — a script can produce a perfect diagnostic log and still fail to stop the pipeline if it doesn't also call `sys.exit(1)` (for example, if the failure path uses a `return` inside a try/except instead of exiting the process). Confirm this exact bug pattern by testing the validation script's failure path directly, not just its happy path, and add an explicit CI test asserting that a known-bad input causes the pipeline job to be marked failed — closing the gap between "detected the problem" and "the problem actually stopped anything."

---

**Situation:** A frontier-lab-style team is deciding whether staged rollout (canary → percentage ramp) should happen before or after their full offline LLM-judge evaluation suite passes.

Model answer: Recommend staged rollout as an additional gate layered after the offline eval suite, not a replacement for it — offline evals catch known, anticipated failure modes cheaply and at scale, but cannot fully capture real-world usage-distribution shift, so a canary is the safety net for the failure modes offline evaluation structurally can't see. Running staged rollout only, without an offline eval gate first, wastes real user exposure on failure modes a cheap, fast offline check could have caught before any live traffic was involved.

---

**Situation:** A junior engineer asks why the churn-retrain pipeline example in this module posts a Slack notification on both success and failure, rather than only alerting when something breaks.

Model answer: Explain the reasoning: a success notification (new version registered, accuracy delta reported) gives the team passive visibility into what's actually shipping without anyone having to check the registry manually, and — more importantly for trust in the system — makes the pipeline's *silence* meaningful. If the team only ever hears from the pipeline on failure, a long stretch without any Slack message is ambiguous (did it run and pass quietly, or silently stop running entirely?); a success message closes that ambiguity and makes a missing message itself a detectable anomaly.

---

**Situation:** Your organization is deciding whether to hand-roll drift detection with raw `scipy.stats.ks_2samp` calls per feature, or adopt Evidently AI's Test Suites, for a model with around 40 input features.

Model answer: Use the module's own guidance directly: hand-rolled KS-test calls are reasonable for a handful of key features, but at roughly 40 features, a hand-rolled per-feature loop becomes its own maintenance burden (threshold tuning, dashboard-building, alerting logic all built from scratch). Recommend adopting Evidently AI Test Suites at this scale, since it packages PSI/KS/Jensen-Shannon checks as ready-made CI-friendly pass/fail checks — this is exactly the point in the "how many features do you need to monitor, and how often" decision tree where the module recommends moving off hand-rolled checks.

---

**Situation:** A stakeholder asks how a classical model-evaluation gate (AUC ≥ baseline) would need to be redesigned if the team's product later adds an LLM-generated summary layer on top of the classical model's predictions.

Model answer: Explain this requires adding a second, structurally different gate rather than modifying the existing one — the classical model's AUC gate stays as-is for the underlying prediction quality, and a new Gate 3 (response/prompt quality) is needed specifically for the LLM summary layer, since it produces open-ended text that a scalar accuracy metric can't assess. This new gate needs its own golden eval set and typically an LLM-as-judge scorer for coherence, groundedness (does the summary accurately reflect the classical model's actual prediction and its inputs), and safety — a genuinely more expensive and noisier check than the AUC comparison it sits alongside, not a drop-in replacement for it.
