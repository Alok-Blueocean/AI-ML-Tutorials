# Drift Detection and Retraining Decisions — Scenario-Based Q&A

**Situation:** A PSI drift alert fires on your model's input features, and an engineer immediately kicks off a full retraining job before looking at anything else. What would you do and why?

Model answer: Stop the reflexive retrain and treat the alert as an instruction to gather evidence, not an instruction to retrain. Input drift alone is not sufficient grounds — a model can be robust to a given shift, so drift-without-quality-impact is common and shouldn't trigger automatic retraining. Check whether quality drift (an eval/judge score or downstream KPI) has actually dropped alongside the input drift, and check the deploy log for a recent prompt/model/pipeline change that might better explain the shift. Only retrain when input drift is large, sustained across multiple windows, and correlated with a confirmed quality drop — the strongest signal the decision matrix defines.

---

**Situation:** Your LLM-based support assistant's refusal rate jumps from a steady 1% to 12% overnight, but the distribution of incoming user queries looks statistically unchanged. What would you do and why?

Model answer: Classify this precisely as behavioral drift with no corresponding input drift, which points strongly toward a change in the model, prompt, or pipeline rather than a change in the world. Check the deploy/prompt registry for anything that shipped in the same window — a system-prompt edit, a silent model-provider upgrade behind an API alias, or a guardrail threshold change are the most common causes of exactly this pattern. If a recent change is found, roll it back first and verify the refusal rate recovers before doing anything else; retraining would not fix a prompt or pipeline regression and is the wrong tool here even though "quality" nominally changed.

---

**Situation:** A stakeholder asks why the team hasn't just set up one blended "drift score" that combines input, output, and quality signals into a single dashboard number, to simplify monitoring. What would you do and why?

Model answer: Push back on collapsing the three signal types — the entire operational value of separating data drift, behavioral drift, and quality drift is that the correct response depends on which one (or which combination) fired, and blending them into one number destroys exactly the distinction that tells an on-call engineer what to do. Keep three separate tracked signals and, if dashboard clutter is the real complaint, address that with better dashboard layout (grouped panels, a single "drift status" summary row above the detailed signals) rather than mathematically merging signals that need to stay distinguishable to be actionable.

---

**Situation:** Your team copies Evidently's default PSI thresholds (0.1 warning, 0.25 critical) directly into production alerting for a feature that naturally oscillates a lot week to week, and the on-call channel is immediately flooded with alerts. What would you do and why?

Model answer: Diagnose this as a threshold-calibration failure, not a tool failure — PSI's 0.1/0.25 bands are an industry starting heuristic, not a law, and copying them verbatim without checking a feature's own historical noise floor is one of the most common drift-monitoring mistakes. Recompute the metric retrospectively over a known-healthy historical period, find its natural week-to-week noise floor, and set the alert threshold a small number of standard deviations above that floor rather than at the generic default. Also require a sustained-breach window (a `for` duration) instead of alerting on a single noisy data point, and schedule a periodic (e.g., monthly) review of false-positive rate to keep the threshold honest as traffic patterns evolve.

---

**Situation:** A data scientist computes PSI bin edges from the current production window instead of the frozen baseline, and the drift dashboard has been reporting suspiciously low PSI values for months. What would you do and why?

Model answer: Recognize this as a specific, well-known correctness bug: computing bin edges from the current window instead of the baseline silently understates drift, because the current data is by construction evenly distributed across bins built from itself. Fix the pipeline to always derive bin edges from the frozen baseline (captured at the last validated deploy), never from the rolling current window, and re-run the drift calculation over the historical period to find out how much real drift was masked by this bug. Treat this as reason to also audit the baseline itself — check it wasn't accidentally set up as a rolling window (a related "baseline creep" bug), which would have the same effect of silently raising the detection threshold over time.

---

**Situation:** Leadership wants to know, in one meeting, whether last month's accuracy drop was caused by "the world changing" or "the model getting worse," and wants a retraining decision on the spot. What would you do and why?

Model answer: Decline to give a snap answer and instead walk through the decision matrix live: pull the input drift signal (did the incoming data distribution actually shift?), the quality drift signal (is there a confirmed accuracy/judge-score drop against ground truth or a trusted golden set?), and the deploy log (was there a recent prompt, model, or pipeline change?). If input drift is high and quality is confirmed dropping with no recent change nearby, that supports retraining. If quality dropped with input and behavioral signals both looking stable, the more likely culprit is the measurement pipeline itself (labeling lag, judge miscalibration, a stale golden set) and needs investigating before any retrain — a decision made without checking evidence risks wasting a retraining cycle on the wrong fix.

---

**Situation:** Your only ground-truth accuracy signal for a fraud-detection model comes from confirmed chargebacks, which arrive 30-60 days after a prediction — by the time you'd notice quality drift the traditional way, a month of bad decisions has already happened. What would you do and why?

Model answer: This is precisely the labeled-latency problem that label-free performance estimation exists to solve. Adopt a tool like NannyML's CBPE, which learns the relationship between prediction confidence and correctness on a labeled reference period, then estimates current performance on unlabeled recent predictions without waiting for chargebacks to arrive. Run this alongside — not instead of — the eventual ground-truth accuracy check once chargebacks do arrive, so the label-free estimate can be validated and recalibrated periodically against real outcomes rather than trusted blindly forever.

---

**Situation:** A KS test on your model's input feature distribution reports a p-value of 0.0001 on a batch of 500,000 production requests, and an engineer treats this as proof of severe drift requiring immediate action. What would you do and why?

Model answer: Push back on reading the p-value alone — with production-scale sample sizes, KS will report a statistically significant p-value for even a trivially small, operationally meaningless shift, which is the single most common KS-test mistake in production drift monitoring. Look at the KS statistic's actual magnitude (the maximum gap between the two empirical CDFs) alongside a practically-meaningful effect-size threshold, not just whether the p-value cleared an arbitrary bar. If the statistic itself is tiny despite the low p-value, this is very likely statistical significance without practical significance, and doesn't warrant the same response as a large, sustained shift.

---

**Situation:** Your RAG chatbot's input queries are free text, and someone asks why you can't just run the same PSI test you use on tabular features to check for query drift. What would you do and why?

Model answer: Explain that PSI and KS are built for binned numeric or categorical data, and there's no natural "bin" for a user's free-text query — applying them directly to raw text doesn't work. The correct approach is embedding-space drift: embed a baseline sample and a current sample of queries with the same embedding model, then compare either the centroid (cosine) distance between the two samples' mean vectors, or a dispersion/spread delta to catch cases where the average topic stayed put but the variety of topics being asked about widened. Note the limitation up front — centroid distance alone will miss a shift that only affects spread, so track both together rather than relying on centroid distance as a complete drift signal.

---

**Situation:** A model that predicts holiday retail demand shows a large PSI spike on order-volume-related features every December, and a new on-call engineer pages the team at 2 a.m. convinced something is broken. What would you do and why?

Model answer: Recognize this as the textbook example of expected, benign covariate shift — a retail demand model seeing higher order volumes in December is completely expected and requires no action, whereas the same magnitude of shift in July would be a legitimate reason to investigate. This is exactly why raw drift magnitude alone shouldn't auto-page anyone: pair drift detection with either a seasonally-aware baseline/threshold, or an explicit runbook note telling on-call "expected seasonal PSI spike, no action needed unless quality drift also fires," so a predictable, harmless pattern doesn't keep generating false alarms and eroding trust in real ones.

---

**Situation:** Three weeks after a prompt template change shipped, your behavioral-drift dashboard shows response length has crept up steadily, but nobody connected it to the deploy because the change happened gradually, not as a step function. What would you do and why?

Model answer: Treat this as a monitoring-cadence gap rather than an unpredictable, purely gradual drift — cross-reference the trend's start date precisely against the deploy log rather than assuming a gradual-looking trend can't have a discrete cause; a prompt change's effect on response length can ramp in as traffic mix shifts even if the underlying cause was a single deploy. Going forward, annotate drift dashboards with deploy events (prompt changes, model version bumps, pipeline config changes) as vertical markers on the same timeline as the metric trend, so a correlation like this is visually obvious rather than requiring someone to manually cross-reference two separate logs after the fact.

---

**Situation:** Your team wants to skip building a dedicated drift dashboard and just have engineers manually check distributions "when something feels off." What would you do and why?

Model answer: Push back — this defeats the entire point of drift monitoring, which exists precisely because a drifted model fails silently: it returns HTTP 200 with fluent, confident output, so there's no natural trigger for "something feels off" until a customer notices. Build the dashboard and alerting together, not as an afterthought — a dashboard nobody checks and an alert with no dashboard to triage against are both incomplete halves of the same system. At minimum, instrument PSI per key feature (or embedding-drift score for text), response length/refusal-rate trends, and the evaluation score trend, with alerting tuned against each metric's own historical noise floor.

---

**Situation:** An alerting rule fires because the refusal rate crossed 15% for a single 5-minute window during a brief traffic spike, paging an engineer who found nothing actually wrong by the time they investigated. What would you do and why?

Model answer: Diagnose this as a missing sustained-breach requirement — a single noisy window shouldn't page anyone, exactly as with any other production alerting discipline. Add a `for` duration requirement (e.g., sustained breach for 15+ minutes) so a transient spike during unusual but harmless traffic doesn't trigger a page, while a genuine, sustained shift still does. Also check whether the underlying metric's noise floor was ever calibrated for this traffic pattern — a threshold that's fine under steady load may need a higher bar or a longer sustain window specifically around known traffic-spike periods.

---

**Situation:** A retraining job runs automatically every time drift is detected, and the team notices that a few of the resulting retrained models have actually performed worse than the model they replaced. What would you do and why?

Model answer: Point out the actual risk automatic, reflexive retraining creates — retraining is expensive and, worse, retraining on a small, recent, or unrepresentative window can produce a genuinely worse model, especially if the "drifted" window itself was anomalous (a temporary spike, a data-quality issue) rather than a stable new normal. Replace blind auto-retrain-on-drift with the decision matrix: retrain only when input drift is large and sustained and correlated with a confirmed quality drop. Also require every retrained model to pass the same evaluation gate as any new model version (Module 03/13's registry and promotion workflow) before replacing the production model, rather than deploying a freshly retrained model automatically just because retraining completed.

---

**Situation:** Your team is deciding between Evidently AI, NannyML, and Arize Phoenix and wants to pick just one to standardize on for all monitoring needs. What would you do and why?

Model answer: Explain that no single tool owns "drift monitoring" end to end, and forcing one tool to cover every need usually means using it well outside its actual strength. Evidently is strong for fast, batch-style tabular drift reports and CI-pipeline drift gates; NannyML uniquely solves label-free performance estimation when ground truth is slow or sparse; Arize Phoenix is the strongest fit specifically for LLM/embedding-space drift and trace-level debugging. Recommend composing two or three of these around the actual gaps in your pipeline — for example, Evidently for structured feature drift plus Phoenix for embedding/query drift — rather than picking one tool and stretching it to cover use cases it wasn't built for.

---

**Situation:** A monthly review of the drift-alerting system finds that half of all fired alerts over the past quarter were false positives that engineers learned to dismiss without investigating. What would you do and why?

Model answer: Name this explicitly as alert fatigue and treat it as a monitoring-system defect that has already started to erode its own value — engineers who've learned to dismiss half of all alerts will also dismiss the real ones. Run the threshold-calibration procedure properly: recompute each alerting metric's noise floor over a genuinely healthy historical period, retune thresholds to sit a defensible number of standard deviations above that floor, and add or verify sustained-breach windows. Schedule this false-positive/false-negative review as a recurring practice (monthly is reasonable) rather than a one-time cleanup, since traffic patterns evolve and yesterday's well-tuned threshold can silently become today's noisy one.
