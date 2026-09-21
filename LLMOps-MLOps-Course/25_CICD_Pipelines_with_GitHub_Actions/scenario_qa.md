# CI/CD Pipelines with GitHub Actions — Scenario-Based Q&A

**Situation:** A pull request that changed a data-preprocessing function passed all CI checks and was merged, but broke the production training pipeline the next day because a dependency version installed locally by the author differed from what's in `requirements.txt`. What would you do and why?

Model answer: This points to CI not actually rebuilding a clean, pinned environment — if it had, the same version mismatch would have failed in CI before merge, not in production after. Audit the CI workflow to confirm it truly installs from the pinned `requirements.txt` (or lockfile) inside a fresh runner rather than reusing a cached environment that might contain stale or manually-installed packages. Once confirmed clean, this specific failure suggests the local dev environment and CI environment have diverged — enforce that all dependency changes go through the pinned file and get tested in CI, and consider adding a check that fails if `requirements.txt` doesn't match what's actually importable/used in the code.

---

**Situation:** Your team's CI pipeline takes 25 minutes to run, and engineers have started merging pull requests without waiting for it to finish because "it's basically always green anyway." What would you do and why?

Model answer: Treat this as the pipeline losing its function as a gate in practice, regardless of branch protection settings — a slow pipeline that people route around provides none of CI's actual safety benefit while still costing compute and maintenance. Profile the 25 minutes to find the actual bottleneck (dependency installation without caching is a common one — add `actions/cache` for pip/conda environments), parallelize independent test suites across multiple jobs instead of running them sequentially, and consider splitting fast unit tests (which run on every push) from slower integration/evaluation tests (which could run less frequently or only on PRs targeting main). The goal is fast enough that waiting is the path of least resistance, not something engineers are incentivized to bypass.

---

**Situation:** A critical bug reached production because a pull request's tests passed, but the branch protection rule requiring the CI workflow to pass before merge wasn't actually enabled on the `main` branch. What would you do and why?

Model answer: Recognize the gap precisely: a green CI workflow that isn't wired to branch protection is a notification, not a gate — anyone can merge regardless of its result. Enable branch protection on `main` requiring the CI workflow to succeed before merge is allowed, and audit whether other critical branches have the same gap. Treat this as a process failure worth a short retro: the workflow existing and appearing to work (green checkmarks visible on PRs) created false confidence that it was actually enforced, which is exactly the kind of silent gap that's worth explicitly verifying rather than assuming across all protected branches.

---

**Situation:** An engineer wants to add a nightly scheduled workflow that automatically retrains the model and opens a pull request if evaluation metrics improve, but a colleague is nervous about "the pipeline making decisions on its own." What would you do and why?

Model answer: Clarify what the workflow should and shouldn't automate — automating the retraining run and evaluation comparison is safe and valuable (it removes manual toil and runs consistently every night), but automating the merge itself is where the colleague's concern becomes legitimate, since a model regression that passes a flawed automated check could reach production unreviewed. Design the workflow to open a pull request with the new model's evaluation results clearly reported, requiring human review and explicit merge approval — this keeps CI/CD doing what it's good at (consistent, automatic verification) while keeping a human as the final gate for consequential decisions, matching the "keep a human in the loop for high-stakes actions" principle from the governance module.

---

**Situation:** Your CI workflow runs `pytest` but doesn't include the model-evaluation script, and a change that silently degraded model accuracy merged cleanly because standard unit tests don't check model quality. What would you do and why?

Model answer: Point out that "tests passing" and "the model still works well" are different claims, and CI needs to check both — standard unit/integration tests verify code correctness, but only an evaluation gate (running the golden dataset through the model and checking metrics against a threshold) catches a quality regression that doesn't break any function's contract. Add an evaluation step to the workflow that fails the build if a key metric (accuracy, faithfulness, whatever's relevant) drops below a defined threshold compared to the current production baseline, similar to the deployment quality gates covered in Module 12. Balance this against pipeline speed — if the full evaluation set is slow, consider running a fast subset on every push and the full set before merge to main or before deployment.

---

**Situation:** A team member proposes giving the CI runner production database credentials so integration tests can run "against real data" instead of a test fixture. What would you do and why?

Model answer: Push back strongly — CI runners are ephemeral, often run untrusted code from pull requests (including from forks, in open-source-style workflows), and log their output, all of which make them a poor place to hold production credentials; a compromised or misconfigured workflow step could leak or misuse them. Use a dedicated test database (seeded with realistic but non-sensitive fixture data) or a sandboxed copy of production data with sensitive fields scrubbed, injected via the CI's own secrets mechanism scoped only to that test environment. If "testing against real data" is about catching real-world edge cases, address that by making the fixture data more representative, not by widening production credential access to a CI runner.

---

**Situation:** After merging a change, the deployment step silently failed because a required environment variable wasn't set in the CD workflow, but the CI workflow (tests) showed green, and nobody noticed for two days. What would you do and why?

Model answer: This shows CI and CD being treated as one undifferentiated "pipeline passed" signal when they're answering different questions — tests passing tells you the code is correct, not that it deployed. Add explicit monitoring/alerting on deployment workflow outcomes specifically (a Slack or email notification on CD failure, not just a GitHub UI checkmark easy to miss), and add a post-deployment smoke test or health check that verifies the new version is actually serving traffic correctly, catching silent deployment failures immediately rather than after days of unnoticed staleness. Audit the CD workflow's required environment variables/secrets against what's actually configured in the deployment target to close this specific gap.

---

**Situation:** Your organization wants to start deploying model updates automatically via CD, but has never done automated deployment before and leadership is nervous about a bad model reaching all users at once. What would you do and why?

Model answer: Recommend starting with CI-only for a period (automated build/test/evaluate on every change, but manual deployment) to build trust in the gates before adding automatic deployment, exactly as the module suggests — CD should be earned, not assumed. When ready to add CD, don't jump straight to "every merge auto-deploys to 100% of traffic" — use a gradual rollout pattern (canary or blue-green deployment, feature flags, or a percentage-based rollout) so a bad model or prompt change affects a small fraction of traffic before being caught by monitoring and automatically or manually rolled back. This directly addresses the "all at once" fear with a concrete mechanism rather than just reassurance.

---

**Situation:** A workflow file references a third-party GitHub Action (`some-random-user/cool-action@v1`) that a team member found in a blog post, and someone raises a concern about supply-chain risk. What would you do and why?

Model answer: Take the concern seriously — an Action is just code that runs with access to your repository and any secrets exposed to that workflow, and pinning to a mutable tag like `@v1` means the Action's maintainer could push new, potentially malicious code to that tag without your workflow file changing at all. Prefer well-known, widely-used Actions (like `actions/checkout`, `actions/setup-python`) maintained by GitHub or reputable organizations, and where a third-party Action is genuinely needed, pin to a specific commit SHA rather than a tag, and review the Action's source before adopting it. Treat this the same as any other third-party dependency: unreviewed code with access to your secrets is a real risk, not a theoretical one.

---

**Situation:** Your CI pipeline lints, tests, and evaluates the model on every single push to every branch, including work-in-progress branches with dozens of small commits per day, and the team's Actions minutes bill has grown noticeably. What would you do and why?

Model answer: Reconsider what should trigger the full pipeline versus a lighter check — running the complete suite (including a potentially slow evaluation step) on every WIP commit to a feature branch is expensive for little benefit, since the meaningful gate is really "does this pass before merging to main." Restructure triggers so pushes to feature branches run fast checks only (linting, quick unit tests), while the full suite including evaluation runs on pull requests targeting `main` (or right before merge), matching thoroughness to the moment it actually matters. This cuts cost without weakening the actual gate, since nothing merges to main without the full check regardless of how the interim commits were validated.

---

**Situation:** A workflow step to install dependencies takes 8 of the pipeline's 25 minutes, reinstalling the same packages from scratch on every single run. What would you do and why?

Model answer: This is a caching gap — dependencies that haven't changed shouldn't be reinstalled from the internet every run. Add `actions/cache` keyed on a hash of the dependency lockfile (`requirements.txt` or `poetry.lock`), so the cache is reused whenever dependencies are unchanged and only invalidated when the lockfile itself changes. This is one of the highest-leverage, lowest-risk speed improvements available for a typical Python CI pipeline, and it's worth checking across all workflows in the repo, not just the one that prompted the investigation, since the same pattern often repeats.

---

**Situation:** Your team just adopted GitHub Actions and a stakeholder asks why the workflow needs to be "reviewed and versioned like the code," assuming CI configuration is just plumbing that doesn't need the same scrutiny. What would you do and why?

Model answer: Explain that the workflow file is exactly as consequential as the code it gates — it decides what quality bar is enforced before anything reaches production, and a subtle misconfiguration (a test that's silently skipped, a threshold set too loosely, a branch-protection gap) can undermine every downstream safety guarantee the team believes it has. Because it lives in `.github/workflows/` as YAML in the repo, it already gets the same review and version-history benefits as application code — the point is to actually use those tools (PR review on workflow changes, not just application code changes) rather than treating pipeline YAML as a lower-scrutiny category by convention.

---

**Situation:** After a workflow change, someone notices the CI pipeline has been silently skipping the model-evaluation step for the last three weeks due to a typo in a conditional (`if: false` accidentally left in from debugging), and nobody caught it because the workflow still reported green. What would you do and why?

Model answer: This is a sharp reminder that "green" doesn't necessarily mean "did what you think it did" — a skipped step can still leave the overall workflow status green if it's not marked as required, so passing status alone isn't sufficient verification. Fix the immediate typo, then add a safeguard against recurrence: either make the evaluation step required (so its absence, not just its failure, breaks the build) or add a lightweight check that asserts the expected number of steps actually ran. Retroactively check whether any of the three weeks of merges would have failed evaluation had it actually run, since those changes reached main unverified on this dimension.

---

**Situation:** A team wants to run their GPU-dependent model-evaluation step in CI, but GitHub's free hosted runners don't provide GPU access, and the team isn't sure how to proceed. What would you do and why?

Model answer: Introduce self-hosted runners as the standard solution for this exact gap — GitHub allows registering your own machine (including one with a GPU) as a runner for your workflows, so GPU-dependent steps can run in the same CI pipeline as everything else. Weigh the tradeoff: self-hosted runners require you to manage the machine's security and uptime yourself (unlike GitHub's fully managed hosted runners), so scope what runs on it carefully, especially if the repository accepts external pull requests, since a self-hosted runner executing untrusted PR code is a real security consideration. If GPU evaluation is only needed occasionally rather than on every push, consider running it on a schedule or only for pull requests targeting main, rather than paying the self-hosted-runner overhead on every commit.

---

**Situation:** Your team's `main` branch requires CI to pass, but an engineer under deadline pressure asks if they can just push directly to `main`, bypassing the pull request process entirely "just this once" to save time. What would you do and why?

Model answer: Decline, and explain precisely why the exception is more costly than the time saved — bypassing the PR process means bypassing both the CI gate and human review in one move, and "just this once" is exactly the kind of exception that, if granted, quietly erodes the norm that main is always in a deployable state. If speed is the real constraint, address it directly: check whether the CI pipeline itself is too slow (fix that, as in an earlier scenario) or whether the review process has unnecessary friction — but the fix should make the proper path faster, not create a bypass that undermines the entire point of branch protection.

---

**Situation:** A new hire on the team asks why the course separates "CI" and "CD" as distinct concepts when their previous company just called the whole thing "the pipeline." What would you do and why?

Model answer: Explain the distinction because it maps to a real difference in risk tolerance and cadence — Continuous Integration (build, test, evaluate on every change) is safe to run constantly and aggressively, since its only effect is reporting pass/fail; Continuous Delivery/Deployment (actually shipping the result — a Docker image, a deployed model endpoint) has real production consequences and usually warrants more deliberate triggers (merge to main only, manual approval gates, gradual rollout). Conflating them under one mental model risks either being too cautious with CI (slowing down every push unnecessarily) or too casual with CD (auto-deploying on triggers that were only ever validated as safe for testing purposes). Naming them separately keeps the team's caution calibrated to where it actually matters.
