# Module 02 — Hands-On Exercises

These exercises build on `tutorial.md` and `architecture.md`. Work through them in
order — each one assumes the code/artifacts produced by the previous one still exist in
your working directory. You'll need Python 3.12+, and by Exercise 2 you'll want
`pandera`, `scipy`, `pandas`, `scikit-learn`, and `mlflow` installed
(`pip install pandera scipy pandas scikit-learn mlflow`). A local MLflow server
(`mlflow server --backend-store-uri sqlite:///mlflow.db`) is enough for every exercise
that touches MLflow.

For each exercise: read the **Goal**, do the work, then check yourself against
**Definition of Done** before moving on. Don't skip the "done" check — later exercises
assume earlier artifacts exist and are correct.

---

## Exercise 1 (Warm-up) — Gate or No Gate? Scope Judgment Calls

**Goal:** Build intuition for *why* a given change needs an ML/LLM-specific gate (vs.
being adequately covered by standard DevOps CI) before you touch any tooling.

For each scenario below, decide (a) whether standard DevOps CI/CD (build + unit tests)
would catch the problem, and (b) if not, which of the three gates from `tutorial.md`
§3.1 (data validation, model evaluation, response/prompt quality) would:

1. A data vendor silently renames a column from `spend_usd` to `total_spend_usd`.
2. An engineer introduces an off-by-one bug in a feature-engineering function.
3. A nightly retrain produces a model with materially lower AUC than yesterday's,
   using clean data and bug-free code.
4. A non-engineer product manager edits a production system prompt directly in an
   internal admin tool, with no code change or PR.
5. An upstream marketing campaign shifts the customer geography mix feeding a churn
   model, with no schema violation anywhere.
6. A foundation-model provider silently upgrades the backing model version behind an
   API endpoint your service calls, with no version pin on your side.

**Definition of Done:** all six classified with a one-sentence justification each
(answers: 1-Gate 1/schema, 2-standard CI/unit test, 3-Gate 2, 4-Gate 3 / release
manifest problem, 5-Gate 1/drift not schema, 6-Gate 3 + versioning discipline from
Module 03). You should be able to explain *why* standard CI misses 1, 3, 4, 5, and 6 —
not just recite the label.

---

## Exercise 2 — Write a Pandera Schema Gate From Scratch

**Goal:** Get hands-on with schema validation as an enforced (not decorative) gate.

1. Generate a synthetic "churn" dataset of ~2,000 rows with columns
   `customer_id` (str, unique), `transaction_count_30d` (int ≥ 0), `avg_spend_usd`
   (float ≥ 0), `churn_label` (int, either 0 or 1).
2. Write a `pandera.pandas.DataFrameSchema` (model it on `ChurnSchema` in `tutorial.md`
   §3.4) that enforces all four constraints, with `strict=False`.
3. Deliberately corrupt three separate copies of the dataset: (a) drop the
   `churn_label` column entirely, (b) set 5 rows of `churn_label` to `2`, (c) convert
   `avg_spend_usd` to a string dtype.
4. Run `ChurnSchema.validate(df, lazy=True)` against the clean dataset and all three
   corrupted copies. Confirm the clean one passes and each corrupted one raises
   `pandera.errors.SchemaErrors` with the *specific* failing column named.
5. Wrap the check in a function that emits a structured JSON line (`{"gate":
   "data_validation", "status": ..., ...}`) and calls `sys.exit(1)` on failure — do not
   just `print()` a warning and continue.

**Definition of Done:**
- All three corrupted datasets fail validation with a clear, column-specific error
  message captured in your JSON output — not a generic stack trace.
- The clean dataset passes cleanly.
- Running your script against a corrupted file and checking `echo $?` (or
  `$LASTEXITCODE` on Windows) afterward shows a non-zero exit code — prove the gate is
  enforceable, not just informative.

---

## Exercise 3 — Add Row-Count and Drift Checks (Gate 1, Complete)

**Goal:** Extend Exercise 2 into the full three-part Gate 1 from `tutorial.md` §3.4:
schema + row-count + distribution drift.

1. Add a `MIN_ROWS = 1_000` check that fails (structured JSON + `sys.exit(1)`) if the
   incoming batch has fewer rows.
2. Save your clean dataset from Exercise 2 as `baseline.parquet`. Create a "current"
   batch by resampling `avg_spend_usd` from a shifted distribution (e.g. add a
   constant offset or scale it by 1.5x) for 30% of rows, leaving the rest unchanged.
3. Run `scipy.stats.ks_2samp(baseline["avg_spend_usd"], current["avg_spend_usd"])` and
   check `p_value < 0.05` as your drift-fail condition (as in `tutorial.md`).
4. Separately, implement PSI by hand (bucket both samples into the same 10 quantile
   bins derived from the baseline, then sum `(pct_actual - pct_expected) *
   ln(pct_actual / pct_expected)` across bins). Compare your PSI value against the
   standard thresholds table in `tutorial.md` §3.4 (< 0.10 stable, 0.10–0.25 slight,
   > 0.25 significant).
5. Run both drift checks against (a) an unshifted "current" batch and (b) your shifted
   one. Confirm KS-test and PSI **agree** on which one is drifting.

**Definition of Done:**
- Row-count check correctly fails on a truncated (e.g. 500-row) batch and passes on a
  full one.
- KS-test `p_value` and your hand-rolled PSI both flag the shifted batch as drifted and
  both pass the unshifted batch as stable.
- You can explain in one paragraph a scenario where KS-test and PSI would *disagree*
  (hint: think about sample size sensitivity and how each metric weights the tails vs.
  the bulk of the distribution).

---

## Exercise 4 — Wire Up the Model Evaluation Gate (Gate 2) with MLflow

**Goal:** Implement Gate 2 exactly as in `tutorial.md` §3.4 — block promotion of an
underperforming model.

1. Train a baseline `LogisticRegression` (or `RandomForestClassifier`) on your churn
   dataset from Exercise 2 and record its AUC as `BASELINE_AUC`.
2. Write `evaluate_model.py` (model it closely on the tutorial's version): load a
   candidate model, compute AUC on a held-out split, log it to MLflow via
   `mlflow.log_metric`, and `sys.exit(1)` if `auc < BASELINE_AUC` (or a chosen
   threshold like `BASELINE_AUC * 0.99`).
3. Train two deliberately different candidates: one trained on the *full* dataset
   (should beat or match baseline) and one trained on a 5% random subsample (should
   underperform).
4. Run the gate against both candidates and confirm one passes (exit 0) and one is
   correctly blocked (exit 1), with both runs visible as separate MLflow runs logging
   `auc` and `baseline_auc`.

**Definition of Done:**
- Both MLflow runs are queryable (`mlflow ui` or `MlflowClient`) and show the logged
  `auc` metric.
- The underperforming candidate's run correctly triggers a non-zero exit and a
  structured `"fail"` JSON log naming both the measured AUC and the baseline.
- You can articulate why this gate must run *before* any model registration step (tie
  it to the race-condition risk described in `tutorial.md` §7, Q7).

---

## Exercise 5 — Assemble the Full Fail-Fast GitHub Actions Workflow

**Goal:** Turn Exercises 2–4 into one production-shaped, trigger-driven CI/CD pipeline.

1. Write `.github/workflows/churn-retrain.yml` modeled directly on `tutorial.md` §3.3's
   full example: `schedule` + `workflow_dispatch` triggers, a `concurrency` group,
   fail-fast step ordering (Gate 1 before training, training before Gate 2), and Slack
   (or a stubbed `curl` to a local webhook receiver / `echo` if you have no Slack) on
   both success and failure.
2. Add a `dataset.hash` step (SHA-256 of the input parquet file) written alongside the
   trained model, and have your registration step use it as an MLflow tag — this is
   the idempotency mechanism from `tutorial.md` §3.3.
3. Locally simulate three runs of the pipeline (you can do this with a plain shell
   script calling your Python scripts in sequence, without needing a live GitHub
   Actions runner) corresponding to the three branches in `architecture.md` §2:
   Branch A (happy path — both gates pass), Branch B (Gate 1 blocked — truncate the
   input file), Branch C (Gate 2 blocked — use your underperforming candidate from
   Exercise 4).
4. Re-run Branch A twice with the *identical* input file and confirm the dataset hash
   is identical both times — prove idempotency, not just assert it.

**Definition of Done:**
- All three branches produce the correct terminal state: A registers a new model
  version and "notifies success"; B and C both stop before registration and "notify
  failure," with the *previous* model version still the one an alias/pointer resolves
  to.
- Your workflow YAML would pass `actionlint` (or at minimum, valid YAML syntax) even
  if you never run it on a real GitHub repo.
- The two identical re-runs of Branch A produce the same `dataset.hash` value.

---

## Exercise 6 — Add Gate 3: A Minimal LLM Response Quality Gate

**Goal:** Extend the pipeline to an LLM-serving scenario and implement the
LLM-specific gate from `tutorial.md` §3.1 and §3.4.

1. Define a small "golden eval set" of 10 fixed prompts for a customer-support-style
   assistant (e.g. "Explain why my order is delayed" style prompts), each with a short
   rubric of what a *good* answer contains (groundedness, no invented policy details,
   appropriate tone).
2. Write `evaluate_response_quality.py` that calls an LLM (any provider's API, or a
   stubbed/mocked function if you don't want to spend API budget) once per golden
   prompt and scores each response using a second LLM call acting as judge — ask the
   judge to output a JSON `{"score": 0-1, "reason": "..."}` per response.
3. Compute the mean judge score across all 10 prompts and fail (structured JSON +
   `sys.exit(1)`) if it's below a threshold you choose and justify (e.g. 0.80).
4. Deliberately test the failure path: swap in a deliberately bad "candidate" system
   prompt (e.g. one that encourages made-up policy details) and confirm the gate
   correctly fails, then swap back and confirm it passes.
5. Build a `ReleaseManifest` (from `tutorial.md` §3.2's code block) for this scenario
   and show that changing *only* the `prompt_template_version` field changes the
   manifest's `content_hash()` — proving the mechanism that would re-trigger Gate 3 on
   a prompt-only change with zero code change.

**Definition of Done:**
- The bad system prompt run scores measurably lower and correctly fails the gate; the
  good one passes.
- You can name at least one weakness of your own LLM-judge setup (e.g. judge score
  variance across repeated calls, the judge itself being foolable) — this is a
  deliberate check that you understand judge-based eval is noisier than a scalar
  metric like AUC, per `tutorial.md` §7, Q6.
- The manifest hash changes when (and only when) the prompt version field changes,
  confirmed by printing before/after hashes for a code-only change (hash unchanged)
  vs. a prompt-only change (hash changed).

---

## Exercise 7 (Capstone) — Full Three-Gate Pipeline with All Failure Branches, End to End

**Goal:** Combine everything into one runnable pipeline that exercises all three gates
and all failure branches described in `tutorial.md` and `architecture.md`, as a single
orchestrating script (`gates/run_all_gates.py`-style, per `tutorial.md` §3.1) that a CI
system could call as one step.

1. Assemble `run_all_gates.py` calling, in fail-fast order: your Gate 1 (Exercise 3),
   Gate 2 (Exercise 4), and Gate 3 (Exercise 6) — each as its own function, each
   emitting structured JSON and calling `sys.exit(1)` independently.
2. Construct and run five distinct scenarios end to end, logging the JSON output of
   each:
   - **All pass.** Clean data, a model that beats baseline, a good system prompt.
   - **Gate 1 fails on schema.** Corrupted `churn_label` values.
   - **Gate 1 fails on drift.** Distribution-shifted `avg_spend_usd`.
   - **Gate 2 fails.** Underperforming candidate model.
   - **Gate 3 fails.** Bad system prompt, but data and model are both fine.
3. For each of the four failing scenarios, confirm the pipeline stops at the failing
   gate and **does not** execute any gate after it (e.g. confirm Gate 3's function is
   never even called when Gate 1 fails — assert this with a counter or a mock, not
   just visual inspection of logs).
4. Write a one-paragraph "incident narrative" for the drift-failure scenario, written
   as if it happened in production: what would have silently gone wrong if only
   Gate 1's schema check (and not the drift check) had existed, tying it back to the
   "silent failure" concept in `tutorial.md` §1.
5. Produce a short table (5 rows, one per scenario) summarizing: scenario name, which
   gate failed (if any), exit code, and whether the previous model/prompt version would
   still be the one serving.

**Definition of Done:**
- All five scenarios produce the correct pass/fail outcome and correct exit code (0 for
  the all-pass case, 1 for each failure case).
- You have positive proof (not just an assumption) that a later gate's function is
  never invoked once an earlier gate fails — e.g., a call counter, `unittest.mock`
  assertion, or equivalent instrumentation.
- Your incident narrative correctly identifies that schema validation alone would have
  let the drifted batch through (same columns, same types, same value ranges — just a
  shifted population) and explicitly ties this back to the "schema vs. distribution
  are not redundant" point in `tutorial.md` §3.4.
- The summary table is accurate against your actual run logs, not reconstructed from
  memory.
