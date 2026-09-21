# Module 02 — Quiz: Foundations of ML and LLM CI/CD

Test your understanding of `tutorial.md` and `architecture.md`. Mix of multiple-choice
(MC) and short-answer (SA) questions. Try to answer without looking back at the
tutorial first — then check the answer key at the end.

---

**Q1 (MC).** Why is "unit tests pass" insufficient as the sole quality gate for an ML
or LLM system?

A. Unit tests are inherently unreliable and produce false positives
B. Unit tests validate code paths, not data quality or model quality, and a system can
   have 100% code coverage while training on corrupted data or serving a degraded model
C. Unit tests take too long to run in CI, so teams skip them anyway
D. ML systems don't need automated testing of any kind

---

**Q2 (MC).** What is the key assumption that traditional (code-centric) DevOps CI/CD
makes, which ML/LLM systems specifically violate?

A. That code should be reviewed before merging
B. That behavior is a pure function of code alone, given the same inputs
C. That deployments should happen during business hours
D. That all tests must run in under 10 minutes

---

**Q3 (SA).** Name the three "extra" gates this module adds on top of standard
build/unit-test/integration-test/deploy gates, and state in one phrase each what
failure mode each one catches.

---

**Q4 (MC).** A batch of incoming data passes Pandera schema validation perfectly (all
columns present, correct types, values within declared ranges) but represents a
meaningfully different customer population than the training data (e.g., after a
marketing campaign shifted geography mix). What has this scenario demonstrated?

A. Pandera has a bug
B. Schema validation and distribution/drift validation catch different failure classes,
   and passing one does not imply passing the other
C. The marketing campaign should have been blocked by CI/CD
D. Row-count validation was insufficient

---

**Q5 (SA).** What is PSI (Population Stability Index), and give the three standard
threshold bands and their interpretations.

---

**Q6 (MC).** In the fail-fast ordering principle, why should a row-count/schema check
run *before* training a candidate model rather than after?

A. Training always fails automatically if data is bad, so the order doesn't matter
B. Cheap checks (milliseconds) should catch problems before expensive steps (minutes to
   hours of training compute) are spent on data already known to be unusable
C. GitHub Actions requires validation steps to run first by convention
D. Schema checks are only valid before training, never after

---

**Q7 (MC).** What does "idempotency" mean in the context of an ML retraining pipeline?

A. The pipeline can only be run once per day
B. Re-running the pipeline with the same input data produces the same output (e.g., an
   identical model content hash), which matters for safe retries after transient
   failures
C. The pipeline never fails
D. Every run must produce a strictly different model version number

---

**Q8 (SA).** Explain the concrete mechanism (what gets computed, and where it's used)
by which the churn-retrain pipeline in `tutorial.md` achieves idempotency across
retries.

---

**Q9 (MC).** Which statement correctly describes the release-unit relationship between
DevOps, MLOps, and LLMOps?

A. They are three competing, mutually exclusive methodologies — a team picks one
B. They are nested, expanding scopes: MLOps requires everything DevOps requires plus
   data/model concerns; LLMOps requires everything MLOps requires plus
   prompt/retrieval/context concerns
C. LLMOps replaces the need for DevOps practices like code review and CI builds
D. MLOps and LLMOps are the same discipline with different names

---

**Q10 (MC).** According to `tutorial.md` §3.4, what is the correct way for a validation
script to signal failure so that a CI/CD system actually stops the pipeline?

A. Print a message containing the word "ERROR" to stdout
B. Raise any Python exception, regardless of exit code
C. Emit a structured JSON log line AND exit with a non-zero status code (e.g.
   `sys.exit(1)`) — the exit code is what the CI/CD system actually reads
D. Log a "WARNING" level message and return normally, so the pipeline can decide later

---

**Q11 (SA).** Describe a concrete way a team can accidentally defeat their own data
validation gate without anyone realizing it — i.e., the check still technically "runs"
and even correctly detects the problem, but the pipeline proceeds anyway.

---

**Q12 (MC).** A production model's infrastructure dashboard shows 100% uptime, low
latency, and zero 5xx errors, but the business conversion metric has quietly degraded
over the past week. What concept from this module does this scenario illustrate?

A. A false positive in the monitoring system
B. "Silent failure" — infrastructure health and model/business quality are orthogonal;
   a service can be fully "up" while its predictions are simply wrong
C. Proof that infrastructure monitoring is unnecessary for ML systems
D. A sign that the model should be retrained daily instead of weekly

---

**Q13 (MC).** Why does an LLM-based system's Gate 2/Gate 3 evaluation tend to be
qualitatively (not just quantitatively) harder than a classical ML model's evaluation
gate?

A. LLMs are always slower to run than classical models, which is the only difference
B. Classical evaluation compares a single scalar metric (AUC, accuracy) against a fixed
   baseline; LLM evaluation must assess open-ended free-text quality (coherence,
   grounding, hallucination, safety), typically via a noisier LLM-as-judge score that
   itself needs validation
C. LLMs cannot be evaluated offline at all, only in production
D. There is no meaningful difference; the same AUC-style threshold applies directly

---

**Q14 (SA).** In an MLflow-based pipeline, where should the model-evaluation gate sit
relative to model registration, and why does the ordering matter?

---

**Q15 (MC).** Per the "when NOT to over-engineer" guidance in `tutorial.md` §3.1 and
§3.3, which scenario is the best candidate for *skipping* the full three-gate CI/CD
machinery?

A. A recommendation model retrained nightly and serving live production traffic
B. A one-off research notebook model that will never be retrained and has no
   production consumer
C. A customer-support LLM assistant with a weekly prompt-iteration cadence
D. A fraud-detection model gated on AUC and deployed via canary rollout

---

## Answer Key

**Q1.** B — Unit tests validate code correctness, not data or model quality; a pipeline
can have full test coverage and still silently train/serve a degraded model because
none of that is a code-path problem.

**Q2.** B — Traditional DevOps CI/CD assumes behavior is a pure function of code;
testing the code is therefore sufficient. ML/LLM systems break this because data,
model artifacts, and (for LLMs) prompt/retrieval config can each independently change
behavior with zero code change.

**Q3.** (1) Data validation — catches schema drift, missing/null values, incomplete
loads. (2) Model evaluation — catches a newly trained model that is objectively worse
than the one in production. (3) Response/prompt quality (LLM-specific) — catches
hallucination, incoherence, or unsafe output from a new prompt/model/RAG version.

**Q4.** B — This is the "schema validation and distribution validation are not
redundant" point: schema answers "is this the same kind of data structurally?"; drift
answers "is this the same population statistically?" Data can pass one while failing
the other.

**Q5.** PSI is a single scalar summarizing how much a feature's distribution has
shifted between a baseline sample and a current sample — the standard "traffic light"
drift metric. Bands: **< 0.10** = stable, no significant shift; **0.10–0.25** = slight
drift, worth watching but not necessarily blocking; **> 0.25** = significant drift,
investigate and likely block promotion.

**Q6.** B — Fail-fast ordering means cheap, likely-to-catch-a-problem checks
(row-count/schema, milliseconds) run before expensive steps (training, minutes to
hours), so no compute is wasted training on data that was already known-bad.

**Q7.** B — Idempotency means the same input reliably produces the same output
(verifiable via a content hash); this matters because production pipelines get retried
after transient failures, and without idempotency a retry could create a confusingly
different "new" model version.

**Q8.** `train.py` computes a SHA-256 hash of the input dataset and writes it alongside
the trained model (`dataset.hash`). `register_model.py` uses this hash as an MLflow
tag on registration. If the workflow is re-run against the *same* input window, the
resulting model hash is identical, so re-registration becomes a safe no-op rather than
creating a confusing duplicate "new" version.

**Q9.** B — DevOps, MLOps, and LLMOps are nested, expanding scopes: each is a superset
of the previous one's concerns applied to a wider release unit (code → +data +model →
+prompt +retrieval +context). A team practicing LLMOps still needs every DevOps
discipline; it just isn't sufficient alone.

**Q10.** C — Both are mandatory and separate: structured JSON logs make failures
machine-parseable and debuggable, but the exit code is the *entire* enforcement
mechanism a CI/CD system reads to decide whether to continue or stop.

**Q11.** Logging a validation/schema failure as a `"WARNING"`-level message (or
printing a warning string) and then letting the script `return` normally instead of
calling `sys.exit(1)` — the check still runs and even correctly detects the problem in
its logs, but since the CI/CD system only reacts to the process's actual exit code, a
non-zero-less "detected" failure is invisible to the pipeline and bad data/models
proceed anyway.

**Q12.** B — This is a "silent failure": all infrastructure health checks (uptime,
latency, error rate) are orthogonal to model/business quality; a service can respond
200 with well-formed output on every request while being statistically wrong, which
infra dashboards alone will never surface.

**Q13.** B — Classical model evaluation is a cheap, largely deterministic scalar
comparison. LLM evaluation must judge open-ended generated text for coherence,
groundedness, and safety, typically requiring an LLM-as-judge whose scores have
variance and which itself can drift or be gamed — a qualitatively harder problem, not
just a slower version of the same check.

**Q14.** The evaluation gate must run *before* registration/promotion: evaluate the
candidate against baseline first, and only register to MLflow (or transition an
alias/stage) if it passes. Evaluating after registering risks a race condition where
something downstream picks up the not-yet-validated version before the gate has even
run.

**Q15.** B — A one-off notebook model with no recurring retrain cadence and no
production consumer doesn't benefit from gate machinery, since gates are an investment
that pays off across *repeated* runs; the other three options all have a recurring
cadence and/or a real production consumer, which is exactly when the full gate
machinery earns its cost.
