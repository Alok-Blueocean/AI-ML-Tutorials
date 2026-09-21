# Books — Reproducibility and Environment Management

---

### Designing Machine Learning Systems
**Author:** Chip Huyen — O'Reilly Media, 2022
**Relevant chapters:** Ch. 6 ("Model Development and Offline Evaluation" — experiment tracking/versioning) and Ch. 10 ("Infrastructure and Tooling for MLOps" — dev/staging/prod environments, containers)
**Difficulty:** Intermediate
**Estimated reading time:** 2–3 hours for the two relevant chapters (full book: 12–15 hours)
**What it teaches:** How versioning experiments, code, data, and models together enables reproducibility and fair comparison between runs; why infrastructure (compute, storage, orchestration) needs to be treated as a first-class, versionable layer rather than an afterthought. Huyen frames reproducibility not as a nice-to-have but as the precondition for trustworthy iteration — you cannot know if a change helped unless you can hold everything else constant.
**Why it matters for this module:** This is the book that most directly informs the mental model behind "the reproducibility paradox" — the harder ML systems are to reproduce (data, hardware, stochastic training, dependency sprawl), the more essential reproducibility tooling becomes, not less. It also gives the conceptual vocabulary (data versioning, model versioning, environment versioning) that this module operationalizes with Docker/Conda/lockfiles.
**Companion resource:** Official chapter summaries and code notes — https://github.com/chiphuyen/dmls-book

---

### Practical MLOps: Operationalizing Machine Learning Models
**Authors:** Noah Gift and Alfredo Deza — O'Reilly Media, 2021
**Relevant chapters:** Ch. 1 ("Introduction to MLOps") and the containerization/CI chapters covering Docker-based deployment across cloud providers
**Difficulty:** Beginner → Intermediate
**Estimated reading time:** 1.5–2 hours for the relevant chapters
**What it teaches:** A hands-on, tool-forward tour of building reproducible, deployable ML pipelines across AWS, Azure, and GCP, with an emphasis on treating a "reproducible GitHub project" (pinned dependencies + Dockerfile + CI config committed together) as the base unit of shippable ML work.
**Why it matters for this module:** Directly reinforces the "pin-and-hash-everything" philosophy as an industry-standard expectation, not a personal preference — and shows what a reproducible project *looks like* end-to-end rather than only in isolated snippets. Good practical counterweight to the more conceptual "Designing Machine Learning Systems."

---

### The Twelve-Factor App (free, online)
**Author:** Adam Wiggins / Heroku engineering — https://12factor.net
**Relevant sections:** Factor II (Dependencies), Factor III (Config), Factor X (Dev/prod parity)
**Difficulty:** Beginner
**Estimated reading time:** 30–45 minutes for the three relevant factors (full document: ~1 hour)
**What it teaches:** The foundational, platform-agnostic argument for explicitly declaring and isolating dependencies, separating config from code, and minimizing the gap between development and production environments. Though written for web apps, its reasoning is the direct ancestor of "don't rebuild in each environment, promote the artifact forward."
**Why it matters for this module:** This is the conceptual root of environment-promotion discipline. Reading Factor X in particular before the promotion.yaml pipeline section makes the "promote the artifact, don't rebuild" principle feel like the ML-specific instance of a much older, well-tested software engineering rule rather than an ML-specific invention.

---

### Improving Reproducibility in Machine Learning Research (NeurIPS 2019 Reproducibility Program Report)
**Authors:** Joelle Pineau et al. — published in JMLR, 2021 (arXiv:2003.12206)
**Format:** Research paper, not a book, but included here as required foundational reading
**Difficulty:** Intermediate (accessible to engineers, written for a research audience but not math-heavy on this topic)
**Estimated reading time:** 45–60 minutes
**What it teaches:** A rigorous, empirical account of why ML experiments fail to reproduce in practice — undocumented hyperparameters, unpinned library versions, unspecified hardware/random-seed effects, and missing code. Introduces the NeurIPS reproducibility checklist that shaped how the ML research community now reports methodology.
**Why it matters for this module:** Gives the empirical evidence behind the "reproducibility paradox" claim in the source transcript — this isn't a hypothetical problem, it's a measured, widespread failure mode in published ML research, which is exactly why production ML teams need enforced tooling (lockfiles, pinned images, promotion gates) rather than relying on discipline alone.
