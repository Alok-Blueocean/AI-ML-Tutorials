# Videos — Building Evaluation Datasets

Curated external video resources to accompany this module. Ranked by relevance and quality for engineers building production evaluation datasets (production-log sampling, edge-case/adversarial sampling, gold labeling, and bias auditing).

---

### ⭐⭐⭐⭐⭐ AI Evaluations Crash Course (Real Example)
- **Creator/Channel:** Hamel Husain (independent ML/LLM consultant, formerly GitHub/Airbnb; co-creator of the "AI Evals for Engineers & PMs" course)
- **Approx. duration:** ~50 minutes
- **Difficulty:** Intermediate
- **Why it's worth watching:** Walks through building an evaluation set for a *real* AI agent starting from a spreadsheet of production transcripts — exactly the "production-log-to-gold-set" workflow this module teaches. Demonstrates error analysis (open coding → axial coding) as the precursor to deciding what categories to stratify your sample by, which is the missing intuition most teams skip before they start labeling.
- **Complements:** Part 1 (production-log sampling, stratification by category, gold-set construction).
- Reference article/companion write-up: https://creatoreconomy.so/p/ai-evaluations-crash-course-in-50-minutes-hamel-husain

---

### ⭐⭐⭐⭐⭐ How to Construct Domain-Specific LLM Evaluation Systems
- **Creator/Channel:** Hamel Husain & Emil Sedgh — YouTube (hosted talk/interview format)
- **Difficulty:** Intermediate/Advanced
- **URL:** https://www.youtube.com/watch?v=eLXF0VojuSs
- **Why it's worth watching:** Focuses on the hard part senior engineers actually get paid for: building evaluation criteria and gold labels that are specific to a company's domain rather than generic benchmark metrics. Directly informs the "two domain experts + Cohen's kappa" gold-labeling workflow and why a single "benevolent dictator" labeler is sometimes preferred in practice for consistency, with the trade-offs made explicit.
- **Complements:** Part 1 (gold labels, domain-expert labeling, inter-annotator agreement).

---

### ⭐⭐⭐⭐ Automated Testing for LLMOps (short course)
- **Creator/Channel:** DeepLearning.AI, taught by Rob Zuber (CTO, CircleCI)
- **Approx. duration:** ~1 hour 12 minutes (6 video lessons + 4 hands-on code labs)
- **Difficulty:** Beginner/Intermediate
- **Why it's worth watching:** An official DeepLearning.AI short course that walks through building rules-based and model-graded evaluation pipelines, running them in CI, and covering common failure classes (hallucination, data drift, harmful output) with runnable code labs. Good structural companion to the CI/CD-integrated dataset versioning material in this module since it shows where the eval dataset plugs into a pipeline, not just how to build it.
- **Complements:** Part 1 and Part 3 (running gold-set evaluation as a CI gate; connects back to Modules 01–02 CI/CD foundations).
- URL: https://www.deeplearning.ai/short-courses/automated-testing-llmops/

---

### ⭐⭐⭐⭐ LLM Red Teaming: The Complete Step-by-Step Guide (companion video/blog series)
- **Creator/Channel:** Confident AI (makers of DeepEval)
- **Difficulty:** Intermediate/Advanced
- **Why it's worth watching:** Explains the manual-vs-automated adversarial testing trade-off and how to build a systematic library of adversarial prompts (jailbreaks, prompt injection, safety probes) rather than ad hoc "try to break it" sessions. Maps directly onto the "adversarial category" (~20% edge-case allocation) from Part 2 of this module.
- **Complements:** Part 2 (adversarial/failure-driven sampling categories).
- URL: https://www.confident-ai.com/blog/red-teaming-llms-a-step-by-step-guide

---

### ⭐⭐⭐ Hamel Husain — YouTube channel (ongoing talks/podcast appearances)
- **Creator/Channel:** Hamel Husain
- **Difficulty:** Intermediate
- **Why it's worth watching:** A running channel of talks and interview appearances on applied LLM evaluation practice (error analysis, eval taxonomies, working with domain experts, avoiding "vibes-based" evaluation). Useful to browse for the newest talk once a specific one referenced above ages out — the practices (error analysis before metrics, spreadsheet-first workflows) remain stable even as specific videos are replaced.
- **Complements:** Part 1, Part 3 (ongoing practitioner perspective on dataset quality and bias).
- URL: https://www.youtube.com/@hamelhusain7140

---

## Notes on sourcing
All videos/courses above were verified to exist via search as of July 2026. Where an exact runtime was not independently confirmable, it has been omitted rather than guessed. Hamel Husain and Shreya Shankar's paid Maven cohort course ("AI Evals for Engineers & PMs") is referenced in `books.md`/`references.md` rather than here since it is a paid, non-video-platform course rather than a freely watchable video.
