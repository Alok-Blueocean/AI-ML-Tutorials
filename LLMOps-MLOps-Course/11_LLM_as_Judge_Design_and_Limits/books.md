# Books — Module 11: LLM-as-Judge: Evaluator Design and Limits

There is not yet a book dedicated solely to LLM-as-judge design — the field is too new and moves too fast for that. Instead, this module is best supported by chapters inside two broader, well-regarded books, plus one long-form community reference that functions like a book chapter in depth and rigor.

---

### 1. *AI Engineering: Building Applications with Foundation Models* — Chip Huyen (O'Reilly, 2025)
- **What it teaches:** This is the closest thing to a canonical textbook treatment of LLM evaluation as of 2025-2026. Huyen (previously known for "Designing Machine Learning Systems") devotes substantial material to evaluation methodology for foundation-model applications: comparing exact-match/statistical metrics against model-based (LLM-as-judge) evaluation, discussing the failure modes of using an LLM to grade another LLM's output, and framing evaluation as a first-class systems-design problem rather than an afterthought. It sits one level above this module — the module's transcripts (judge prompt design, composite scoring, bias/calibration) are the hands-on implementation of the evaluation philosophy this book lays out.
- **Difficulty:** Intermediate — assumes working familiarity with LLMs and production systems, does not require deep ML theory.
- **Estimated reading time:** The evaluation-focused chapters run several hours; the whole book is a multi-day read.
- **Why it matters for this module:** Gives the "why" behind everything the transcripts teach as "how." Read this to understand why the industry converged on LLM-as-judge as a pragmatic (if imperfect) middle ground between cheap-but-shallow statistical metrics and expensive-but-slow human review — context that makes the rest of the module's design choices (CoT-first prompts, explicit rubrics, temperature=0) feel motivated rather than arbitrary.

### 2. *Designing Machine Learning Systems* — Chip Huyen (O'Reilly, 2022)
- **What it teaches:** Predates the LLM-as-judge boom, but its chapters on monitoring, model evaluation in production, and the general discipline of "how do you know your model is actually good" establish the systems-thinking foundation this module builds on: the idea that evaluation is a continuous production concern with thresholds, alerts, and review triggers — not a one-time offline benchmark.
- **Difficulty:** Intermediate.
- **Estimated reading time:** 1-2 hours for the evaluation/monitoring chapters specifically.
- **Why it matters for this module:** The composite-scoring and review-trigger material in this module (weighted score, per-dimension thresholds, routing low-confidence cases to a human) is a direct descendant of the "monitoring and continual learning" mindset this book teaches for classical ML — this module shows what that mindset looks like once the model under evaluation is itself a generative LLM.

### 3. "Your AI Product Needs Evals" and the companion "Evals FAQ" — Hamel Husain and Shreya Shankar (hamel.dev)
- **What it teaches:** Functions as a de facto book chapter: a long-form, rigorously argued reference covering error analysis, building an LLM-as-judge from scratch (100+ labeled examples, iterative critique-and-fix cycles), calibrating a judge against human raters with Cohen's kappa / inter-annotator agreement, and the organizational discipline of keeping a "benevolent dictator" domain expert in the loop rather than diffusing judgment across too many annotators.
- **Difficulty:** Intermediate to Advanced — written for practitioners already building LLM products, not for beginners.
- **Estimated reading time:** 25-40 minutes for the FAQ; the companion essay is a similar length.
- **Why it matters for this module:** This is arguably the single most load-bearing piece of writing in the current (2025-2026) LLM evaluation community, and it maps almost one-to-one onto this module's scope: judge design, calibration against humans, and the human-escalation pattern the transcripts describe with `escalator.py`. Treat it as required reading alongside the video course in `videos.md`.

---

## A note on freshness

This module sits in a genuinely fast-moving part of the field. As of mid-2026, no single-author print book has yet caught up fully with the current generation of practices (panel-of-judges ensembling, agentic trajectory evaluation, judge-time compute scaling via libraries like Verdict). The framework documentation in `references.md` and `github.md` is, in practice, more current than any book on this specific sub-topic — treat the books above as giving you the durable mental models, and the docs/papers as giving you the current state of the art.
