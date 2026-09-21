# Module 19 — Exercises: Drift Detection and Retraining Decisions

These exercises build from hand-computing drift statistics to assembling a full decision-matrix-driven monitoring pipeline with alerting. Do them in order — later exercises reuse code from earlier ones.

---

## Exercise 1 (Beginner) — Compute PSI and KS by Hand on Synthetic Data

**Goal:** Build intuition for what PSI and the KS statistic actually measure by computing both yourself on data where you control the ground truth about whether drift exists.

**Task:**
1. Generate a synthetic "baseline" numeric feature: 5,000 samples from `Normal(mean=100, std=15)`.
2. Generate four "current" samples, each 1,000 points: (a) drawn from the *same* distribution (no drift), (b) `Normal(mean=105, std=15)` (small mean shift), (c) `Normal(mean=130, std=15)` (large mean shift), (d) `Normal(mean=100, std=30)` (variance-only shift, same mean).
3. Implement `compute_psi` and `compute_ks_test` yourself (do not import a library that computes PSI directly — implement the binning and formula from the tutorial's §3.2 code).
4. Print PSI, PSI verdict (stable/moderate/significant), KS statistic, and KS p-value for all four current samples against the same baseline.

**Done looks like:**
- Case (a) shows PSI < 0.1 and a KS p-value that is *not* tiny (no false alarm on genuinely identical distributions).
- Case (b) shows PSI in the moderate range.
- Case (c) shows PSI solidly in the "significant" range and a very small KS p-value.
- Case (d) — the variance-only shift — is detected by PSI/KS (both are sensitive to spread, not just mean) even though the mean didn't move; you can explain *why* in one sentence referencing how PSI bins interact with a wider spread.
- A short written note (3-5 sentences) explaining, in your own words, why PSI needs bin edges derived from the baseline and what would go wrong if you derived them from the current sample instead — then demonstrate the bug by actually doing it wrong and showing the understated PSI.

---

## Exercise 2 (Beginner/Intermediate) — Multi-Feature Drift Report

**Goal:** Move from a single feature to a full tabular drift report, the shape every production drift job produces.

**Task:**
1. Take (or construct) a small tabular dataset with at least 5 numeric features and 1,000+ rows (a classic dataset like the UCI Adult/Census dataset, or a synthetic one you generate, both work).
2. Split it into a baseline half and inject drift into 2 of the 5 features in the "current" half (shift the mean, or resample from a different distribution) while leaving the other 3 features untouched.
3. Implement the `feature_drift_report` function from the tutorial (§3.2), producing a DataFrame with PSI, PSI verdict, KS statistic, and KS p-value per feature, sorted by PSI descending.
4. Confirm the report correctly identifies your two drifted features at the top and the three untouched features as stable.

**Done looks like:**
- A single function call producing a sorted drift report DataFrame.
- The two intentionally-drifted features clearly separated from the three stable ones by PSI value (at least a 3x gap).
- A brief explanation of one feature where PSI and KS *disagree* on severity ranking (if none arises naturally, engineer one by drifting a feature's tails without moving its median) and why that can happen given what each statistic actually measures.

---

## Exercise 3 (Intermediate) — Embedding-Space Drift for Text Queries

**Goal:** Extend drift detection beyond tabular data into free text, the LLM-specific case §3.2 covers.

**Task:**
1. Assemble (or synthesize) two sets of short text queries: a "baseline" set of ~200 queries about one topic domain (e.g., cooking questions) and a "current" set of ~200 queries, where you deliberately create three variants: (a) same domain (no real drift), (b) a mixed set — half cooking, half a completely different domain (e.g., tax/legal questions), (c) same domain but noticeably longer/more complex phrasing.
2. Embed all queries with any embedding model you have access to (a local sentence-transformers model, or any API embedding endpoint you already have configured elsewhere in this course).
3. Implement `embedding_centroid_drift` and `embedding_dispersion_delta` from §3.2 and compute both for all three current variants against the baseline.
4. Write up which of the two metrics (centroid distance vs. dispersion delta) catches which variant, and which variant (if any) neither metric catches well.

**Done looks like:**
- Variant (b), the mixed-domain set, shows a clearly larger centroid distance than variant (a).
- You can explain, with your own numbers, a case where dispersion delta moves but centroid distance barely does (or construct one deliberately if your first attempt doesn't produce it) — demonstrating why relying on centroid distance alone is insufficient.
- A short paragraph on what real production signal (e.g., a support-bot suddenly getting billing questions when it was trained/tuned for technical questions) this experiment is a stand-in for.

---

## Exercise 4 (Intermediate) — Behavioral Drift Metrics from Response Logs

**Goal:** Build the output/behavioral drift side of the monitoring pipeline, independent of any input-drift computation.

**Task:**
1. Construct a synthetic log of 500 "baseline" LLM responses and 200 "current" responses as plain strings (you can template-generate these — vary length, and inject explicit refusal phrases like "I can't help with that" into roughly 3% of the baseline and a higher rate, say 15%, of the current set to simulate a real regression).
2. Implement `compute_behavioral_metrics` from §3.1 (mean/P95 response length, refusal rate) for both sets.
3. Compute a z-score for the shift in mean response length (baseline mean/std vs. current mean) and compare the refusal rates directly.
4. Feed the results into `evaluate_thresholds` from §3.4 using the `DriftThresholds` defaults, and print the resulting alert list.

**Done looks like:**
- The refusal-rate shift correctly triggers a "critical" alert given the defaults in the tutorial.
- You can state, in one sentence, why this exercise's signal (behavioral drift) would be **completely invisible** to a monitoring system that only checks input-side PSI on user query features — tying back to §3.1's core distinction.

---

## Exercise 5 (Intermediate/Advanced) — Implement and Test the Decision Matrix

**Goal:** Turn the theory in §3.3 into a tested, trustworthy piece of decision logic — the component every other exercise's alerts should ultimately feed into.

**Task:**
1. Implement `DriftEvidence` and `decide_action` exactly as in §3.3 (or your own refined version, if you have a principled reason to change the rule ordering — document the reason if so).
2. Write at least 8 unit tests (pytest is fine, or plain asserts), one for each meaningfully distinct combination of `(input_drift_high, behavioral_drift_high, quality_dropping, recent_change_detected)` discussed in the tutorial's decision-matrix table, asserting the expected `Action` is returned.
3. Add one deliberately ambiguous/edge case not explicitly covered in the tutorial's table (e.g., all four booleans `True` simultaneously) and write a test asserting your chosen resolution, with a one-sentence comment explaining your reasoning.

**Done looks like:**
- All unit tests pass.
- The edge case you added has an explicit, justified resolution rather than silently falling through to a default — and you can explain why that resolution is defensible.
- A short reflection (3-5 sentences) on what would break if this logic were instead a black-box ML classifier trained on past incidents rather than a transparent rule table — referencing the tutorial's argument for staying rule-based.

---

## Exercise 6 (Advanced) — Threshold Calibration Against a Historical Noise Floor

**Goal:** Practice the hardest, most-skipped part of production drift monitoring: setting thresholds from evidence rather than intuition.

**Task:**
1. Simulate 90 days of a "healthy" refusal-rate time series (e.g., daily refusal rate sampled from `Normal(mean=0.03, std=0.008)`, clipped to `[0, 1]`) representing a known-good historical period with no real incidents.
2. Compute the empirical mean and standard deviation of this healthy series, and derive an alert threshold at `mean + 3*std` per the tutorial's §3.4 calibration procedure.
3. Now simulate 14 more days that include one genuine 5-day regression (refusal rate jumps to `Normal(mean=0.14, std=0.02)` for days 91-95, then returns to baseline) and one single-day noise spike (one random day at `0.09`, otherwise healthy).
4. Apply your calibrated threshold with a `for`-style sustained-window rule (e.g., "alert only if 3+ consecutive days exceed the threshold") and confirm it fires during the 5-day regression but does *not* fire on the single-day noise spike.
5. Repeat using a naive, non-calibrated threshold picked "by intuition" (e.g., a flat 0.10) with no sustained-window requirement, and compare false-positive/false-negative behavior between the two approaches.

**Done looks like:**
- Your calibrated + sustained-window approach fires exactly once, spanning (a portion of) the 5-day regression, and does not fire on the single-day spike.
- The naive approach either misses the regression, false-alarms on the spike, or both — and you can articulate why, connecting back to §3.4's alert-fatigue argument.
- A one-paragraph writeup comparing the two approaches as if explaining the choice to a skeptical engineering manager who thinks "just alert if refusal rate > 10%" is good enough.

---

## Exercise 7 (Advanced) — End-to-End Drift Pipeline with Slack Alerting (Mocked)

**Goal:** Assemble everything so far — input drift, behavioral drift, decision matrix, thresholds — into one runnable pipeline with alerting, matching the architecture diagram in §3.4.

**Task:**
1. Using your work from Exercises 1-6, build a single script/module `drift_pipeline.py` that: (a) computes input-drift PSI/KS on a tabular feature set, (b) computes behavioral metrics (response length z-score, refusal rate) on a response log, (c) computes or mocks a quality/eval score against a baseline, (d) runs `evaluate_thresholds` to produce a list of alerts, (e) runs `decide_action` via a `DriftEvidence` object derived from the same underlying signals, (f) calls a **mocked** `send_slack_alert` (do not hit a real webhook — write a version that just prints/logs the payload it *would* send, or point it at a local `requests-mock`/`responses` test double).
2. Run the pipeline against three scenarios: (i) everything healthy, (ii) input drift + quality drop, (iii) behavioral drift only with a recent prompt change flagged.
3. Confirm the mocked Slack payload and the decision-matrix action are consistent with each other for all three scenarios (e.g., scenario (ii) should both alert *and* recommend "retrain").

**Done looks like:**
- One command runs the full pipeline end to end against all three scenarios and prints, for each: the raw signals, the alert list, the recommended action, and the mock Slack message text.
- Scenario (i) produces no alerts and an action of `monitor_only` / no action.
- Scenarios (ii) and (iii) each produce an alert whose severity/content is consistent with the decision-matrix action chosen for that scenario.
- Code is organized into functions/classes reusable by Exercise 8, not one monolithic script.

---

## Exercise 8 (Advanced/Capstone) — Full Incident Runbook Simulation with Documentation Artifact

**Goal:** Exercise the entire module end to end: a drift alert fires, and you walk it through the five-step runbook from §3.6, producing a real documentation artifact at the end — the step most teams skip under pressure.

**Task:**
1. Starting from Exercise 7's pipeline, wrap the "behavioral drift only, recent prompt change" scenario (or a new scenario of your choosing) in the `DriftRunbook` class from §3.6.
2. Walk through all five steps explicitly: `validate` (write a `recompute_fn` that actually re-derives the drift signal from a fresh simulated data pull, not just `lambda: True`), `assess_impact`, `review_recent_changes` (pull from a small mock "deploy log" you construct — a list of dicts with `timestamp`, `change_type`, `description`), `act`, and `document`.
3. Have `document` write the resulting `IncidentRecord` to a JSON file (`incidents/drift-<timestamp>.json`) rather than just printing it — this is your durable documentation artifact.
4. Write a second, separate scenario where `validate` returns `False` (the alert doesn't reproduce on a fresh pull) and confirm the runbook correctly short-circuits to "closed as noise" without proceeding through steps 2-4.
5. Optional stretch: add a `follow_up_check_at` re-verification step that, given a later "next window" data pull, checks whether the chosen action (e.g., rollback) actually resolved the original drift signal, and appends that outcome to the same incident JSON file.

**Done looks like:**
- Two incident JSON files exist on disk: one representing a fully-worked real incident (all 5 steps, ending in a specific action and rationale) and one representing a false-positive correctly closed at step 1.
- The JSON schema is consistent and includes, at minimum: `alert_id`, `signals_fired`, `validated_real`, `business_impact_confirmed`, `recent_changes_found`, `action_taken`, `rationale`, `owner`, timestamps.
- You can explain, referencing a specific field in your JSON, how a future on-call engineer investigating a similar alert 3 months from now would use this record to move faster than you did.
- (If attempted) the follow-up re-verification correctly reports whether the chosen action resolved the drift in the simulated next window, closing the loop the tutorial's architecture diagram describes.
