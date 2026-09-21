# Prompt Lifecycle and Statistical Evaluation — Scenario-Based Q&A

**Situation:** An engineer needs to fix a bug in a live production prompt fast and edits the deployed prompt string in place to save time, planning to "document it properly later." What would you do and why?

Model answer: Stop the in-place edit before it ships — editing a "live" prompt file directly destroys reproducibility and makes every past log entry that references a version number silently wrong, since anything logged against "prompt v3" no longer reflects what v3 actually said. Require even urgent fixes to go through the same versioning discipline: create a new version (or at minimum a patch bump), tag it, and update the alias/pointer used by production, so the change is auditable and any regression can be bisected against a real version history rather than a retroactive memory of what changed.

---

**Situation:** A team compares a new candidate prompt against the current baseline by running the candidate on this month's live traffic sample and comparing its scores to last month's baseline evaluation numbers, and reports a clear win. What would you do and why?

Model answer: Flag this as the single most invalidating mistake in the module's list: comparing candidate and baseline on different eval examples (here, different traffic samples from different months) confounds any observed delta with which examples happened to be easier or harder in each period, and no amount of statistical sophistication afterward can rescue a comparison run on non-matching data. Insist on re-running both baseline and candidate on the exact same fixed, versioned eval set before drawing any conclusion, and treat the original "clear win" report as unverified until that re-run happens.

---

**Situation:** A candidate prompt scores 3 percentage points higher than baseline with p = 0.03 on a 5,000-example eval set, and the team wants to promote it immediately based on statistical significance. What would you do and why?

Model answer: Push back that p < 0.05 alone is not a sufficient promotion criterion — with a large enough sample size, trivially small and operationally meaningless improvements will reach statistical significance, and 3 points on 5,000 examples needs to be checked against a practical-significance floor (does a 3-point lift actually matter for the product, given the cost of the change) before promotion. Ask what minimum effect size was pre-registered as "worth shipping" before the experiment ran, and only promote if the observed lift clears both the statistical-significance bar and that practical-significance threshold — otherwise this is a textbook case of a large-n test detecting a real but irrelevant difference.

---

**Situation:** A team pre-computed a required sample size of 2,000 examples for a prompt-comparison experiment using power analysis, but three days in, with only 600 examples evaluated so far, someone notices the running p-value has already dipped below 0.05 and wants to call the experiment early. What would you do and why?

Model answer: Explain that repeatedly checking and stopping early once a fixed-n test crosses significance is classic p-hacking via optional stopping — a one-shot t-test's p-value is only valid if you commit to the pre-computed n and look once, and peeking repeatedly inflates the true false-positive rate well above the nominal 5%. Either commit to running the full pre-computed 2,000-example sample before looking at the result again, or, if continuous monitoring during accrual is genuinely needed, switch to a sequential testing method designed for repeated looks (which controls the false-positive rate under peeking) rather than treating an early significant result from a fixed-n test as valid.

---

**Situation:** A candidate prompt shows a large, statistically significant improvement on "correctness" as scored by an LLM judge, and the team is ready to promote it to 100% traffic. What would you do and why?

Model answer: Raise two separate concerns before promoting: first, the module warns against optimizing and gating on a single metric while ignoring latency, cost, and format/behavioral stability — a prompt that's "better" on correctness alone can still be worse in production economics or user experience, so check those dimensions too before promotion. Second, an LLM-judge "correctness" score is not ground truth — it carries the judge's own known biases (position bias, verbosity bias, self-enhancement bias), so a statistically significant lift in judge score is only as trustworthy as the judge itself; spot-check a sample of judged examples with human review to confirm the judge's verdict actually reflects real quality improvement before treating the statistical result as conclusive.

---

**Situation:** A team runs a prompt-promotion decision entirely through an automated statistical gate, and a change that clearly worsens the brand's tone of voice (subjectively obvious to anyone reading a few outputs) sails through because it didn't move any of the automated numeric metrics. What would you do and why?

Model answer: Identify this as fully automating a decision on a qualitative/subjective metric with no human-review escape hatch — tone and brand voice are exactly the kind of dimension that genuinely requires human judgment, and an all-automated gate either false-blocks good changes or, as happened here, lets bad ones through silently when the automated metrics don't capture the dimension that actually regressed. Add a human-review step to the promotion gate specifically for qualitative dimensions (a sampled manual read-through before full rollout), keeping the automated statistical gate for the metrics it's actually well-suited to measure, rather than trying to fully automate a judgment call.

---

**Situation:** After a rollback from a bad prompt version, the on-call engineer verifies the alias now points to the old version and closes the incident, but ten minutes later some users are still receiving responses that look like they came from the bad version. What would you do and why?

Model answer: Recognize the alias-cache-propagation gap explicitly called out in the module: repointing an alias doesn't mean every running instance has picked up the change instantly — some instances may still be serving a cached prior resolution of the alias for several minutes after the repoint. Don't close the incident the moment the alias shows the correct target; wait out (or actively invalidate) the known cache-propagation window and confirm via logs or telemetry that no instance is still resolving the old version before declaring the rollback complete, and document this propagation delay in the incident runbook so future on-call engineers don't repeat the same premature all-clear.

---

**Situation:** A team ships a new prompt version and changes its output schema — adding a required new field — but tags the release as a minor version bump since "it's just an addition." Several downstream consumers break immediately. What would you do and why?

Model answer: Identify this as conflating a MAJOR breaking change with a MINOR bump — adding a new *required* field changes the contract in a way that breaks consumers who assumed backward compatibility within a minor version, even though it "feels" additive from the producer's side. Correct the versioning convention going forward: any change that could break an existing consumer's parsing (new required fields, renamed/removed fields, changed types) is a MAJOR bump requiring explicit consumer coordination, while genuinely backward-compatible additions (new optional fields, non-breaking prompt wording tweaks with no schema change) are MINOR or PATCH. For the immediate incident, either roll back or make the new field optional with a sensible default until consumers can be migrated deliberately.

---

**Situation:** A team wants to choose a sample size for an upcoming prompt-comparison experiment and settles on "let's just use 50 examples, that feels like enough" to keep the eval fast. What would you do and why?

Model answer: Reject the arbitrary sample-size choice and require a power analysis instead — choosing n by feel rather than by tying it to the minimum effect size that actually matters for the decision risks an underpowered experiment that can neither reliably detect a real improvement nor give a trustworthy null result. Run a power analysis using the smallest effect size the team would actually act on (the practical-significance floor) to compute the required n before running the experiment, accepting that this may mean a larger, slower eval than 50 examples if the effect size of interest is small — an underpowered fast experiment that produces an unreliable answer is slower in the end than a properly sized one.

---

**Situation:** A team pairs baseline and candidate prompt outputs on the same eval examples but runs an unpaired (independent-samples) statistical test to compare their scores, reasoning "it's just a t-test either way." What would you do and why?

Model answer: Correct the test choice: this is paired data (same example evaluated under both baseline and candidate), and using an unpaired test on paired data wastes statistical power at best, and can produce an outright invalid p-value at worst, since it ignores the correlation between the two measurements taken on the same example. Switch to the correct paired test (e.g., a paired t-test or Wilcoxon signed-rank test depending on the score distribution), and treat "same underlying eval examples, two conditions" as the signal that should always trigger a paired-test choice, not an incidental detail.

---

**Situation:** A team notices that their eval-set-based promotion decisions have started drifting from what production metrics actually show after rollout — prompts that "won" the offline comparison don't reliably perform better live. What would you do and why?

Model answer: Investigate whether the eval dataset itself has been silently changing between comparisons — if new examples have been added or old ones edited between two prompt evaluations without formal versioning, the fixed-comparison guarantee that made the original statistical test valid has been broken, since "compare on the same eval set" quietly stopped being true. Audit whether the eval dataset is versioned like code (a specific, immutable version referenced by each comparison run), and if it isn't, implement that discipline immediately — an eval set that can silently change is one of the most common, hardest-to-notice causes of offline-online metric divergence, independent of anything wrong with the prompts themselves.

---

**Situation:** A prompt-lifecycle retrospective is scheduled after a bad prompt version caused a two-hour production incident, and the team's instinct is to add more manual sign-offs before any prompt can ship. What would you do and why?

Model answer: Use the retrospective to trace exactly which lifecycle safeguard was missing rather than defaulting to "add more approvals" — check specifically whether the eval set used to validate the bad version was the correct, versioned, fixed comparison set; whether the promotion decision passed a practical-significance floor or just a bare p-value; whether latency/cost/format-stability were checked alongside correctness; and whether the alias-repoint and cache-propagation window were handled correctly during what should have been a fast rollback. A missing statistical or lifecycle safeguard (identified precisely) is a more durable fix than a generic extra human sign-off, which adds friction without necessarily catching the specific failure mode that actually occurred.

---

**Situation:** A data scientist argues that since their LLM-judge-based "correctness" metric already incorporates GPT-4-class reasoning, it should be trusted as ground truth and no further human validation is needed going forward. What would you do and why?

Model answer: Push back directly: an LLM judge is not ground truth, and the module explicitly names known judge biases — position bias (favoring answers in a particular position), verbosity bias (favoring longer answers regardless of quality), and self-enhancement bias (a judge favoring outputs from the same model family it belongs to) — that can systematically distort judge scores in ways a single confident-sounding score doesn't reveal. Recommend periodic calibration of the judge against a smaller human-labeled sample (checking agreement rate, not just trusting judge output), and treat any statistically significant judge-score lift as provisional until spot-checked against human judgment, especially before a high-stakes promotion decision.

---

**Situation:** Two teams are comparing prompt versions for the same underlying task but on different eval datasets each team built independently, and are confused why their conclusions about "which version is better" disagree. What would you do and why?

Model answer: Point out that this isn't necessarily a contradiction requiring reconciliation — different eval sets can have different example distributions (different difficulty mix, different edge-case coverage), so two valid, independently-run comparisons on different datasets can legitimately reach different conclusions about relative performance. Recommend consolidating onto one shared, versioned eval set for this task going forward specifically so future comparisons are commensurable, and in the meantime, don't treat either team's result as simply wrong — instead audit what each eval set actually covers to understand why the conclusions diverged, since the disagreement itself may reveal a real difference in how each prompt performs on different input distributions.

---

**Situation:** A team wants their prompt-promotion CI check to be fully automated with no human step at all, arguing that fast iteration requires removing every manual gate. What would you do and why?

Model answer: Support automating what genuinely can be automated well — paired statistical significance testing against a fixed, versioned eval set, a practical-significance floor check, and format/latency/cost regression checks are all good candidates for a fully automated CI gate — but explicitly carve out an exception for qualitative/subjective dimensions like tone or brand voice, where the module's guidance is that full automation either false-blocks good changes or silently lets bad ones through. Propose a hybrid gate: automated statistical checks block or pass the bulk of routine changes fast, with a lightweight, sampled human-review step specifically for changes flagged as touching subjective/qualitative behavior, preserving iteration speed for the common case without giving up the judgment that automation can't provide.
