# Module 03 — Scenario-Based Q&A

**Situation:** At 2 a.m., a production accuracy alert fires — 0.89 down to 0.81 — while every infrastructure health check is green. The on-call engineer's instinct is to immediately roll back just the model to the previous registry version. What would you do and why?

Model answer: Stop the model-only rollback before it happens. First confirm the drop is real (rule out a monitoring blip or a data-pipeline gap), then query the lineage store for the exact currently-deployed release tuple — model, prompt, and dataset versions together — not just the model version. Identify the last known-good tuple, and restore all three components atomically. The reason this matters concretely: if the actual fault is an interaction between a newly-changed prompt and a newly-refreshed dataset's distribution, rolling back only the model leaves the real cause untouched and the incident may persist or resurface, costing more time than doing the full-tuple restore correctly the first time.

---

**Situation:** A teammate describes their release process as "we track the model version in MLflow, and the prompt lives in a Python string in the serving repo, versioned by git commit." They ask if this is good enough.

Model answer: No — this is the classic "versioning only the model, not the triple" mistake. Git commit history for the prompt string is a form of versioning, but it isn't logged per-inference alongside the model version, so a production regression can't be traced back to "which exact prompt was live for this specific bad output" without archaeology through commit timestamps and deploy logs. Recommend moving the prompt into a proper registry (MLflow 3's GenAI Prompt Registry, or an equivalent) so both model and prompt versions can be logged together on every inference record — closing the gap that makes reconstructing a historical output possible at all.

---

**Situation:** A candidate in an interview describes promoting a model to production by calling `client.transition_model_version_stage(model_name, version, stage="Production")`. How would you respond, and what would you want them to say instead?

Model answer: Flag this as describing a deprecated pattern — MLflow's fixed Stages (`None → Staging → Production → Archived`) have been deprecated since MLflow 2.9 in favor of aliases (`@champion`, `@challenger`) plus tags. Ask a clarifying follow-up rather than failing them outright, since this is a common and easy-to-miss knowledge gap: "what's the current recommended way to do this, and why did it change?" A strong recovery explains that aliases are arbitrary named pointers a team defines itself, which scales better than a fixed enum once you're serving many concurrent model variants (per-market, per-experiment-arm) rather than exactly one "the" production model.

---

**Situation:** A data scientist wants to promote their newly retrained fraud model to production themselves, by manually repointing the `@champion` alias after looking at the offline evaluation dashboard and deciding "it looks good."

Model answer: Block this and explain why: the single most important principle in this module is that promotion must never be manual — not because of distrust of the individual, but because "it looked fine in the demo" substitutes a subjective judgment call for a statistically grounded, reproducible comparison against the current production model. Require an automated promotion gate that checks at minimum an absolute quality floor, a relative regression guard against the current champion, and a latency/cost ceiling, wired into CI so the decision is reviewable and re-runnable — not a manual alias repoint based on eyeballing a dashboard.

---

**Situation:** Your promotion gate currently only checks `challenger_accuracy >= 0.88` as an absolute floor. A postmortem reveals a new model passed this bar but was measurably worse than the model it replaced, and users noticed the regression before your team did.

Model answer: Diagnose the gap as missing a relative regression guard — an absolute floor alone can let a legitimately "okay in isolation" model through even when it's worse than what's already live, which is the most common real-world regression pattern. Add a second check comparing the challenger against the current `@champion`'s own logged metrics (e.g., relative delta ≥ −1%), not just a fixed historical threshold. Both checks matter for different reasons: the absolute floor guards against a system that's objectively bad, and the relative guard specifically prevents shipping a step backward from wherever the bar currently sits.

---

**Situation:** A regulated-industry client (credit decisioning) asks whether your fully automated promotion pipeline is appropriate for their use case, given everything you've built for less-regulated products.

Model answer: Recommend layering a human approval step on top of, not instead of, the automated gate — the module's guidance is explicit that fully automated promotion isn't appropriate for high-stakes regulated domains, but the fix is "automated gate as the floor, plus required human sign-off," never a human gate with no objective floor underneath it. Concretely: keep the absolute-quality, regression, and latency checks fully automated and blocking, and add a mandatory human review step (reviewing the same evidence, not re-deriving it from scratch) before the alias actually repoints to `@champion` — satisfying both the compliance need for human accountability and the engineering need for an objective, non-subjective baseline.

---

**Situation:** An engineer proposes skipping the canary stage entirely for a new prompt release because "the offline eval scores were excellent, better than we've ever seen," and going straight to 100% traffic to move faster.

Model answer: Push back regardless of how strong the offline numbers look — offline evaluation, however excellent, cannot fully simulate adversarial or unusual real-world input distributions, and some failure modes only appear on real production traffic. Recommend the standard canary ramp (e.g., 5% → 25% → 100%) with automatic rollback-to-zero on any threshold breach, framing it as a bounded-blast-radius safety net that catches exactly the category of failure offline eval structurally can't — not as a bureaucratic delay that strong eval numbers should let you skip.

---

**Situation:** Your team is choosing between MLflow, Weights & Biases, and building a custom registry, and a senior engineer argues for W&B "because it's what I personally know best and it has the nicest dashboards."

Model answer: Redirect the decision criteria — registry choice should follow the team's operating model (release cadence, governance/approval load, centralized vs. decentralized deployment, required lineage depth, existing tooling gravity), not individual tool familiarity or UI polish alone. Ask specifically whether the team needs prompt versioning alongside model versioning (if this is an LLM-facing team, MLflow 3's native GenAI Prompt Registry closes a real gap W&B doesn't natively cover as first-class), and whether the team is already deeply invested in W&B for experiment tracking (a real, legitimate factor, just not "I like it best"). If there's no existing W&B investment and prompt/model unification matters, recommend MLflow as the stronger open-source default.

---

**Situation:** A platform team at a company running roughly 5,000 concurrently-served models is debating whether to keep using a small number of MLflow model-version tags loosely, or invest in building "Gallery," a fully custom internal registry, citing Uber's Michelangelo as precedent.

Model answer: Use the decision tree's scale/compliance threshold directly: at Uber's Michelangelo-class scale (thousands of concurrently served models, deep coupling to a bespoke internal deployment platform), a custom registry can be justified because off-the-shelf tools' generic lineage/audit/access-control features don't scale to that level of deployment-specific integration. But caution that this is a high bar, not a default — building a custom registry means budgeting real engineering time for lineage, audit, and access control, which are easy to get subtly wrong, and most teams well below thousands-of-models scale are better served defaulting to MLflow (self-hosted or Databricks-managed with Unity Catalog) than reinventing this.

---

**Situation:** During a rollback drill (not a real incident), the team discovers their "last known good" logic simply decrements the release ID by one — it assumes the immediately preceding release was fine.

Model answer: Flag this as the "conflating last previous version with last known good" mistake explicitly named in this module — the release immediately before the current bad one might itself have been a bad release that hadn't been caught yet, or might share the same root cause. Fix the rollback logic to query the lineage store for the most recent release explicitly tagged GOOD (validated in production, not simply "not yet known to be bad"), not to decrement a counter. This distinction matters most exactly when it's least obvious — during a real incident, under time pressure, when a quick "previous version" assumption is most tempting and most dangerous.

---

**Situation:** A new model version passes its promotion gate and gets registered, but a teammate points out the registered version has no tags beyond a bare version number — no run ID, no dataset version, no evaluation metrics attached.

Model answer: Call this out as "registry as pure binary storage" — treating the registry as just a place model files live, rather than populating it with the metadata (metrics, run reference, data version, environment) that makes it usable during an actual incident. Fix by tagging every registered version with, at minimum, its semantic version, the dataset version it trained on, its run ID (for full lineage back to hyperparameters and training data), and its key evaluation metrics — otherwise, six months from now, an incident responder looking at "version 14" has a number with no way to answer "was this good, and what produced it."

---

**Situation:** A new model requires a dataset schema change (a new required feature column), but the release pipeline doesn't check this — and a well-intentioned rollback later redeploys this new model version against an old dataset snapshot that predates the schema change, silently breaking inference.

Model answer: Identify the missing piece as compatibility rules — the release manifest (`release.yaml`) should declare explicit compatibility constraints (e.g., `min_dataset_major: 5`) so a model requiring a new schema structurally cannot be deployed, whether via forward release or rollback, against an incompatible dataset version. Add this check to both the promotion gate and the rollback engine, since rollback is exactly the path most likely to accidentally reintroduce an old, now-incompatible combination if compatibility isn't enforced symmetrically in both directions.

---

**Situation:** After a full-tuple rollback during an incident, nobody records why the rollback happened, who authorized it, or which specific versions were restored — three weeks later, a postmortem is being reconstructed entirely from memory and scattered Slack messages.

Model answer: Diagnose this as "no audit trail on rollback events" — every rollback should be an explicitly recorded event: reason, target versions restored, timestamp, and operator, written to the lineage store as its own auditable record, not just an implicit consequence of alias repoints. Fix by making the rollback script itself write this record as a mandatory final step (as shown in the module's full-tuple rollback code, which logs a dedicated `rollback-event` run with structured tags) — the goal is that any future postmortem is a lineage-store query, not an act of memory reconstruction.

---

**Situation:** A model, prompt, and dataset triple all changed in the same release (a routine, non-incident deployment about to ship), and a reviewer asks whether it's safer to ship them as three separate, independently-gated releases instead of one combined release.

Model answer: Generally recommend against splitting them for this case unless there's a specific reason to isolate one leg's effect — the deployment triple concept exists because production behavior depends on the exact combination of these three components together, and testing/promoting them as one coherent, evaluated bundle (with the shared `release.yaml` manifest) is what makes the eventual regression-guard and rollback logic deterministic. If there's a real need to isolate a specific change's individual effect (e.g., a risky new prompt being tested independently of a routine dataset refresh), that's better served by canary-ing the risky component at low traffic first, not by weakening the "always deploy and log as a triple" discipline.

---

**Situation:** Your serving code currently hardcodes `model_uri = "models:/fraud-detector/18"` (a fixed version number) rather than resolving an alias. A teammate proposes this is actually safer because "it's explicit about exactly what's running."

Model answer: Push back — hardcoding a version number means every promotion or rollback requires a code change and a redeploy of the serving code itself, turning what should be an atomic, fast alias repoint into a slower, riskier release. Recommend resolving `models:/fraud-detector@champion` at load time instead: promotion becomes repointing the alias with no serving-code redeploy required, and rollback becomes equally fast and low-risk — "explicit about what's running" is better achieved by logging the resolved version per-inference (the deployment-triple logging pattern) than by hardcoding it into the serving code path itself.

---

**Situation:** A junior engineer asks why this module insists on canary releases even for changes that passed a rigorous offline evaluation suite with strong metrics, calling it "redundant caution."

Model answer: Explain that canaries and offline evaluation catch structurally different failure classes, so one doesn't substitute for the other: offline evaluation validates against a fixed held-out or historical dataset, which by construction cannot capture adversarial or unusual real-world input patterns the model hasn't seen represented in that set. A canary exposes the release to a small, bounded percentage of real production traffic and observes live quality/latency/stability metrics before ramping further — it's the safety buffer specifically between "passed offline eval" and "safe at 100% production traffic," not a redundant repeat of the same check.

---

**Situation:** An incident responder, restoring a full-tuple rollback under pressure, wants to skip writing the audit event to save two minutes during an active outage, planning to "log it properly afterward."

Model answer: Recommend against skipping it, but pragmatically — the audit-event write in the module's rollback script is a lightweight, fast operation (an MLflow run with a handful of tags), not a blocking multi-step process, so the actual time saved by skipping it is negligible relative to the risk of it being forgotten once the outage is resolved and attention moves elsewhere. If genuinely every second counts, the audit write should be fire-and-forget (asynchronous, non-blocking to the restore itself) rather than skipped outright — the goal is that the restore's speed and the audit trail's completeness are not actually in tension once the write is engineered correctly.

---

**Situation:** A frontier-lab-style team is deciding how many independently-versioned "legs" their production LLM system actually has, since their system involves a base model, a system prompt, a safety/guardrail configuration, and a few-shot example set backing certain requests.

Model answer: Recommend treating this as at least four independently-versioned legs, not folding them into "the model" and "the prompt" as if that were exhaustive — the safety/guardrail configuration and the few-shot example set are both artifacts that can independently change behavior and independently cause a regression, exactly like the model or the main system prompt can. Extend the deployment-triple concept to whatever the actual number of independently-changing legs is for a given system (it's a tuple, not necessarily always exactly three), and log all of them per-inference — the principle generalizes even when the specific system has more moving parts than the canonical model/prompt/dataset example.

---

**Situation:** Someone on your team asks whether "last known good" should be defined as the most recent release that passed its promotion gate, or the most recent release that has since accumulated enough healthy production traffic to be trusted.

Model answer: Define it as the latter — a release that merely passed its promotion gate (an offline/pre-production check) hasn't yet been validated against real production traffic the way a canary or a meaningfully long healthy production window has. Recommend explicitly tagging a release as GOOD in the lineage store only after it has cleared its canary ramp and run cleanly in production for a defined minimum window, not the instant it passes the gate — this closes a subtle gap where a rollback target could otherwise be a release that technically passed its tests but was never actually proven healthy under real traffic.
