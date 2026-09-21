# MLOps and Model Deployment — Scenario-Based Q&A

**Situation:** Your model scored 94% accuracy in offline evaluation, but after deployment, product metrics (conversion, engagement) haven't moved, and support tickets suggest the model is making poor calls in production. What would you do and why?

Model answer: Treat the offline score as a hypothesis, not proof. First check for train/serve skew — confirm the exact same feature computation logic runs in production as in training (a classic silent failure). Then check whether the offline eval set actually represents current production traffic, since data collected months ago can silently stop matching reality (temporal leakage or drift). Pull a sample of live predictions with their inputs and manually audit them against what a human would decide. Finally, verify the offline metric itself is the right proxy for the business outcome — a model can be accurate on its label definition while that label doesn't actually track what the business cares about.

---

**Situation:** Your monitoring shows the input feature distributions have drifted meaningfully from training data, but you don't yet have new labels to confirm whether accuracy has actually dropped. Do you retrain now, and why or why not?

Model answer: Data drift alone doesn't guarantee performance degradation — the input-output relationship might be unchanged even though inputs shifted. Before committing to a costly retrain, check secondary signals: has the prediction distribution shifted in a way that looks suspicious (e.g., a class suddenly predicted far more or less often), and are there any proxy/delayed labels or partial ground truth available (e.g., short-term outcomes if the true label takes 30 days to arrive)? If the drift is large and sustained and proxy signals suggest degradation, retrain on a rolling recent-data window; if it's a temporary blip (e.g., a one-day traffic anomaly), monitor rather than react immediately to avoid retraining on noise.

---

**Situation:** You're asked to add a new production model without disrupting the currently stable one, and leadership is risk-averse after a past bad rollout caused a revenue-impacting incident. What would you do and why?

Model answer: Use a staged rollout: first shadow-deploy the new model so it scores real traffic without affecting any user, and compare its predictions against the current model's offline. Once it looks healthy, canary it to a small traffic percentage with real user impact and tight monitoring (latency, error rate, prediction distribution, and any fast business-metric proxy). Only after the canary holds up for a defined period do you A/B test at a larger split to get a statistically confident read on business impact, then ramp to 100%. Keep the rollback plan (previous model version, one command away) ready at every stage.

---

**Situation:** A stakeholder asks why you need a feature store when the data science team already computes features fine in their training notebooks. What would you do and why?

Model answer: Explain train/serve skew concretely: a notebook computing "average purchase value in the last 30 days" can subtly differ from however the real-time serving system computes the same feature (different window boundaries, different null-handling, different timezone). A feature store forces both training and serving to call the same feature definition, eliminating an entire class of production bugs that are notoriously hard to detect because the model still runs and returns plausible-looking predictions — it just gets a systematically wrong signal. Cite this as one of the most common root causes of "great offline, bad in production" incidents.

---

**Situation:** Your model's average latency looks fine on the dashboard (p50 well within SLA), but a subset of users are complaining about slow responses. What would you do and why?

Model answer: Look past the average — check p95/p99 latency, not just p50, since tail latency is what individual unhappy users actually experience. Segment latency by request characteristics (payload size, user region, cache hit/miss, cold-start vs warm instance) to find where the tail comes from. Common culprits: autoscaling cold starts, a specific feature-lookup path that's slow for a subset of users, or GPU batching queuing effects under load. Fix the specific bottleneck rather than tuning the whole system for a marginal average improvement.

---

**Situation:** Your team wants to move from monthly batch retraining to continuous training, but you're worried about destabilizing production with frequent model changes. What would you do and why?

Model answer: Continuous training should not mean continuous unconstrained deployment — separate "retrain" from "promote to production." Automate retraining on a schedule or trigger (e.g., detected drift), but gate promotion behind the same CI checks used for any model update: evaluation against a fixed holdout, comparison against current production metrics, and a canary rollout before full traffic. This gets the freshness benefit of continuous training without removing the safety checks that prevent a bad retrain from reaching all users.

---

**Situation:** After a model rollback due to a serious regression, leadership asks for a plan so this doesn't happen again. What would you do and why?

Model answer: Do a blameless post-incident review identifying exactly which check should have caught the issue (missing data validation, missing slice-level eval, no canary stage, no automatic rollback trigger) and close that specific gap rather than adding generic process. Concretely: add the failing case to the permanent regression eval set, add an automated rollback trigger tied to the signal that would have caught it early (e.g., prediction-distribution bounds), and ensure the rollback path is a rehearsed one-command action, not a manual scramble.

---

**Situation:** You're serving an LLM-based feature and inference cost has become a significant line item as usage scales. What would you do and why?

Model answer: Profile where cost actually concentrates — model size, prompt length (including any repeated context/history), and request volume. Apply the standard levers in order of effort: caching repeated/common queries, reducing prompt/context size, batching requests, and only then moving to model-level optimization like quantization or distillation to a smaller model, or routing simple requests to a cheaper smaller model and only escalating hard cases to the larger one. Validate that any optimization doesn't silently degrade quality by re-running it against the evaluation set, not just checking cost dropped.

---

**Situation:** Design the deployment strategy for a churn-prediction model: how would you decide it's safe to fully roll out to all customers?

Model answer: Frame it as a staged pipeline: validate offline on a proper time-based holdout (not random k-fold, since churn is time-ordered and leakage-prone), shadow-deploy to compare live predictions against the current heuristic/model with zero user impact, then canary to a small customer segment where you can measure whether flagged "at risk" customers actually respond to retention interventions better than a control group, and only then roll out fully with an A/B test comparing overall retention rate. Keep the rollback plan defined and keep monitoring prediction distribution post-full-rollout, since customer behavior (and thus churn patterns) shifts over time.
