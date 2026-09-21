# Module 03 — Hands-On Exercises

These exercises build on `tutorial.md` and `architecture.md`. Work through them in order — each one assumes the artifacts (registered models, aliases, `release.yaml`) produced by the previous one still exist. Use a local MLflow server (`mlflow server --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns`) unless an exercise says otherwise.

For each exercise: read the **Goal**, do the work, then check yourself against **Definition of Done** before moving on. Don't skip the "done" check — several later exercises depend on artifacts you're expected to have created correctly.

---

## Exercise 1 (Warm-up) — Semantic Versioning Judgment Calls

**Goal:** Build intuition for what counts as MAJOR/MINOR/PATCH across the three artifact types before you touch any tooling.

Given the table in Section 3.1 of `tutorial.md`, classify each of the following changes as MAJOR, MINOR, or PATCH, and write one sentence justifying each:

1. A fraud-detection model is retrained weekly on a rolling data window; this week's retrain used the same architecture and added no new features.
2. A prompt's output format changes from free-text to a strict JSON schema that downstream code now parses.
3. A dataset gets 500 new duplicate rows removed, with no other changes.
4. A model is upgraded from a logistic regression baseline to a gradient-boosted tree architecture.
5. A prompt's wording is reworded for clarity but the task, inputs, and output format are unchanged.
6. A dataset schema adds a new required column that the model now depends on.

**Definition of Done:**
- All six items classified correctly (answers: 1-MINOR, 2-MAJOR, 3-PATCH, 4-MAJOR, 5-MINOR, 6-MAJOR) with a one-sentence justification each, referencing *why* the classification follows from the "breaks the contract with consumers" logic in the tutorial — not just pattern-matching to the table.

---

## Exercise 2 — Register a Model with the Current Alias-Based API

**Goal:** Get hands-on with the MLflow Model Registry using the 2026 alias/tag pattern — and prove you can spot the deprecated pattern if you see it.

1. Train a simple `RandomForestClassifier` (or any sklearn model) on a toy dataset (e.g. `sklearn.datasets.load_breast_cancer`).
2. Log the run to MLflow, log `val_accuracy` as a metric, and register the model under the name `demo-classifier`.
3. Using `MlflowClient`, tag the resulting version with `semver=1.0.0` and `dataset_version=demo-data:1.0.0`.
4. Set the alias `@challenger` on this version.
5. Write a short script that loads the model via `models:/demo-classifier@challenger` and confirms it can produce a prediction.
6. In a comment at the top of your script, explain in 2-3 sentences why `client.transition_model_version_stage(...)` would be **wrong** to use here in 2026.

**Definition of Done:**
- A registered model `demo-classifier` exists with at least one version, tagged with `semver` and `dataset_version`.
- The `@challenger` alias resolves and successfully loads a working model via `mlflow.pyfunc.load_model`.
- Your script never calls the deprecated stage-transition API, and your comment correctly explains why (fixed enum vs. flexible aliases, deprecated since MLflow 2.9).

---

## Exercise 3 — Write a Code-Driven Promotion Gate

**Goal:** Replace "looks fine to me" with an automated, defensible promotion decision.

Using the `demo-classifier` from Exercise 2:

1. Train a second version (change a hyperparameter, e.g. `max_depth`) and register it as version 2. Set `@challenger` to point to it.
2. Write a `passes_promotion_criteria()` function (model it on the tutorial's `fraud-detector` example) that checks:
   - Absolute accuracy ≥ some floor you choose and justify (e.g. 0.90).
   - Relative regression guard vs. the current `@champion` (if none exists yet, treat this as the first-ever promotion).
   - A fake "latency" gate — you can hardcode a `measure_latency_ms()` stub that returns a constant, but the gate logic must actually consult it.
3. Run the gate. If it passes, repoint `@champion`. If it fails, leave `@challenger` as-is and print a clear reason why it failed.
4. Now deliberately make version 2 **worse** (e.g. train on 5% of the data) and re-run the gate — confirm it correctly blocks promotion.

**Definition of Done:**
- The gate function returns `True`/`False` based on all three criteria, not just accuracy.
- You can demonstrate one run where promotion succeeds and one where it is correctly blocked, with printed reasoning for both.
- No manual alias-setting outside of what the gate function itself does upon passing.

---

## Exercise 4 — Version the Prompt Leg and Assemble a `release.yaml`

**Goal:** Close the "prompt" gap and produce the single-source-of-truth manifest the tutorial centers on.

1. Using `mlflow.genai.register_prompt(...)`, register a prompt named `demo-prompt` with an initial template and a commit message.
2. Set a `production` alias on it.
3. Make one wording-only edit (a PATCH-level change) and register a second version; do **not** move the `production` alias yet.
4. Hand-write a `release.yaml` (following the structure in `tutorial.md` Section 3.1) that references:
   - `demo-classifier` version and semver
   - `demo-prompt` version and semver
   - A fictitious dataset name/version of your choosing
   - A `compatibility` block with at least one real constraint (e.g. `required_prompt_major: 1`)
   - A `rollout` block with `stage: staging` and `approval: pending`

**Definition of Done:**
- `release.yaml` is valid YAML, parses cleanly, and every version it references actually exists in your MLflow instance (spot-check with `client.get_model_version` / `client.get_prompt_version`).
- You can explain, in one paragraph, what would happen if you bumped `demo-classifier` to require `dataset_major: 2` while your dataset stayed at `1.x` — i.e., demonstrate you understand what the `compatibility` block is *for*, even though you're not yet enforcing it in code.

---

## Exercise 5 — Enforce Compatibility Rules in CI (Intermediate → Advanced)

**Goal:** Move the compatibility check in `release.yaml` from "documentation" to "enforced."

1. Write a Python script `validate_release.py` that:
   - Loads a `release.yaml`.
   - Resolves the model, prompt, and dataset versions it references against the registry (or a stub if you don't have a live dataset registry).
   - Parses each version's semver and checks it against the `compatibility` block (e.g. rejects deployment if `dataset.version`'s major is below `min_dataset_major`).
   - Exits with a non-zero status code and a clear error message if any rule fails; exits 0 and prints a summary if all rules pass.
2. Wrap this script in a GitHub Actions workflow (or equivalent CI config) that runs on any pull request touching `release.yaml`, modeled on the "Promote Model Candidate" workflow in `tutorial.md` Section 3.3.
3. Prove it works: create one `release.yaml` that should fail validation (incompatible dataset major) and one that should pass, and show both CI outcomes.

**Definition of Done:**
- `validate_release.py` correctly fails on the incompatible case and passes on the compatible case, with human-readable error output either way.
- The CI workflow file is present and would actually run this script on a PR (dry-run/local `act` execution is acceptable if you don't have a live GitHub repo).
- You can explain why this check belongs in CI rather than as a comment/convention that engineers are expected to remember.

---

## Exercise 6 — Simulate a Canary Ramp with Rollback-on-Breach

**Goal:** Implement the traffic-ramp pattern from Section 3.4, including the automatic rollback trigger — without needing a real Kubernetes/service-mesh setup.

Build a small simulation (a single Python script is fine):

1. Model "traffic" as a loop of 1,000 simulated requests split between `@champion` and `@challenger` according to a ramp schedule: 5% → 25% → 100%, advancing every 200 requests if healthy.
2. For each request routed to `@challenger`, simulate an accuracy outcome by sampling from a distribution you control (e.g. Bernoulli with `p=0.85` to represent a *regressing* challenger, and a second run with `p=0.92` to represent a *healthy* one).
3. After every 50 challenger requests, compute a rolling accuracy; if it drops below a threshold you define (e.g. 0.87), immediately trigger `canary_percent = 0` and log a rollback event with reason, timestamp, and the rolling accuracy that triggered it.
4. Run the simulation twice: once where the challenger is healthy (should ramp all the way to 100%) and once where it's regressing (should trigger the automatic rollback before reaching 100%).

**Definition of Done:**
- Both simulation runs produce distinct, correct outcomes (full ramp vs. early automatic rollback).
- The rollback path prints/logs a structured event (reason, timestamp, rolling metric value, ramp stage at time of rollback) — not just a bare "rolled back" message.
- You can articulate, in your own words, why this rolling-window approach is closer to production reality than checking a single request's outcome.

---

## Exercise 7 (Capstone) — Full-Tuple Lineage-Based Rollback Under a Simulated Incident

**Goal:** Reproduce the entire incident narrative from Section 3.4 end-to-end, using your own artifacts, and prove your rollback restores the *whole* triple — not just the model.

1. Create at least three "release" records in a dedicated MLflow experiment (`releases`), each tagged with `release_id`, `model_version`, `prompt_version`, `dataset_version`, and `status` (`GOOD`/`BAD`). Make the most recent one `BAD` and at least one earlier one `GOOD`.
2. Implement `get_last_known_good(before_release_id)` exactly as sketched in `tutorial.md` Section 3.4 — it must query by `status = 'GOOD'` and return the most recent one strictly before the bad release, not simply "the previous release regardless of status." Prove this distinction matters by inserting a *second* BAD release between two GOOD ones and confirming your function still skips over it correctly.
3. Implement `restore_full_tuple(...)` that repoints the model alias, "repoints" the prompt alias, and updates a stubbed `update_serving_dataset_pointer()` function — log to console what each step does.
4. Write a rollback audit event to the `releases` experiment recording reason, operator, restored versions, and timestamp.
5. Write a short incident report (5-10 lines, timestamped like the tutorial's example) narrating what you simulated: what broke, how you detected it (even if "detection" here is just you deciding to trigger it), what LKG tuple you found, and confirmation that all three legs were restored — not just the model.

**Definition of Done:**
- `get_last_known_good` correctly skips a BAD release sandwiched between two GOOD ones (this is the core test — a naive "get the previous release" implementation would fail here).
- `restore_full_tuple` demonstrably updates all three legs (model alias actually changes in the registry; the other two are logged even if stubbed) and writes one audit run/record with all required fields.
- Your incident report explicitly states why a model-only rollback would have been insufficient in your scenario (tie it back to the "interaction between components" reasoning in Section 3.4).
- Bonus (optional, for the ambitious): wire this into the CI validation script from Exercise 5 so that a rollback also re-validates the restored tuple against `release.yaml` compatibility rules before declaring success.
