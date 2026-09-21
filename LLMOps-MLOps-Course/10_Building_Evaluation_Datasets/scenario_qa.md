# Building Evaluation Datasets — Scenario-Based Q&A

**Situation:** A team building their first eval set for a new RAG assistant asks their most senior engineer to hand-pick 100 "interesting" examples that showcase the kinds of questions the system should handle. What would you do and why?

Model answer: Push back on hand-picking "interesting" examples as the sourcing method — this is selection bias by construction, and even a well-intentioned expert curator reliably misses the boring-but-common failure modes that dominate real user pain, because "interesting" examples are, almost by definition, not representative of typical traffic. Recommend production-sourced, stratified sampling instead: pull a representative sample of real user queries (stratified across query type, difficulty, and known categories), supplemented deliberately with a smaller, explicitly-labeled slice of edge cases — not a purely curator's-judgment selection of what seems notable.

---

**Situation:** Six months after launch, a team's eval set is still 100% synthetically generated data created before the product had any real users, and the eval scores look great while user complaints keep rising. What would you do and why?

Model answer: Diagnose the gap directly: synthetic examples are clean and well-formed in ways real users are not, so a system scoring well only on synthetic data has an unmeasured gap against real-world messiness — typos, ambiguous phrasing, multi-intent queries, and genuinely novel edge cases that a synthetic generator wouldn't naturally produce. Synthetic data is appropriate for bootstrapping before real usage exists, but past that phase, replace or heavily supplement it with production-sourced examples (real logged queries, properly stratified and gold-labeled), and treat continued reliance on synthetic-only eval past the bootstrap phase as a known blind spot until that migration happens.

---

**Situation:** A team labels their eval set using a single subject-matter expert and reports the labels as "gold" in their documentation, without ever measuring agreement with a second labeler. What would you do and why?

Model answer: Flag this as treating "we have labels" as sufficient without measuring agreement — a single annotator's opinion, however expert, is not a validated gold label until a second independent rater has labeled the same data and a computed agreement statistic (like Cohen's kappa) confirms real consensus rather than one person's idiosyncratic judgment. Have a second qualified labeler independently label a meaningful sample (or the whole set, if feasible), compute inter-rater agreement, and only call the result "gold" once agreement is confirmed at an acceptable level — if agreement is low, that's information the team needs before trusting any eval result built on those labels, not something to skip past.

---

**Situation:** After computing inter-rater agreement, a team is alarmed to see a low kappa score on their edge-case-heavy eval slice and immediately schedules a full re-labeling effort to "fix" the disagreement. What would you do and why?

Model answer: Slow down before re-labeling — this may be the kappa paradox: kappa can look artificially low even when raw percent agreement is high, specifically when the label distribution is highly skewed (as an edge-case-heavy slice, by design, often is), because kappa corrects for chance agreement in a way that behaves counterintuitively under skew. Check raw agreement and the label-distribution skew before concluding the labelers actually disagree — if raw agreement is high and the low kappa is a distributional artifact of an intentionally imbalanced edge-case slice, re-labeling a perfectly fine batch wastes effort and doesn't fix anything, since there's nothing broken to fix.

---

**Situation:** A team notices their frozen 500-example eval set includes very few examples of a minority language variant their product actually serves, and someone proposes deleting the few examples that are there since "they're too small a sample to matter statistically anyway." What would you do and why?

Model answer: Reject the deletion outright — the module's explicit guidance is flag, don't delete: removing underrepresented categories to "fix" a balance number hides a real coverage gap rather than documenting and addressing it, and a small sample being statistically underpowered on its own is a reason to grow that slice, not erase it. Flag the underrepresentation explicitly in the eval set's documentation, treat closing the gap as a tracked follow-up (deliberately sourcing more examples of that language variant), and in the meantime report results on that slice with appropriate caveats about sample size rather than pretending the gap doesn't exist by deleting the evidence of it.

---

**Situation:** A team ran a prompt comparison on their 80-example eval set and is excited that the candidate scored 2 points higher than baseline, and wants to ship it immediately. What would you do and why?

Model answer: Raise the sample-size concern directly — freezing an eval set with fewer than roughly 200 examples and treating small deltas as meaningful risks acting on noise, since an underpowered eval set produces noisy pass/fail deltas that don't reliably reflect real regressions or improvements. Recommend growing the eval set toward an adequately powered size (informed by a power analysis tied to the effect size that actually matters, per Module 09) before trusting a 2-point delta on just 80 examples, and in the meantime treat this result as suggestive, not a shipping decision, especially for a change with real production consequences if wrong.

---

**Situation:** A team's eval set was carefully built and frozen a year ago, and lately it keeps giving passing scores to a system that production monitoring shows is clearly struggling with newer types of user queries that didn't exist when the eval set was built. What would you do and why?

Model answer: Identify this as a stale edge-case slice — freezing the edge-case allocation once and never refreshing it means it increasingly tests yesterday's failure modes rather than today's, and a system can look fine against a year-old eval set while genuinely failing on newer query patterns the eval set never anticipated. Schedule a real, tracked quarterly (or similarly regular) refresh of the edge-case slice, sourcing new examples from recent production traffic and recent incident/failure reports specifically, and treat "the eval set passes" as meaningful only in the context of how recently its edge-case coverage was refreshed.

---

**Situation:** Midway through an in-flight prompt-comparison experiment, an engineer notices a labeling error in the eval set and quietly fixes three mislabeled examples so the ongoing comparison uses "better" data. What would you do and why?

Model answer: Stop the mid-experiment edit — silently resampling or re-labeling a dataset mid-experiment invalidates every in-flight comparison and is one of the most common causes of "the eval said it improved, but we changed the test at the same time" confusion, since any observed delta can no longer be attributed cleanly to the prompt change alone. Let the current experiment finish on the original (even if flawed) frozen version of the eval set, document the labeling error found, fix it in a new, explicitly versioned release of the eval set, and only use the corrected version for the next comparison going forward — never mutate a dataset a comparison is currently running against.

---

**Situation:** A team designs their eval dataset schema entirely around DeepEval's specific field-naming conventions, and eighteen months later, when they need Ragas-specific faithfulness and context-precision metrics, discover a costly migration is required just to make their data compatible. What would you do and why?

Model answer: Point to this as exactly the risk the module warns about: designing the dataset schema around one eval framework only creates a costly migration later when the team needs the other framework's specific metrics, since field names and structural assumptions baked in early become a compatibility tax down the line. Recommend, for future datasets, designing the schema to be compatible with both DeepEval and Ragas conventions from the start (or maintaining a thin, framework-agnostic canonical schema with adapters to each framework's expected format), and for the current migration, budget it as a one-time schema-generalization cost rather than repeatedly patching around the DeepEval-specific assumptions.

---

**Situation:** A team is proud that their eval set "looks fine" on manual review — a few engineers skimmed a sample and didn't see anything obviously wrong — and treats that as sufficient quality assurance before freezing it. What would you do and why?

Model answer: Reject "looks fine" as a substitute for the bias audit — the module frames every one of the five bias types (selection, distributional/coverage, labeling, framework-lock-in-adjacent schema bias, and staleness) as measurable, meaning there's a concrete checklist that can actually be run rather than relying on a subjective skim. Run the formal bias audit checklist before freezing (checking source representativeness, category balance with explicit flags for gaps rather than deletions, label agreement via kappa, and edge-case slice recency) and record the results in the dataset's documentation, since a documented audit is falsifiable and auditable in a way "a few engineers looked at it and it seemed okay" is not.

---

**Situation:** A team needs to add 50 new examples to their production eval set to cover a recently discovered failure mode, and debates whether this requires a new dataset version or can just be appended to the existing file in place. What would you do and why?

Model answer: Treat this the same as code: version the dataset like code, meaning this addition should produce a new, explicitly versioned release of the eval set (not a silent in-place append), with a changelog entry describing what was added and why. Any in-flight or historical comparison that referenced the prior version needs to keep referencing that exact prior version, and new comparisons going forward use the new version — silently appending in place breaks the same fixed-comparison guarantee that mid-experiment resampling breaks, even though the motivation (closing a real coverage gap) is legitimate.

---

**Situation:** A team building an eval set for a customer-support classifier only sources examples from tickets that were successfully auto-resolved, since those are the easiest to pull from the support system's database. What would you do and why?

Model answer: Identify a subtle selection bias: sourcing only from successfully auto-resolved tickets systematically excludes the harder, ambiguous, or escalated cases that are exactly where classification failures matter most — an eval set built this way will look great while never testing the failure modes that actually drive human-escalation cost. Deliberately stratify the sourcing to include escalated, human-handled, and disputed tickets alongside auto-resolved ones, even though they're harder to pull and label, since the easiest-to-access data being systematically unrepresentative of the hardest cases is a recurring, easy-to-miss pattern in eval dataset construction.

---

**Situation:** A newly hired engineer, reviewing the team's eval dataset documentation, notices there's no record of when the dataset was created, what its intended coverage is, or what bias-audit results exist — just a CSV file sitting in a shared drive. What would you do and why?

Model answer: Treat the missing documentation as a real gap worth fixing before trusting the dataset further, not a cosmetic issue — without a recorded creation date, sourcing methodology, coverage intent, and bias-audit results, no one can tell whether the dataset is stale, whether it was ever audited, or whether its labels were ever agreement-checked, which undermines confidence in every comparison that's ever used it. Backfill what can be reconstructed (approximate creation date, best-effort sourcing description) and run a fresh bias audit and kappa check going forward if none exists in any recoverable form, then require every future eval dataset (and every future version bump) to ship with this documentation as a non-optional part of the release, not an afterthought.

---

**Situation:** A team is deciding how large an initial edge-case allocation should be within a new eval set and debates between a token handful (5%) versus a substantial slice (20%), with an engineer arguing "our system handles typical cases fine, edge cases are rare in real traffic anyway." What would you do and why?

Model answer: Push back on sizing the edge-case slice purely by its real-traffic frequency — edge cases are rare in raw traffic precisely because they're edge cases, but they're disproportionately where failures concentrate and disproportionately costly when they go wrong (a wrong answer on a common, easy query is usually low-stakes; a wrong answer on a genuinely hard edge case is often where real harm or embarrassment happens). Favor a deliberately over-weighted edge-case allocation (the module's 20% figure is a reasonable anchor) relative to its real-traffic frequency, precisely because the eval set's job is to stress-test the system's weak points, not to mirror the traffic distribution exactly — and pair it with the quarterly refresh discipline so that slice doesn't go stale.
