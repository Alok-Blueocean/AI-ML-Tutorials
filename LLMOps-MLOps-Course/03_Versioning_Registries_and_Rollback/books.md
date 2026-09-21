# Books — Module 03: Versioning, Registries, and Rollback

## Designing Machine Learning Systems — Chip Huyen (O'Reilly, 2022)
- **Relevant chapters:** Ch. 6 ("Model Development and Offline Evaluation") and Ch. 9 ("Continual Learning and Test in Production") are the most relevant; versioning and reproducibility concerns are threaded throughout the book.
- **What it teaches:** How to think about reproducibility as a first-class engineering requirement — why a "model" is meaningless without the exact code, data, and config that produced it (the same idea the source transcripts call the "deployment triple"). Also covers shadow deployment, canary release, and A/B testing as the production-facing counterpart to registry promotion.
- **Difficulty:** Intermediate.
- **Estimated reading time:** ~3–4 hours for the two most relevant chapters; ~10–12 hours cover to cover.
- **Why it matters for this module:** This is the closest thing the field has to a canonical textbook, and it's the book most senior MLOps interviewers assume you've read. Its framing of "the model is a function of code + data + config, all three must be versioned together" is the direct intellectual ancestor of the deployment-triple concept this module teaches. Companion repo of chapter summaries: https://github.com/chiphuyen/dmls-book and a community-written detailed summary: https://github.com/serodriguez68/designing-ml-systems-summary.

## Introducing MLOps — Mark Treveil et al. (O'Reilly, 2020, Dataiku-sponsored)
- **What it teaches:** A business- and process-oriented view of the ML lifecycle, including model registries, governance, and monitoring as organizational functions, not just tooling. Good for understanding *why* enterprises mandate registries (audit, compliance, handoff between data science and ops teams) rather than just *how* to click through one.
- **Difficulty:** Beginner–Intermediate (light on code, heavy on process).
- **Estimated reading time:** ~4–5 hours.
- **Why it matters for this module:** Balances this module's heavy engineering content (MLflow API calls, rollback scripts) with the organizational "why" — useful for the interview-readiness angle of the course, since senior MLOps interviews often probe governance/compliance reasoning, not just tool syntax.

## Machine Learning Design Patterns — Valliappa Lakshmanan, Sara Robinson, Michael Munn (O'Reilly, 2020)
- **Relevant pattern chapters:** "Reproducibility" design patterns section (covers Transform, Repeatable Splitting, Bridged Schema) and the "Responsible AI" section touching on model versioning for fairness audits.
- **What it teaches:** Concrete, named design patterns for reproducibility problems — e.g., how to version a preprocessing transform so that training-time and serving-time featurization never drift apart, which is a common silent cause of "the model regressed after promotion" incidents.
- **Difficulty:** Intermediate.
- **Estimated reading time:** ~2 hours for the relevant pattern chapters.
- **Why it matters for this module:** Gives you the vocabulary ("Transform pattern", "Repeatable Splitting") that shows up in senior-level system design interviews when discussing why a promoted model regressed in production despite passing offline evaluation.

## Building Machine Learning Powered Applications — Emmanuel Ameisen (O'Reilly, 2020)
- **Relevant chapters:** Ch. 10–11 on deploying and monitoring models.
- **What it teaches:** A practitioner's narrative of shipping ML into production end-to-end, including what breaks when you don't version models/data together, and pragmatic monitoring signals that should gate a promotion decision.
- **Difficulty:** Beginner–Intermediate.
- **Estimated reading time:** ~2 hours for the relevant chapters.
- **Why it matters for this module:** Very readable, concrete complement to the more abstract "Designing Machine Learning Systems" — good if you want a second, more narrative explanation of why promotion gates and rollback plans matter before you dive into the MLflow code walkthrough.

## Hidden Technical Debt in Machine Learning Systems — D. Sculley et al. (NeurIPS 2015 paper, treated here as required reading alongside the books)
- **URL:** https://papers.nips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems
- **What it teaches:** The foundational argument for why ML systems accumulate a distinct kind of technical debt — entanglement, undeclared consumers, unstable data dependencies — and why versioning + freezing critical data/model artifacts is one of the few effective mitigations.
- **Difficulty:** Intermediate (short paper, ~6 pages of dense content).
- **Estimated reading time:** ~30–45 minutes.
- **Why it matters for this module:** It's the paper every senior MLOps engineer is expected to have read at least once; its "CACE principle" (Changing Anything Changes Everything) is the theoretical justification for why you need a deployment triple, a registry, and a rollback plan rather than treating a model file as an isolated artifact.
