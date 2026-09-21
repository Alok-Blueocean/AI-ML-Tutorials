# Books — Deployment Quality Gates and Release Dashboards

No single book is written specifically about "LLM release gates and Grafana
dashboards" — this is a fast-moving, blog-and-docs-driven practice area in mid-2026.
The books below are the closest durable, high-quality treatments of the underlying
principles (offline/online evaluation design, release engineering, and monitoring
system design) that this module builds on top of.

---

### 1. *Designing Machine Learning Systems* — Chip Huyen (O'Reilly, 2022)
**What it teaches:** Chapter 6 ("Model Development and Offline Evaluation") is the
best available treatment of building a rigorous offline evaluation harness — slice-based
evaluation, evaluating for robustness/fairness/calibration before a model reaches
production, and why a single aggregate accuracy number is an inadequate release
signal. Chapter 7 ("Model Deployment and Prediction Service") covers deployment
patterns (shadow, canary, A/B) that this module's release-gate and rollback logic
depend on. Chapter 8 ("Data Distribution Shifts and Monitoring") is the direct
ancestor of this module's "reading dashboards" section — it explains *why* you need
trend views and per-dimension breakdowns rather than a single up/down health check.
**Difficulty:** Intermediate (assumes working ML/software engineering background,
which this course's readers have).
**Estimated reading time:** Chapters 6–8 together: 4–6 hours.
**Why it matters for this module:** The three-tier evaluation strategy and the
green/yellow/red dashboard framing in this module are a direct extension of Huyen's
offline/online evaluation split and her monitoring chapter, adapted specifically for
LLM-judge-based metrics instead of classical ML metrics. A companion chapter-by-chapter
summary repo is maintained at `github.com/chiphuyen/dmls-book` (see github.md).

---

### 2. *AI Engineering: Building Applications with Foundation Models* — Chip Huyen (O'Reilly, 2024/2025)
**What it teaches:** The direct successor to *Designing Machine Learning Systems*,
focused specifically on LLM/foundation-model applications. Covers evaluation
methodology for generative systems in depth: exact-match vs. semantic-similarity vs.
AI-judge scoring, the calibration problem with LLM judges, and how evaluation
criteria should be tied to product-specific failure modes (format violations,
hallucination, safety) rather than generic benchmark scores.
**Difficulty:** Intermediate to advanced.
**Estimated reading time:** The evaluation-focused chapters: roughly 5–7 hours.
**Why it matters for this module:** This is the most directly relevant book-length
treatment of the "hard blocks for format/hallucination" concept this module's quality
gates section covers, and of why relative gates (regression vs. last known-good)
matter as much as absolute gates (minimum bar) for foundation-model release decisions.

---

### 3. *Accelerate: The Science of Lean Software and DevOps* — Nicole Forsgren, Jez Humble, Gene Kim (IT Revolution Press, 2018)
**What it teaches:** The empirical research behind deployment frequency, lead time,
change failure rate, and time-to-restore as the four key metrics of software delivery
performance — the DORA metrics. Also covers the organizational and technical
practices (trunk-based development, deployment automation, monitoring/observability)
that make frequent, low-risk releases possible.
**Difficulty:** Beginner to intermediate (no ML content — pure software delivery
research).
**Estimated reading time:** 4–5 hours for the full book; the metrics chapters alone
are about 1.5 hours.
**Why it matters for this module:** Release dashboards for LLM systems are, at their
core, an LLM-specific instantiation of DORA's "change failure rate" and "time to
restore" metrics. Understanding the general software-delivery research this module's
release-manager workflow is built on top of makes the LLM-specific version easier to
reason about, and it is standard reference material in senior MLOps/platform
engineering interviews.

---

### 4. *Site Reliability Engineering* — Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy, eds. (O'Reilly / Google, 2016) — free online at sre.google/sre-book
**What it teaches:** Chapters on SLOs, error budgets, monitoring philosophy
(symptoms vs. causes), and the four golden signals (latency, traffic, errors,
saturation). Freely available in full online.
**Difficulty:** Intermediate.
**Estimated reading time:** The monitoring and SLO chapters: about 2–3 hours.
**Why it matters for this module:** The green/yellow/red readiness-signal framing in
this module's dashboard section is a direct descendant of SRE's error-budget and
golden-signals thinking. Understanding error budgets is what lets a release manager
turn "the hallucination rate ticked up 0.3 points" into an actual go/no-go decision
rather than a vague feeling — the same logic used to decide whether to burn an SRE
error budget on a risky release.

---

## What's deliberately not here

There is, as of mid-2026, no widely adopted, durable book specifically covering
"LLM evaluation-as-CI-gate" or "Grafana dashboards for AI observability" — this
practice is documented almost entirely in vendor engineering blogs (Grafana Labs,
Arize, Evidently, promptfoo, Confident AI/DeepEval) and a handful of very recent
arXiv preprints (see references.md), not in book form yet. Treat the references.md
entries as the primary source material for this module and the books above as the
conceptual foundation underneath them.
