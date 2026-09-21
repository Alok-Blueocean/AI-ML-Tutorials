# Recommended Books — Experiment Tracking and MLflow Deep Dive

## Designing Machine Learning Systems
**Author**: Chip Huyen — O'Reilly Media, 2022
**What it teaches**: Chapter 6 ("Model Development and Offline Evaluation") contains a section specifically on **experiment tracking and versioning** — it explains *why* experiment tracking exists as a discipline (the combinatorial explosion of code + data + hyperparameter + environment combinations once a team has more than one person iterating on a model) before naming any specific tool. Huyen deliberately treats MLflow, W&B, and Comet as interchangeable implementations of the same underlying need, which is exactly the framing this module wants you to internalize before you get attached to one tool's UI.
**Difficulty**: Intermediate–Advanced (assumes you already know basic ML training loops; the value is in the systems-design reasoning, not code samples)
**Estimated reading time**: ~45–60 minutes for the relevant section; the book as a whole is a multi-week read
**Why it matters for this module**: Gives the conceptual "why" layer underneath the mechanical "how do I call `mlflow.log_metric()`" content — read this section either right before or right after the hands-on MLflow walkthrough so the code has a purpose beyond "the tutorial told me to."

## Practical MLOps: Operationalizing Machine Learning Models
**Authors**: Noah Gift and Alfredo Deza — O'Reilly Media, 2021
**What it teaches**: Dedicated, code-heavy coverage of MLflow as part of a broader MLOps toolchain — setting up tracking servers, logging runs, and integrating experiment tracking into CI-driven training pipelines (connecting back to this course's earlier CI/CD modules). Less theory than Huyen's book, more "here is the exact command/config."
**Difficulty**: Intermediate
**Estimated reading time**: 2–3 hours for the MLflow-focused chapters
**Why it matters for this module**: Bridges this module to modules 01–06 (CI/CD, versioning, reproducibility) by showing experiment tracking wired into an actual pipeline rather than run ad hoc from a notebook — reinforces that experiment tracking is a production engineering concern, not a data-scientist convenience feature.

## Machine Learning Engineering with MLflow
**Author**: Natu Lauchande — Packt Publishing, 2021
**What it teaches**: An MLflow-first (rather than MLflow-as-one-tool-among-many) book: Tracking, Projects, Models, and Model Registry each get sustained treatment, plus deployment patterns and a chapter on running MLflow at team/organization scale. The most MLflow-specific book on this list.
**Difficulty**: Intermediate
**Estimated reading time**: 3–4 hours for the Tracking/Registry chapters most relevant to this module
**Why it matters for this module**: If you want a single-source deep dive that mirrors this chapter's own structure (Tracking → Projects → Models → Registry, in that order), this book is the closest published match. Treat the 2021 edition's Model Registry chapter as describing the older stage-based (`Staging`/`Production`/`Archived`) workflow — cross-check against the mid-2026 official docs in `references.md`, which now recommend tags + aliases instead.

## Introducing MLOps
**Authors**: Mark Treveil et al. (Dataiku team) — O'Reilly Media, 2020
**What it teaches**: A management/organizational-lens overview of MLOps, including a chapter framing experiment tracking as a governance and collaboration problem (who ran what, when, with which data) rather than purely a technical one — useful context for the "tagging strategy" content in this module, since tagging conventions are as much a team-process decision as a technical one.
**Difficulty**: Beginner (deliberately light on code; concept-first)
**Estimated reading time**: 30–40 minutes for the relevant chapter
**Why it matters for this module**: A good "why does my team need a tagging convention at all" primer before the hands-on `compare_runs.py` and tagging-strategy sections — read this if you're the engineer who'll be proposing the tagging convention to a team, not just implementing one.
