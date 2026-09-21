# Module 03 — Quiz: Versioning, Registries, and Rollback

Test your understanding of `tutorial.md` and `architecture.md`. Mix of multiple-choice (MC) and short-answer (SA) questions. Try to answer without looking back at the tutorial first — then check the answer key at the end.

---

**Q1 (MC).** In an ML/LLM production system, why is versioning only the model version usually insufficient to explain a behavior change?

A. Model version numbers are unreliable because MLflow doesn't enforce uniqueness
B. Production behavior is a function of model, prompt, and dataset version together, and any of the three can change independently
C. Model versions don't capture hyperparameters
D. It is sufficient — prompt and dataset changes are application-code concerns, not versioning concerns

---

**Q2 (MC).** Under the current (2026) MLflow Model Registry API, what is the correct way to mark a model version as the one actively serving production traffic?

A. `client.transition_model_version_stage(name, version, stage="Production")`
B. `client.set_registered_model_alias(name, "champion", version)`
C. Rename the model version to include the word "production"
D. Set the model's `stage` tag to `"prod"` — tags are what MLflow uses to gate serving

---

**Q3 (SA).** Explain, in 2-3 sentences, why MLflow deprecated the fixed Stages (`None`/`Staging`/`Production`/`Archived`) model in favor of aliases and tags. What operational problem does the old fixed-enum approach create at scale?

---

**Q4 (MC).** A team's promotion policy only requires that a challenger model's accuracy be ≥ 0.90 in isolation. What class of regression does this policy fail to catch?

A. A challenger that is 0.91 accurate but is a *worse* model than the current 0.95-accurate champion
B. A challenger with a schema mismatch
C. A challenger that hasn't been registered yet
D. A challenger with a missing signature

---

**Q5 (SA).** Write (in pseudocode or real Python) a one-line boolean expression for a "relative regression guard" that fails a challenger if it is more than 1% relatively worse than the current champion's accuracy. Define your variables.

---

**Q6 (MC).** During an incident, accuracy drops from 0.89 to 0.81 but all infrastructure health checks (uptime, latency, error rate) are green. What does this scenario primarily illustrate?

A. The monitoring system has a bug and should be ignored
B. Infra health checks are not sufficient signals for ML/LLM systems — business/quality metrics must be first-class on-call signals
C. This can only happen if the model was never evaluated before deployment
D. Rollback is unnecessary if infra is healthy

---

**Q7 (MC).** Why does this module insist on rolling back the *entire* deployment triple (model + prompt + dataset) rather than just the model, during an incident?

A. Because MLflow's API only supports rolling back all three at once
B. Because a regression can arise from the interaction between components, and reverting only one leg may leave the actual fault untouched
C. Because prompts and datasets don't have version numbers, so they must be reset along with the model as a matter of convenience
D. Because canary releases require all three to move together

---

**Q8 (SA).** Define "last known good" (LKG) state, and explain why "the immediately preceding release" is *not* a safe substitute for it. Give a concrete scenario where they would differ.

---

**Q9 (MC).** What is the primary purpose of a canary release in an ML/LLM deployment context, specifically?

A. To reduce cloud infrastructure costs during rollout
B. To bound the blast radius of a new release while observing quality/latency/stability on real production traffic that offline evaluation cannot fully simulate
C. To satisfy a compliance requirement that all releases be gradual
D. To allow A/B testing of UI changes only

---

**Q10 (MC).** A regulated fintech company is deploying a new credit-risk model. According to the module's guidance on automation vs. human sign-off, what is the correct promotion setup?

A. Fully automated promotion with no human involved, since automation is always safer
B. A human approves every promotion manually, with no automated checks, since this is a sensitive domain
C. An automated gate acts as the objective floor (must pass), plus a required human sign-off on top of it — never a human gate with no automated floor underneath
D. Promotion decisions should be made by whichever engineer is on-call that week

---

**Q11 (SA).** List the four fields that a rollback audit event should record at minimum, and explain why omitting any one of them weakens a postmortem investigation.

---

**Q12 (MC).** Which of the following is the *strongest* current (2026) reason to choose MLflow over Weights & Biases or Comet for a team building an LLM-based product specifically (not classic ML)?

A. MLflow is always cheaper
B. MLflow 3 has a native GenAI Prompt Registry, unifying model and prompt versioning in one platform
C. W&B and Comet cannot log metrics at all
D. MLflow has a nicer UI than both competitors

---

**Q13 (MC).** Uber's Michelangelo platform reportedly serves thousands of models concurrently. According to the reasoning in this module, why would a fixed-stage (`Staging`/`Production`/`Archived`) registry model be a poor fit at that scale?

A. Fixed stages are slower to query than aliases at the database level
B. A small fixed enum cannot represent "which of thousands of concurrently-served model variants, across many markets/experiment arms, is live where" — flexible tags/aliases scale to that cardinality naturally
C. Uber does not use MLflow, so the question is moot
D. Fixed stages only support a single model at a time, system-wide

---

**Q14 (SA).** In the `release.yaml` pattern, what is the purpose of the `compatibility` block (e.g. `min_dataset_major`, `required_prompt_major`)? Describe a real failure it is designed to prevent.

---

**Q15 (MC).** Why does the tutorial recommend that serving code resolve `models:/fraud-detector@champion` at load time rather than hardcoding a version number like `models:/fraud-detector/18`?

A. Hardcoded version numbers are not supported by MLflow's API
B. It makes promotion and rollback atomic alias repoints that require no redeploy of serving code
C. It is required for MLflow to compute accuracy metrics
D. Aliases load faster than version numbers due to caching

---

## Answer Key

**Q1.** B — Production behavior depends on the full deployment triple; tracking only the model leaves prompt- and dataset-driven regressions unexplainable.

**Q2.** B — `set_registered_model_alias` is the current (post-MLflow-2.9) pattern. A is the deprecated stage-transition API and should not be used in new code.

**Q3.** Model answer: Fixed stages impose a single global enum (one "Production" slot per model name), which doesn't scale to teams running many concurrent variants (per-market, per-experiment-arm, shadow/canary) and forces awkward workarounds. Aliases are arbitrary named pointers you define yourself, so an org can have as many meaningful environment/role names as it needs, and multiple models can each have their own `@champion`, `@shadow`, `@canary`, etc., without fighting a fixed enum.

**Q4.** A — An absolute floor alone doesn't catch the case where the new model is fine in isolation but strictly worse than what's already live; that requires a relative regression guard against the current champion.

**Q5.** Example: `regression_ok = (challenger_acc - champion_acc) / max(champion_acc, 1e-9) >= -0.01`, where `challenger_acc` is the candidate's metric and `champion_acc` is the current production model's metric (guard against division by zero when there's no prior champion).

**Q6.** B — This is the module's central point about ML/LLM on-call: infra can be perfectly healthy while a business/quality metric has silently collapsed, so quality dashboards must be first-class alerting signals, not an afterthought.

**Q7.** B — Regressions can come from the *interaction* between legs (e.g. a new prompt behaving badly against a new dataset's distribution), so a partial (model-only) rollback may leave the true fault live.

**Q8.** LKG is the most recent deployment tuple that was validated in production and did not trigger any rollback criteria — not merely "whatever was deployed right before this one." They differ whenever the immediately preceding release was itself bad (e.g. two consecutive bad releases: r118 bad, r117 also bad, r116 good — naively reverting to r117 would still be broken; the LKG query must specifically filter for `status = GOOD` and skip over r117).

**Q9.** B — Canaries bound blast radius and surface regressions (drift, edge cases, prompt/data interaction effects) that only appear on real production input distributions, which offline eval sets cannot fully capture.

**Q10.** C — High-stakes/regulated domains need the automated objective floor *plus* a human sign-off layered on top — never a human-only gate with no automated floor, and never fully automated with zero human check in genuinely high-stakes domains.

**Q11.** At minimum: reason (why the rollback was triggered), target/restored versions (the full triple), timestamp, and operator (who/what authorized it). Omitting any one weakens a postmortem: no reason means future investigators can't tell what evidence justified the action; no versions means you can't know what state was restored to; no timestamp breaks correlation with monitoring data; no operator removes accountability and the ability to follow up with whoever authorized it.

**Q12.** B — MLflow 3's native GenAI Prompt Registry closes the model+prompt versioning gap in a single platform, which is a genuine, current differentiator for LLM-centric teams (as opposed to bolting a separate prompt tool like PromptLayer/LangSmith onto a model-only registry).

**Q13.** B — A single fixed "Production" slot per model doesn't represent the cardinality of thousands of concurrently-served variants across markets/experiments; tag/alias-based organization scales to that naturally because you're not constrained to one slot.

**Q14.** The `compatibility` block declares explicit constraints (e.g. minimum dataset schema major version a model requires) so that CI/deployment tooling can refuse to deploy an unsafe combination — e.g. preventing a model that depends on a v5+ dataset schema from being deployed against a v4 dataset, which would otherwise silently break or produce garbage predictions.

**Q15.** B — Resolving an alias at load time means promotion and rollback are both just a repoint of that alias in the registry; no redeploy of the serving binary/container is needed either way, which is what makes both operations fast and safe under incident pressure.
