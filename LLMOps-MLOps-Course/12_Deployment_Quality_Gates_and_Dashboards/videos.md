# Videos — Deployment Quality Gates and Release Dashboards

Curated talks and channel content for evaluation-triggered CI/CD, release gates, and
LLM observability dashboards. Official vendor/engineering channels are prioritized over
generic tutorial creators. Durations are only listed where confirmed; several MLOps
Community pages do not publish an explicit runtime, so that field is omitted rather than
guessed.

---

### 1. How to Systematically Test and Evaluate Your LLM Apps
**Creator/Channel:** MLOps Community (home.mlops.community / MLOps.community YouTube)
**Speakers:** Gideon Mendels (CEO & Co-founder, Comet) with Demetrios Brinkmann
**Difficulty:** Intermediate
**Rating:** ★★★★★

Why it's worth watching: This is the closest publicly available talk to the module's
core thesis — that LLM apps need the same "define metric → build test suite → gate
before deploy" discipline as traditional software, adapted for non-deterministic
outputs. Mendels walks through three concrete metric families (deterministic
assertions, heuristic/embedding-distance metrics, and LLM-as-judge) and explicitly
frames the problem as "is this app ready to be deployed" — i.e., release readiness,
not just offline accuracy.

Complements: Part 1 (evaluation triggers) and Part 3 (reading dashboards) of this
module — it bridges the "what to measure" question with "how do you decide go/no-go."

---

### 2. Evaluating LLM-Judge Evaluations: Best Practices
**Creator/Channel:** MLOps World (mlopsworld.com talk library)
**Difficulty:** Advanced
**Rating:** ★★★★☆

Why it's worth watching: The module's quality-gate section leans heavily on
LLM-as-judge scores as an input to release thresholds. This talk addresses the
meta-problem directly: how do you know your judge is trustworthy enough to gate a
release on? It covers judge-vs-human agreement calibration, which is the exact
diligence a release manager should demand before wiring a judge score into a hard
CI block.

Complements: Part 2 (quality gates and thresholds) — specifically the discussion of
why gate thresholds must be validated against human-labeled data before they are
trusted to block a deploy.

---

### 3. Reliable LLM Products, Fueled by Feedback
**Creator/Channel:** MLOps Community (podcast/video)
**Difficulty:** Intermediate
**Rating:** ★★★★☆

Why it's worth watching: Focuses on closing the loop between production feedback
(the "data-update" trigger in this module's four-trigger model) and evaluation
datasets. Useful for understanding why a schedule-only or PR-only eval strategy is
insufficient once real users are generating edge cases your golden set doesn't cover.

Complements: Part 1 (the data-update trigger and why golden datasets must evolve).

---

### 4. Grafana: How to Monitor LLMs in Production (companion walkthrough content)
**Creator/Channel:** Grafana Labs (official engineering blog has an accompanying
recorded webinar/demo referenced from grafana.com/blog — check the Grafana Labs
YouTube channel for the current "AI Observability" webinar recording, as Grafana
periodically re-records these as the GenAI dashboards ship new panels)
**Difficulty:** Intermediate
**Rating:** ★★★★☆

Why it's worth watching: Grafana Labs' own demonstrations of the prebuilt "GenAI
Observability" and "GenAI Evaluations" dashboards (built on OpenLIT + OpenTelemetry
GenAI semantic conventions) are the most direct, vendor-authoritative visual
reference for the dashboard-design part of this module — panel layout, the
hallucination/bias/toxicity summary panel, and alert-rule wiring on cost and latency
thresholds.

Complements: Part 3 (reading evaluation dashboards) and the expanded Grafana
dashboard-design section of this module.

---

### 5. Arize Phoenix / Arize AI — LLM Evaluation in CI/CD (Arize AI YouTube channel)
**Creator/Channel:** Arize AI (official channel covers their Phoenix OSS evaluation
library and experiments API in product walkthroughs and conference talks)
**Difficulty:** Intermediate
**Rating:** ★★★☆☆

Why it's worth watching: Arize's public content demonstrates the "create dataset →
define task → build evaluator → assert threshold → wire into GitHub Actions" pattern
this module's release-gate code example follows almost verbatim (their reference
blog uses `experiment.get_evaluations()["score"].mean() > 0.8` as a literal gate
condition). Good for seeing the pattern implemented against a real experiment-tracking
backend rather than a toy script.

Complements: Part 2 (the release-gate code example) and the GitHub Actions
integration section.

---

## Notes on searching for more

Search current content on the **MLOps Community** YouTube channel and
**home.mlops.community/public/videos** for the latest "eval-gates" and "release
readiness" talks — this space moves fast and MLOps Community consistently publishes
the highest-signal practitioner talks on evaluation-in-CI first. Also check the
**Grafana Labs** YouTube channel directly for the most recent GenAI dashboard demo,
since Grafana's AI observability panels have shipped multiple revisions since early
2025 and the specific demo video changes with each major dashboard release.
