# Books & Book Chapters — Building Evaluation Datasets

---

## AI Engineering: Building Applications with Foundation Models
**Author:** Chip Huyen (O'Reilly, 2025)

- **What it teaches:** Chapter 4 ("Evaluate AI Systems") and the later "Dataset Engineering" chapter are the two most relevant sections. Chapter 4 covers evaluation criteria design, AI-as-judge methodology, and why evaluation is "the single most critical and difficult part of AI engineering." The dataset engineering material covers how much data you actually need, how to validate data quality, and de-duplication/leakage concerns — directly relevant to the "how big should my gold set be" and "avoid benchmark contamination" questions this module answers.
- **Difficulty:** Intermediate (assumes working knowledge of LLM application architecture; no ML theory prerequisites).
- **Estimated reading time:** ~2–3 hours for the evaluation chapter alone; the book totals roughly 500 pages.
- **Why it matters for this module:** This is the closest thing the field has to a canonical, up-to-date (2025) textbook treatment of evaluation-dataset construction for LLM systems, written by a practitioner with production experience at NVIDIA, Snorkel, and multiple startups. It gives the conceptual vocabulary (what a "golden" is, why AI-judge evaluation needs its own held-out validation set) that the more tactical sampler.py / SQL-query material in this module builds on.
- **Companion repo (free, official):** https://github.com/chiphuyen/aie-book — chapter summaries, notes, and supporting material; explicitly not a code-heavy tutorial repo, but useful for a table-of-contents-level map of what's covered where.

---

## Designing Machine Learning Systems
**Author:** Chip Huyen (O'Reilly, 2022)

- **What it teaches:** Chapter 6 ("Model Development and Offline Evaluation") covers train/validation/test splitting discipline, slice-based evaluation (evaluating performance on specific subgroups/categories — the direct ancestor of "stratified sampling by category" in this module), and the perturbation/invariance testing techniques that map onto adversarial and edge-case test design. Chapter 4 covers labeling, weak supervision, and data quality issues (label bias, sampling bias) that predate the LLM-specific version of the same problems.
- **Difficulty:** Intermediate/Advanced.
- **Estimated reading time:** ~2 hours per relevant chapter.
- **Why it matters for this module:** Pre-dates the LLM wave but is where much of the "classic MLOps" vocabulary for data slicing, sampling bias, and evaluation-set discipline that this module adapts to LLM gold sets originates. Reading it clarifies which parts of this module's practices are LLM-specific innovations versus decades-old ML evaluation discipline applied to a new substrate.

---

## Reliable Machine Learning: Applying SRE Principles to ML in Production
**Authors:** Todd Underwood, Cathy Chen, Kranti Parisa, Niall Richard Murphy, D. Sculley, et al. (O'Reilly, 2022)

- **What it teaches:** Applies Google's SRE discipline to ML systems, including chapters on monitoring for training/serving skew and data quality gates. Useful background on treating evaluation-data quality as an operational SLO rather than a one-time exercise — reinforces the "version your eval dataset like code, re-audit on every freeze" discipline in Part 3 of this module.
- **Difficulty:** Intermediate/Advanced.
- **Estimated reading time:** ~1.5 hours for the most relevant chapters.
- **Why it matters for this module:** Provides the operational-rigor mindset (treat data quality regressions like production incidents) that motivates why this module insists on a formal pre-freeze bias-audit checklist rather than an informal "looks fine" sign-off.

---

## Practical guidance (long-form web "book chapters" in lieu of a dedicated book)

There is not yet a single dedicated book chapter solely on "building LLM evaluation datasets" as a standalone published work as of mid-2026 — the field is young enough that the best long-form treatments are living documents rather than bound books:

- **Hamel Husain, "LLM Evals: Everything You Need to Know" (hamel.dev)** — functions as a book-chapter-length reference (frequently updated) covering sample sizes (100+ trace review as a starting heuristic, saturation-based stopping rules), the five sampling strategies for selecting traces to review (random, clustering, extremes, classifier-flagged, feedback-based), and the case for a single expert labeler vs. multi-rater kappa-based agreement. Read this alongside the book chapters above as the most current, practitioner-tested counterpart. https://hamel.dev/blog/posts/evals-faq/
- **"AI Evals for Engineers & PMs" (Hamel Husain & Shreya Shankar, Maven cohort course)** — not a book, but structured like one (has a syllabus, private community, and per-week reading/lab assignments); widely cited as the most field-tested, up-to-date curriculum on this exact topic, reportedly used by teams at OpenAI, Anthropic, and Google. https://maven.com/parlance-labs/evals

**Difficulty:** Intermediate. **Estimated reading time:** the Hamel FAQ alone runs to a multi-hour read given its length and depth; treat it as a reference to dip into per-topic rather than a linear read.
