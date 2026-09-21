# Module 12 — Exercises: Deployment Quality Gates and Release Dashboards

These exercises progress from "wire a single gate" to "design and defend a full release-readiness
system in an interview setting." Each one builds on artifacts from the previous one where noted.
Use the code in `tutorial.md` (`evals/gate.py`, the GitHub Actions YAML, the Grafana provisioning
YAML) as your starting scaffold rather than writing everything from scratch — this module is about
composing triggers + gates + dashboards correctly, not about re-deriving evaluation harnesses
(that's Modules 09–11).

---

## Exercise 1 — Write your first absolute gate (Beginner)

**Goal.** Implement a single-file Python gate script that reads a JSON metrics file and exits
non-zero when any absolute threshold is violated.

**Task.**
1. Create `results/candidate.json` with fields `eval_score`, `p95_latency_ms`, `hallucination_rate`,
   `format_valid_rate`.
2. Write `gate_v1.py` that loads the file and checks:
   - `eval_score >= 0.82`
   - `p95_latency_ms <= 1200`
   - `hallucination_rate <= 0.03`
   - `format_valid_rate == 1.00`
3. Print a clear pass/fail message and call `sys.exit(1)` on any failure, `sys.exit(0)` on full pass.
4. Run it three times with three different hand-edited `candidate.json` files: one that passes
   everything, one that fails only latency, one that fails only hallucination.

**Done looks like:**
- Three runs, three different exit codes observed via `echo $?` (or `$LASTEXITCODE` in PowerShell)
  immediately after each run.
- The failure message names the specific metric and threshold that failed — not just "FAILED".
- You can explain out loud why this script, as written, does **not** yet distinguish a hard block
  from a regular absolute failure (that's Exercise 3).

---

## Exercise 2 — Add the relative gate against a stored baseline (Beginner/Intermediate)

**Goal.** Extend Exercise 1 so the candidate is also compared against a production baseline file,
not just fixed floors.

**Task.**
1. Create `results/production_baseline.json` with an `eval_score` of `0.83`.
2. Add a relative check: candidate must satisfy `candidate_score >= baseline_score - 0.01`.
3. Construct three test cases:
   - Candidate score `0.90` (clears absolute floor, clears relative check) → PASS
   - Candidate score `0.85` (clears absolute floor 0.82, but is `0.85 < 0.83 - 0.01 = 0.82`... adjust
     your baseline to `0.87` so this case genuinely demonstrates "clears absolute, fails relative")
   - Candidate score `0.60` (fails both) → FAIL, and your reason list should show **two** entries.

**Done looks like:**
- You have one concrete, reproducible test case where a candidate clears the absolute gate but is
  still blocked by the relative gate — and you can explain in one sentence why that's not a bug.
- Your `gate_reason.json` output records both `candidate_metrics` and `baseline_metrics` so a human
  reading it later doesn't have to re-run anything to understand the decision.

---

## Exercise 3 — Introduce the hard-block category (Intermediate)

**Goal.** Refactor Exercise 2's gate so that `format_valid_rate` and `hallucination_rate` failures
are flagged as `hard_block: true` in the output, distinct from ordinary absolute failures like
latency or score.

**Task.**
1. Add a `hard_block` boolean field to your result object (see `GateResult` in `tutorial.md` §3.2
   for the reference shape).
2. Write a test case where `eval_score` is excellent (`0.95`) but `format_valid_rate` is `0.98`
   (i.e., 2% of outputs are malformed). Confirm the gate still blocks the release and the output
   clearly marks it as a hard block.
3. Add a code comment (not a config flag — make it structurally obvious in the code) explaining why
   there is no override parameter for hard blocks, while a "waiver" or "override" path could
   plausibly exist for a latency-only failure.

**Done looks like:**
- Your test case proves that a high composite/quality score cannot mathematically compensate for a
  hard-block failure — this is the "no averaging away a safety failure" property from Common
  Mistake #2 in `tutorial.md`.
- You can point to the exact line of code that would need to change to add an override, and explain
  why you deliberately did not add it.

---

## Exercise 4 — Wire the gate into a real GitHub Actions PR check (Intermediate)

**Goal.** Take the gate script from Exercise 3 and actually make it block a pull request in a real
(or throwaway test) GitHub repository.

**Task.**
1. Create a small repo (or a scratch folder inside an existing one) with your gate script and a
   `requirements-eval.txt`.
2. Write `.github/workflows/pr-gate.yml` with a single job that runs on `pull_request`, generates
   (or reads a checked-in fixture) `candidate.json`, and runs your gate script.
3. Open a PR that intentionally has a `candidate.json` fixture with `hallucination_rate = 0.05`.
   Confirm in the GitHub Actions UI that the check fails and the PR shows a red X / blocked status.
4. Fix the fixture to `hallucination_rate = 0.01`, push a new commit, and confirm the check goes
   green and (if branch protection is enabled) the merge button unblocks.

**Done looks like:**
- A screenshot or a described GitHub Actions run URL showing one red run and one green run on the
  same PR, driven only by the fixture's metrics changing.
- You can explain what would have happened if you had only printed the failure message without
  calling `sys.exit(1)` — i.e., you've verified the "log a warning vs. actually block" distinction
  in a real CI system, not just in theory.

---

## Exercise 5 — Design and cost out a full four-trigger tiered strategy (Intermediate/Advanced)

**Goal.** Given a hypothetical team's cadence, design the tiered evaluation strategy and compute
its weekly cost, mirroring the worked example in `tutorial.md` §3.1.

**Task.** Given: a team merges 8 PRs/day (5 days/week) and pushes to main 3 times/day. Tier 1 costs
$0.36/run, Tier 2 costs $1.44/run, and Tier 3 runs once a week at a flat $6.00/run regardless of
example count.
1. Compute the weekly PR count, weekly push count, and total weekly evaluation spend.
2. Now suppose leadership asks you to cut spend by 40% without removing any trigger entirely.
   Propose a concrete change (e.g., reduce Tier 1 sample size, move Tier 2 to every-3rd-push with a
   nightly batch instead of every push, etc.) and recompute the new weekly cost.
3. Write two sentences justifying to a skeptical engineering manager why you would **not** propose
   removing Tier 3 (the schedule trigger) as the cost-cutting lever, even though it's the least
   frequent and therefore looks like an "easy" cut on paper.

**Done looks like:**
- A clearly shown arithmetic answer for both the original and the cost-reduced weekly spend.
- Your proposed cost-cutting change does not touch any hard-block check's coverage — you can name
  which checks you would never reduce sample size on and why.

---

## Exercise 6 — Build the structured readiness payload and a minimal dashboard (Advanced)

**Goal.** Produce the JSON contract from `tutorial.md` §3.3 from real (or fixture) data, and render
it as a static four-layer dashboard — Grafana if you have it available, or a simple HTML/script
substitute if not.

**Task.**
1. Write a script that takes `candidate.json`, `baseline.json`, and per-dimension scores, and emits
   a `readiness.json` payload matching the shape shown in the tutorial (version, composite_score,
   status, dimensions, trend_30d, regression_vs_baseline).
2. Render it as a dashboard with the four-layer reading order: headline status, dimension
   breakdown, a synthetic 30-day trend (you can fabricate 30 plausible daily points that show one
   real dip and recovery), and regression-vs-baseline.
3. If you have Grafana + Prometheus available (or can stand them up locally via Docker per Module
   04/05), wire this to real panels using the provisioning YAML pattern from `tutorial.md` as a
   starting point. If not, a static HTML page or a Jupyter notebook rendering the same four layers
   in the same order is an acceptable substitute — the reading-order discipline matters more than
   the specific rendering technology for this exercise.

**Done looks like:**
- Your dashboard's headline threshold for "green" uses the **exact same numeric threshold** your
  Exercise 3 gate code uses — you can point to both places and show they read from one shared
  source of truth (a single constants file or config), not two independently typed numbers.
- A colleague (or you, a day later) can look at the dashboard for 30 seconds and state a ship/hold
  decision without opening any logs.

---

## Exercise 7 — Diagnose a broken gate system from symptoms (Advanced)

**Goal.** Practice the diagnostic reasoning a senior engineer needs when a gate system is present
but regressions are still reaching users — this exercise has no code, only a scenario and a
written analysis, mirroring Interview Question 5 in `tutorial.md`.

**Task.** You're told: "Our gate has passed every release for the last two months. Users are now
reporting that the assistant gives outdated answers about a policy that changed six weeks ago."
Write a structured diagnosis covering:
1. Which of the four triggers (PR, push, schedule, data-update) is most likely to have caught this,
   and why the other three plausibly would not have.
2. Two concrete questions you'd ask about the eval set's freshness and representativeness.
3. One check you'd add to the gate/dashboard system specifically to catch this class of failure in
   the future, described precisely enough that another engineer could implement it from your
   description alone.

**Done looks like:**
- Your written diagnosis explicitly rules out the PR and push triggers as "not the right layer for
  this failure" with a one-sentence reason each — not just "add more eval."
- Your proposed fix names a specific trigger tier and a specific eval-set change (e.g., "add a
  policy-currency check to the Tier 3 weekly suite, sourced from a documents-changed feed") rather
  than a vague "improve evaluation."

---

## Exercise 8 — Design the gate/dashboard system for a new artifact type end-to-end (Capstone)

**Goal.** Given a new artifact type your team is about to ship for the first time — a tool-calling
agent that can execute financial transactions on a user's behalf — design the complete
trigger/gate/dashboard system using the decision tree in `architecture.md` §3, and defend it in
writing as if presenting to a review board.

**Task.** Produce a short design document (1–2 pages) covering:
1. Which triggers apply (walk the decision tree from `architecture.md` explicitly — show your Q1
   through Q6 answers for at least three distinct metrics: transaction-format-validity,
   transaction-amount-accuracy, and response-latency).
2. Which metrics are hard blocks and which are standard absolute/relative gates, with a one-sentence
   justification for each classification (a financial agent has at least one hard-block candidate
   that has no analogue in a plain chat assistant — identify it).
3. The full Grafana panel layout (you may reuse the wireframe structure from `architecture.md` §4,
   adapted with your own thresholds).
4. One paragraph on what changes about the alerting/paging strategy given that this agent can take
   real-world financial actions — specifically, what would you add beyond what `tutorial.md`
   describes for a text-only chat assistant, and why the "already live, can only page" branch of the
   decision tree is especially high-stakes here.

**Done looks like:**
- A reviewer with no other context on your system could read your document and understand exactly
  which check blocks what, at which trigger, with which severity — without needing to ask you a
  clarifying question about scope.
- You've identified at least one new hard-block candidate specific to financial actions (e.g.,
  "transaction amount must exactly match user-confirmed amount, zero tolerance") that goes beyond
  the format/hallucination pair used as the running example throughout this module.
- You can explain, out loud in under two minutes, why a scheduled/drift-triggered failure on this
  system is materially more urgent to page on than the same failure on a read-only chat assistant.
