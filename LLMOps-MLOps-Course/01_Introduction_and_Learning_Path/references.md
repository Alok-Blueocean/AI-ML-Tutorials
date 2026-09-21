# References — Module 01: Introduction and Learning Path

Official documentation, foundational papers, and engineering blog posts that establish the vocabulary and mental models for the entire course.

---

## Official documentation & industry guides

### Google Cloud — "MLOps: Continuous delivery and automation pipelines in machine learning"
- **URL:** https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning
- **What it teaches:** Defines MLOps as "an ML engineering culture and practice that aims at unifying ML system development and operation," and walks through the maturity levels of MLOps (manual/no automation → automated pipeline → full CI/CD/CT). Introduces the concept of Continuous Training (CT) alongside CI/CD, which is unique to ML systems.
- **Difficulty:** Beginner to Intermediate
- **Reading time:** ~30–40 minutes
- **Why it matters for this module:** This is the single most-cited definitional document for what "MLOps" means and is the best companion to this module's "MLOps vs DevOps" section — it explicitly enumerates what DevOps CI/CD does NOT cover for ML systems (data validation, model validation, retraining triggers).

### Google Cloud — Practitioners Guide to MLOps (whitepaper)
- **URL:** https://cloud.google.com/resources/mlops-whitepaper
- **What it teaches:** A framework-level companion to the above, aimed at practitioners rather than architects — covers organizational and process maturity, not just pipeline mechanics.
- **Difficulty:** Beginner to Intermediate
- **Reading time:** ~45 minutes
- **Why it matters for this module:** Useful for the "who is this course for" framing — it separates the data-science skill set from the engineering skill set this course assumes and builds toward.

### AWS — "What is MLOps?"
- **URL:** https://aws.amazon.com/what-is/mlops/
- **What it teaches:** A vendor-neutral-in-substance primer covering the same core ideas (versioning, CI/CD/CT, monitoring) from AWS's perspective, useful for cross-referencing terminology against Google's framing.
- **Difficulty:** Beginner
- **Reading time:** ~15 minutes

### IBM — "What is MLOps?" and "What is LLMOps?"
- **URLs:** https://www.ibm.com/think/topics/mlops and https://www.ibm.com/think/topics/llm-orchestration
- **What it teaches:** Concise, up-to-date conceptual explainers with clear before/after framing of how LLM-based systems change operational requirements versus classical ML.
- **Difficulty:** Beginner
- **Reading time:** ~10–15 minutes each

### Red Hat — "What is LLMOps?"
- **URL:** https://www.redhat.com/en/topics/ai/llmops
- **What it teaches:** Frames LLMOps explicitly as an extension layer on top of MLOps rather than a replacement, and lists the LLM-specific additions: prompt management, embeddings/vector databases as first-class artifacts, and gateway-level cost/token tracking.
- **Difficulty:** Beginner
- **Reading time:** ~15 minutes
- **Why it matters for this module:** Directly supports the module's core distinction between the classical-MLOps track (Modules 02–06, 19–22) and the LLMOps track (Modules 07–18) of this course.

### roadmap.sh — MLOps Roadmap
- **URL:** https://roadmap.sh/mlops
- **What it teaches:** An interactive, community-maintained checklist of the skills and tools that make up the MLOps field end-to-end: programming fundamentals, version control, cloud platforms, containerization, ML fundamentals, data engineering, CI/CD, orchestration, experiment tracking, model registries, feature stores.
- **Difficulty:** Beginner (as a map; individual linked topics vary)
- **Reading time:** ~20 minutes to skim the full map
- **Why it matters for this module:** A good secondary cross-check against the 26-module roadmap table in this chapter — helps confirm coverage and spot where this course goes deeper or shallower than a generic industry roadmap.

---

## Foundational papers

### "Hidden Technical Debt in Machine Learning Systems"
- **Authors:** D. Sculley, Gary Holt, Daniel Golovin, Eugene Davydov, Todd Phillips, et al. (Google)
- **Venue:** NeurIPS (NIPS) 2015
- **URL:** https://papers.neurips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems.pdf
- **What it teaches:** The original, widely cited argument for why ML systems accrue disproportionate technical debt compared to traditional software — boundary erosion, entanglement ("CACE": Changing Anything Changes Everything), hidden feedback loops, undeclared consumers, and the fact that the actual ML/model code is a small fraction of a real production ML system.
- **Difficulty:** Intermediate (conceptual, not mathematically heavy)
- **Reading time:** ~30–45 minutes
- **Why it matters for this module:** This paper is the intellectual origin point for the entire discipline of MLOps — it's the "why we need this whole 26-module course" argument in a single ten-year-old-but-still-completely-relevant paper. Read this first if you want the deepest possible motivation for the field before touching any tooling.

---

## Engineering blog posts

### Martin Fowler / ThoughtWorks — "Continuous Delivery for Machine Learning" (CD4ML)
- **Authors:** Danilo Sato, Arif Wider, Christoph Windheuser
- **URL:** https://martinfowler.com/articles/cd4ml.html
- **What it teaches:** How to apply Continuous Delivery principles to ML systems, given that ML applications vary along three axes simultaneously — code, model, and data — rather than just code as in traditional software. Covers experiment tracking, model deployment, orchestration, and monitoring/observability as the technical pillars of CD4ML.
- **Difficulty:** Intermediate
- **Reading time:** ~35–45 minutes
- **Why it matters for this module:** This is the clearest practitioner-level bridge between "DevOps as you already know it" and "MLOps as this course teaches it" — directly supports the module's DevOps-vs-MLOps-vs-LLMOps comparison and previews the CI/CD gate concepts formalized in Module 02.

---

## How to use this list
Read the Google Cloud MLOps architecture doc and the Red Hat LLMOps article first (under an hour combined) — they give you the two official definitions this course's terminology is built on. Read the Sculley et al. paper and the CD4ML article this week or next as deeper, unhurried background; both remain the canonical justification for why MLOps/LLMOps exist as disciplines, and neither has been superseded despite the rapid tooling changes between 2015/2019 and today.
