# References — Prompt Lifecycle: Versioning and Statistical Evaluation

Official docs, papers, and engineering blog posts, verified via direct fetch/search at time of writing (mid-2026).

---

## Official Documentation

### MLflow Prompt Registry
- **Link:** https://mlflow.org/docs/latest/genai/prompt-version-mgmt/prompt-registry/
- **What it teaches:** Git-inspired commit-based prompt versioning, immutability of saved versions, alias-based promotion (e.g., point a `production` alias at version 3, later repoint it at version 4 with zero app-code changes), commit messages, tags for lineage (author/task/language), and linking prompt versions to evaluation runs and traces. Includes working code (`mlflow.genai.register_prompt`, `load_prompt`, `search_prompts`).
- **Difficulty:** Intermediate. **Reading time:** ~20-30 min.
- **Related pages:**
  - Prompt version tracking alongside app versions: https://docs.databricks.com/aws/en/mlflow3/genai/prompt-version-mgmt/prompt-registry/track-prompts-app-versions
  - MLflow GenAI evaluation & monitoring overview: https://mlflow.org/docs/latest/genai/eval-monitor/ (evaluation-driven development, datasets/scorers/predict-fn model, LLM-as-judge scorers — note this section does not itself cover A/B/statistical comparison, which is this module's original contribution on top of MLflow's building blocks)

### LangSmith — Manage Prompts (Prompt Hub)
- **Link:** https://docs.langchain.com/langsmith/manage-prompts
- **What it teaches:** Commit-based prompt version history with diff view; two deployment environments (Staging/Production) each with an ordered commit history; tags as human-readable aliases so application code never references raw commit hashes; rollback by picking any prior commit from an environment's history; "Owners only" access control restricting who can promote/tag. Also documents an operational nuance directly relevant to production incident response: tag repointing is instantaneous, but already-running instances may serve a cached prompt version until a TTL (default ~5 minutes) expires.
- **Difficulty:** Beginner → Intermediate. **Reading time:** ~15-20 min.

### scipy.stats.ttest_rel
- **Link:** https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ttest_rel.html
- **What it teaches:** The exact function used in the module's paired-t-test code pattern. Parameters (`a`, `b`, `axis`, `nan_policy`, `alternative`), return object (`statistic`, `pvalue`, `df`, `.confidence_interval()`), and the underlying formula (mean of the differences divided by standard error). Supports NumPy, CuPy, PyTorch, JAX, and Dask arrays.
- **Difficulty:** Beginner (API reference). **Reading time:** ~10 min.

### statsmodels — Power and Sample Size Calculations
- **Link:** https://www.statsmodels.org/stable/stats.html (Power and Sample Size Calculations section)
- **What it teaches:** `TTestPower` / `TTestIndPower` classes for paired and independent-samples t-test power analysis; solving for any one of {effect size, alpha, power, nobs} given the others via `tt_solve_power` / `tt_ind_solve_power`; related classes for ANOVA (`FTestAnovaPower`), proportions/normal approximation (`NormalIndPower`), and chi-square goodness-of-fit (`GofChisquarePower`).
- **Difficulty:** Intermediate (requires basic familiarity with effect size / Cohen's d). **Reading time:** ~30-45 min including working through an example.

### Databricks — Prompt Registry (Azure/AWS/GCP docs mirror)
- **Links:**
  - https://docs.databricks.com/aws/en/mlflow3/genai/prompt-version-mgmt/prompt-registry/
  - https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/prompt-version-mgmt/prompt-registry/
- **What it teaches:** Same MLflow 3 Prompt Registry capability documented above, with added coverage of Unity Catalog governance (access control, audit trail) for enterprise deployments — useful when this module's "immutability guarantees" section needs an enterprise-governance angle (who is allowed to create/promote a version, and how that's audited).
- **Difficulty:** Intermediate. **Reading time:** ~15 min.

### PromptLayer — Prompt Registry & A/B Testing
- **Link:** https://www.promptlayer.com/
- **What it teaches:** A commercial but widely referenced example of "central registry with immutable version history, diffs, release labels, and rollback," plus explicit prompt A/B testing ("release new prompt versions gradually and compare metrics") and a visual editor letting non-engineers iterate on prompts without a redeploy. Good concrete illustration of decoupling prompt changes from application deploys, which the module's registry-design section should call out as a design goal.
- **Difficulty:** Beginner. **Reading time:** ~15 min.

---

## Papers

### The Hitchhiker's Guide to Testing Statistical Significance in Natural Language Processing
- **Authors:** Rotem Dror, Gili Baumer, Segev Shlomov, Roi Reichart
- **Venue:** ACL 2018
- **Link:** https://aclanthology.org/P18-1128/
- **What it teaches:** A rigorous protocol for choosing the right significance test for a given NLP evaluation setup, and — notably — an empirical audit of ACL/TACL 2017 papers showing that significance testing in NLP research is "frequently ignored or misused." Directly relevant background for why the module insists on a paired test (not an unpaired one) and a fixed n rather than an ad hoc sample.
- **Difficulty:** Advanced (academic paper, assumes stats background). **Reading time:** ~45-60 min for the core sections.

### Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena
- **Authors:** Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, et al. (LMSYS / UC Berkeley)
- **Venue:** NeurIPS 2023 (Datasets and Benchmarks track)
- **Link:** https://arxiv.org/abs/2306.05685
- **What it teaches:** Foundational analysis of using strong LLMs as automatic judges of other LLMs' outputs — quantifies biases (position bias, verbosity bias, self-enhancement bias, limited reasoning) and shows GPT-4-based judging reaches over 80% agreement with human preference, comparable to human-human agreement. Directly relevant to the "correctness" metric in the module's delta report: in most real pipelines, "correctness" for open-ended generation is itself produced by an LLM judge, so its own noise/bias needs to be understood before trusting a paired t-test built on top of it.
- **Difficulty:** Advanced (research paper, but readable methodology sections). **Reading time:** ~40 min.

---

## Engineering Blog Posts

### Your AI Product Needs Evals
- **Author:** Hamel Husain
- **Link:** https://hamel.dev/blog/posts/evals/
- **What it teaches:** A three-level framework for LLM evaluation maturity: Level 1 (fast/cheap unit-test assertions run on every change), Level 2 (human + model/LLM-judge evaluation with tight feedback loops on trace data), Level 3 (A/B testing, reserved for mature products validating real user impact). Also warns against naive agreement metrics on imbalanced data (use precision/recall instead).
- **Difficulty:** Intermediate. **Reading time:** ~20-25 min.
- **Why it matters here:** Gives the module's three transcripts a clean maturity model — Part 1 (logging) and Part 2 (delta gates) map to Husain's Level 1/2, and Part 3 (statistical A/B promotion) is explicitly his Level 3, "reserved for mature products."

### Patterns for Building LLM-based Systems & Products
- **Author:** Eugene Yan
- **Link:** https://eugeneyan.com/writing/llm-patterns/
- **What it teaches:** Seven recurring patterns in production LLM systems (evals, RAG, fine-tuning, caching, guardrails, defensive UX, feedback collection). The "Evals" section specifically critiques classic NLP metrics (BLEU/ROUGE) for poor correlation with human judgment and argues for task-specific eval datasets plus LLM-based reference-free evaluation (G-Eval-style).
- **Difficulty:** Intermediate. **Reading time:** ~35-40 min full article; ~10 min for the Evals section alone.
- **Why it matters here:** Reinforces why the module's delta report must be multi-metric and task-specific rather than a single generic accuracy score, and previews the LLM-as-judge approach underpinning most modern "correctness" scoring.

### It's All A/Bout Testing: The Netflix Experimentation Platform
- **Publisher:** Netflix Technology Blog
- **What it teaches:** How Netflix architected a company-wide, self-service A/B testing platform — experiment configuration, metrics computation pipelines, and the org-level discipline of "everything is an experiment" applied at large scale. A useful real-world analog for scaling a prompt A/B framework beyond a single script to a company-wide service other teams can self-serve.
- **Difficulty:** Intermediate. **Reading time:** ~15-20 min.
- **Note:** Accessed via netflixtechblog.com (Medium-hosted); if the direct link redirects, search "It's All A/Bout Testing Netflix Experimentation Platform" — it is a well-known, frequently cited post in the experimentation-platform literature.

---

## Suggested reading path

1. **MLflow Prompt Registry** docs — see the concrete mechanics of an immutable, versioned, alias-promoted registry (Part 1).
2. **LangSmith Manage Prompts** docs — compare a second real implementation, note the cache-TTL rollback nuance (Part 1 / rollback semantics).
3. **Hamel Husain — Your AI Product Needs Evals** — get the maturity-model framing before diving into automation (bridges Part 1 → Part 2).
4. **Eugene Yan — Patterns for Building LLM Systems**, Evals section — reinforce multi-metric, task-specific evaluation (Part 2).
5. **scipy.stats.ttest_rel** + **statsmodels power** docs — the actual API surface for Part 3's code.
6. **Dror et al., Hitchhiker's Guide to Testing Statistical Significance in NLP** — the academic rigor behind why test choice and sample size matter (Part 3).
7. **Zheng et al., Judging LLM-as-a-Judge** — understand the noise/bias in the "correctness" score itself before trusting a t-test built on it (Part 3, advanced/optional).
8. **Netflix — It's All A/Bout Testing** — see the same ideas at company-wide platform scale (optional, systems-design extension).
