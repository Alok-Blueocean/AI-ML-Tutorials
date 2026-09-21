# Deployment Quality Gates and Release Dashboards — Scenario-Based Q&A

**Situation:** A team has a solid evaluation harness, but a bad prompt change still reached production last week because someone forgot to run it before merging. What would you do and why?

Model answer: Diagnose this as a missing-trigger problem, not a discipline problem — a harness that exists but isn't wired to fire automatically is a manual step, and manual steps get skipped under deadline pressure, precisely when they matter most. Wire evaluation into at least the pull-request and push-to-main trigger points so it runs without anyone remembering to invoke it, and make the check a required status check in the repo's branch protection rules so a merge is structurally blocked, not just discouraged, when it fails. Treat "someone forgot" as evidence the process has a gap, not evidence the person needs a reminder.

---

**Situation:** Your evaluation dashboard clearly showed a 6% drop in composite score two days before a release, and the release still shipped. A postmortem is being written. What's the actual root cause, and what do you fix?

Model answer: The root cause is almost certainly that the evaluation ran and reported but nothing enforced the result — a dashboard showing a regression that nobody was required to act on before merging is a historical record, not a gate. The fix isn't "look at the dashboard more carefully next time," it's converting the check into a fail-fast pipeline step that calls a non-zero exit code and structurally blocks the merge or deploy when a threshold is violated, so the decision doesn't depend on a human noticing a chart in time. Recommend this exact pattern in the postmortem's corrective actions, not just "add a reminder to check the dashboard."

---

**Situation:** An engineer proposes running the full 200-example evaluation suite on every single pull request, including small documentation and config typo fixes, to "never miss a regression." What would you push back on?

Model answer: Push back on cost and cadence mismatch, not on the goal. Running the full, expensive suite on every PR — including changes that can't possibly affect model behavior — burns CI budget and slows iteration without proportional benefit. Recommend a tiered strategy instead: a fast, cheap correctness check (e.g., 50 examples) on every PR to catch obvious regressions quickly, the full evaluation suite gating push-to-main before an artifact is registered, and a scheduled weekly (or daily, for high-risk systems) drift check against real production-sampled traffic. This keeps evaluation affordable enough to run often rather than forcing a binary choice between "run it everywhere, expensively" and "skip it to save cost."

---

**Situation:** Your hallucination-rate check and your composite-quality check are both implemented as relative gates — "must not regress more than 1% versus current production." A new model version passes both, but hallucination rate is now at 4.5%, which is objectively too high for the business. What went wrong with the gate design?

Model answer: The design mistake is using a relative-only gate for a metric that has an absolute, non-negotiable safety floor. Relative gates catch regressions but say nothing about whether the current baseline itself was already unacceptable — if production hallucination rate was already creeping upward release over release, a "no more than 1% worse" gate lets it walk right past a real danger threshold one small step at a time. Add an absolute gate as a hard block specifically for hallucination rate (e.g., must be under 3% regardless of what production currently does), and reserve relative gates for dimensions where "no worse than before" is genuinely the right bar, like general quality trend-tracking.

---

**Situation:** A release manager complains that a candidate release is being blocked by the gate for a formatting/style dimension that scored slightly low, even though correctness and safety are both excellent, and wants the ability to override and ship anyway. How do you respond?

Model answer: Distinguish between hard blocks and everything else before answering. Format-validity and hallucination/safety checks should be designed as hard blocks with zero human-override tolerance, precisely because they're the checks most likely to cause real harm if bypassed under pressure — but a softer style/formatting dimension scoring slightly low, with correctness and safety strong, is a reasonable candidate for a documented human override with sign-off, not a structural block. The actual fix is making sure the gate design already distinguishes these tiers so the override conversation only ever happens on the dimensions where it's actually safe to have it.

---

**Situation:** You're asked to design the Grafana dashboard a release manager will use to decide ship/hold/rollback, and a colleague wants to put twenty metrics on one screen "so nothing is missed." What do you push back on, and what do you build instead?

Model answer: Push back on the wall-of-noise instinct — more panels doesn't mean more signal, it means slower decisions and a higher chance the one panel that matters gets lost. Design the dashboard around the reading order a release manager actually uses: headline release-readiness status (green/yellow/red) first, dimension breakdown second, trend over recent releases third, regression-versus-baseline fourth. Anything that doesn't serve one of those four questions belongs in a secondary, drill-down view, not the primary screen, and panel thresholds/colors should map 1:1 onto the exact same numeric thresholds the automated gate code enforces, so the dashboard and the gate never silently disagree.

---

**Situation:** A hard-block hallucination-rate Grafana alert has been firing every few days for a month, and the on-call engineer has started acknowledging it without investigating because "it's usually nothing." What's the actual problem, and how do you fix it?

Model answer: This is alert fatigue caused by a threshold that's either miscalibrated or paired with no clear action path — a hard-block alert that fires routinely and is routinely dismissed has, in practice, stopped functioning as a hard block at all. Recalibrate the threshold against real historical data so it fires only when the metric is genuinely outside acceptable bounds, and pair the alert with an explicit runbook entry telling the on-call engineer exactly what to check and what action to take, so "acknowledge and move on" stops being the path of least resistance. An alert nobody trusts is worse than no alert, because it creates false confidence that the safety net is working.

---

**Situation:** Your data pipeline updates the RAG corpus nightly, and a bad ingestion run three weeks ago silently introduced a batch of malformed, half-scraped documents into the index. Model code hasn't changed at all, but answer quality has been slowly degrading since. Why did your existing PR and push-to-main gates miss this entirely?

Model answer: PR and push-to-main gates only evaluate code and prompt diffs — they have no visibility into a change to the retrieval corpus itself, which is exactly the gap the data-update trigger exists to close. The fix is adding an evaluation trigger on the data pipeline's own promotion step, so a data-induced regression check runs on every data pipeline run that touches production-facing content, independent of whether any application code changed. Going forward, treat "a code diff caused this" and "the world/data underneath the code changed" as two structurally different regression sources that need two different triggers, not one.

---

**Situation:** A candidate model version passes every quality dimension comfortably but doubles P95 latency and triples per-query cost versus production. The team almost ships it because "the quality score is great." What's the flaw in that reasoning, and what do you do?

Model answer: Point out that production readiness is inherently multi-dimensional, and a single quality pass/fail hides exactly this kind of regression on an orthogonal axis. Add latency and cost as their own gated dimensions with their own thresholds — a release can be more correct and still be unacceptable if it blows the latency SLA or the cost budget — rather than letting a strong quality score implicitly excuse a regression nobody was watching for on a different dimension. Block the release until latency and cost are back within bounds, or make an explicit, documented tradeoff decision with stakeholders who own the SLA and budget, rather than letting the quality dashboard alone drive the call.

---

**Situation:** After a major incident, the postmortem finds that a canary rollout to 5% of traffic did catch the regression early, but nobody was paged, and it took four hours of user complaints before an engineer manually noticed the canary metrics looked off. What structural gap does this expose?

Model answer: The canary evaluation caught the problem, but the pipeline had no automated fail-fast reaction wired to it — canary metrics that require a human to proactively notice them defeat the point of running a canary at all. Fix it by attaching an explicit alert rule to the canary's key metrics (hallucination rate, error rate, composite score) with a threshold and a `for` duration tuned to canary traffic volume, so a regression pages on-call directly instead of waiting for someone to look at a chart, and wire the canary result into the deployment pipeline so a failing canary can automatically halt further rollout rather than continuing to ramp traffic.

---

**Situation:** Your team's frozen evaluation set was built eight months ago, and a new engineer notices that several examples in it reference a product feature that was deprecated and removed two months ago. Should you update the eval set, and if so, how do you avoid breaking score comparability?

Model answer: Yes, update it — a frozen evaluation set testing a removed feature is measuring something that no longer matters and wasting evaluation budget on irrelevant checks, while potentially missing coverage of features that shipped since. But do it as a deliberate versioned change, not a silent edit: create a new version (e.g., "eval-set v2.0"), re-baseline every gate threshold and dashboard trend line against runs on the new version, and keep the old version archived so historical score comparisons remain interpretable as "before/after eval-set v1.0 to v2.0" rather than silently discontinuous. Never edit examples in place under the same version identifier — that breaks the entire premise that score changes reflect model changes rather than test-set drift.

---

**Situation:** A stakeholder asks you in an interview-style question to design a release process for a new LLM-backed customer support feature from scratch. What's your answer?

Model answer: Lay out the three-layer structure end to end: first, wire evaluation triggers at all four points that matter for anything customer-facing — pull request, push-to-main, a scheduled drift check against production-sampled traffic, and a data-update trigger for the underlying knowledge base or corpus. Second, design gates as a mix of absolute hard blocks for non-negotiable dimensions like format validity and hallucination/safety rate, plus relative gates comparing against the current production baseline for general quality trend, wired into the CI/CD pipeline as fail-fast checks with non-zero exit codes rather than warnings. Third, build a release-readiness dashboard read in a fixed order — headline status, dimension breakdown, trend, regression-versus-baseline — with alert rules and thresholds that mirror the gate code exactly, and canary rollout with its own automated fail-fast reaction before full traffic ramp. Close by naming the organizational failure modes this guards against: evaluation that only runs when remembered, evaluation results nobody is required to act on, and single-dimension scores that hide a real regression on cost, latency, or a specific quality dimension.

---

**Situation:** Six months after your quality-gate system launches, engineers start quietly routing around it by merging directly to a hotfix branch that bypasses the standard PR gate "just for urgent fixes," and this has become normal practice. What's the risk, and how do you address it?

Model answer: The risk is that "urgent" has become an unmonitored escape hatch from the exact safety system designed to catch regressions under pressure — which is precisely when a bad change is statistically most likely. Rather than simply banning the hotfix path (which teams will find a way around again under real time pressure), build a lightweight, fast-but-real evaluation tier specifically for the hotfix path — even a stripped-down correctness and hard-block safety check that runs in under a minute — so "urgent" still passes through some automated check rather than none. Audit how often the hotfix path has been used and whether any of those changes would have failed the standard gate, to make the actual risk visible to the team and justify investing in a faster gate rather than an unguarded bypass.

---

**Situation:** A junior engineer asks why the team bothers with both a PR-time evaluation and a push-to-main evaluation when the PR was already approved and evaluated before merge — isn't that redundant?

Model answer: Explain that they check different things, not the same thing twice. The PR check proves the proposed diff doesn't regress against a small sample at proposal time, but between PR approval and merge, other changes can land on main that interact with this one in ways the PR-time check never saw in isolation. The push-to-main check runs the full evaluation suite against the actual merged state that will become a registered, deployable artifact — it's the last checkpoint before something becomes real, and it's deliberately more thorough (larger example count, all metrics) precisely because it's gating artifact registration, not just a proposed diff.

---

**Situation:** During a live incident, the on-call engineer wants to roll back to the previous model version, but the release dashboard only shows the current state — there's no way to see what the readiness signal looked like for the version they're rolling back to. What's missing, and how do you fix it?

Model answer: The dashboard is missing historical, per-version readiness records, not just a live snapshot — during an incident, the fastest safe path is rolling back to a version you can prove was healthy, not just the most recent one. Fix this by persisting every release's gate results and dashboard readiness signal in the experiment tracking system (tied to that version's exact model, prompt, and dataset identifiers) so an on-call engineer can pull up "here's what the readiness dashboard showed for version N-1 at the time it shipped" during an incident, rather than rolling back on faith or re-running a full evaluation under time pressure.
