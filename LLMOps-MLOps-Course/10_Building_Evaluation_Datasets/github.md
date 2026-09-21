# GitHub Repositories — Building Evaluation Datasets

All repositories below were verified to exist and were checked for current star counts / descriptions as of July 2026 (see notes on volatility below — star counts move continuously, treat them as an approximate popularity tier rather than exact figures at the time you read this).

---

## confident-ai/deepeval
- **URL:** https://github.com/confident-ai/deepeval
- **Purpose:** "The LLM Evaluation Framework" — a Pytest-like framework for evaluating LLM applications, with first-class support for `Golden` and `EvaluationDataset` objects, synthetic dataset generation (the `Synthesizer`), and 30+ built-in metrics (G-Eval, DAG, RAG metrics, agent metrics, safety metrics).
- **Popularity tier:** Very popular / widely adopted (~17k stars at time of writing) — one of the two dominant open-source LLM-eval frameworks alongside Ragas.
- **Why it matters:** This is the module's primary reference implementation for dataset *formats*. Its `Golden` schema (`input`, `expected_output`, `context`, `expected_tools`, `additional_metadata`, `custom_column_key_values`) is a well-designed template for what fields a production gold-set row needs, and its `add_goldens_from_csv_file` / `add_goldens_from_json_file` / `push(alias=...)` methods are a concrete pattern for the "dataset as a versioned, shareable artifact" idea this module teaches.
- **How it relates to this module:** Directly informs the "dataset formats compatible with DeepEval" deliverable — build your gold_set JSON/CSV export to match the `Golden` schema so it loads into DeepEval without transformation.

---

## explodinggradients/ragas
- **URL:** https://github.com/explodinggradients/ragas
- **Purpose:** An evaluation toolkit purpose-built for RAG (and increasingly agent) applications, providing both LLM-based and traditional metrics plus an automated **testset generation** pipeline that synthesizes question/context/ground-truth triples from a document corpus using an evolutionary ("Evol-Instruct"-inspired) generation strategy.
- **Popularity tier:** Very popular / widely adopted (~15k stars) — the standard choice specifically for RAG-pipeline evaluation datasets.
- **Why it matters:** Ragas's synthetic testset generator is the most credible reference for the "synthetic" category in the ~20% edge-case allocation from Part 2 of this module — it shows a production-grade pattern for generating hard, multi-hop, and conditional questions programmatically rather than by hand, which is useful once your hand-labeled gold set has covered the obvious cases.
- **How it relates to this module:** Use Ragas's question/contexts/ground_truth schema as the second target format (alongside DeepEval's `Golden`) when designing an export format for your gold_set table — a dataset that satisfies both schemas simultaneously can be evaluated with either tool without a rewrite.

---

## iterative/dvc
- **URL:** https://github.com/iterative/dvc
- **Purpose:** "Git for data" — a CLI (and now also a VS Code extension) for versioning large data files, datasets, and models alongside Git, using lightweight `.dvc` pointer files that reference content-addressed storage in S3/GCS/Azure/SSH.
- **Popularity tier:** Very popular / widely adopted (~15.8k stars), long-standing industry-standard tool for ML data versioning (in use well before the LLM era and still the default choice in mid-2026 for teams that want data versioning without adopting a full feature-store/lakehouse platform).
- **Why it matters:** This is the concrete tool behind the module's "version evaluation datasets like code" workflow. The canonical pattern — `dvc add data/`, `git add data.dvc .gitignore`, `git commit`, `git tag v1.0`, then `git checkout v1.0 && dvc checkout` to reproduce any historical version of the gold set — is exactly the discipline you want so that "which gold set did this eval run against" is always an answerable, reproducible question, mirroring the versioning practices from Module 03 applied specifically to evaluation data.
- **How it relates to this module:** Directly implements Part 3's DVC-style versioning requirement; tag every gold-set freeze the same way you'd tag a model or a code release.

---

## HumanSignal/label-studio
- **URL:** https://github.com/HumanSignal/label-studio
- **Purpose:** An open-source, multi-type data labeling and annotation platform supporting text, image, audio, and time-series data, with multi-annotator workflows, pre-labeling via connected ML models, and structured export formats.
- **Popularity tier:** Very popular / widely adopted (~28k stars) — one of the most widely used open-source annotation tools across the ML industry generally, not LLM-specific.
- **Why it matters:** When Part 1's "two domain experts label independently, then compute Cohen's kappa" workflow needs to scale beyond a shared spreadsheet, Label Studio is the standard open-source tool for running that process with proper task assignment, blind double-labeling, and agreement export — it directly operationalizes the inter-annotator-agreement requirement.
- **How it relates to this module:** Use it (or an equivalent internal tool) as the labeling front-end that feeds your gold_set SQL table; its export format (JSON with per-annotator judgments) is a natural input to a Cohen's kappa computation step in your pipeline.

---

## promptfoo/promptfoo
- **URL:** https://github.com/promptfoo/promptfoo
- **Purpose:** A CLI/library for evaluating and red-teaming LLM applications via declarative YAML/JSON test configs, with built-in red-teaming/vulnerability scanning and side-by-side multi-provider comparison.
- **Popularity tier:** Very popular / widely adopted (~23.8k stars).
- **Why it matters:** Its red-teaming module ships with pre-built adversarial payload datasets (jailbreaks, prompt injection, OWASP LLM Top-10 categories) — a ready-made source of adversarial test cases for the "adversarial" category in Part 2's sampling taxonomy, rather than hand-writing every adversarial probe from scratch.
- **How it relates to this module:** A practical source of pre-built adversarial/edge-case seed data to blend into your own sampler.py output, and a second CI-friendly eval runner (alongside DeepEval) that can consume a gold-set dataset directly.

---

## Giskard-AI/giskard
- **URL:** https://github.com/Giskard-AI/giskard
- **Purpose:** An open-source Python library for testing and evaluating agentic/LLM systems: scenario-based evaluation checks, an automated vulnerability scanner ("Giskard Scan") for prompt injection and OWASP LLM Top-10 issues, and (in its earlier v2 line) explicit bias and performance-issue detection plus RAG evaluation (RAGET).
- **Popularity tier:** Popular / actively used (~5.7k stars), smaller than DeepEval/Ragas but notable specifically for its bias/vulnerability-scanning angle.
- **Why it matters:** Directly relevant to Part 3 (avoiding bias in test data) — Giskard's scanner is one of the few open-source tools that operationalizes automatic detection of certain bias/robustness issues in a model+dataset pair, complementing the manual 5-point pre-freeze audit checklist this module teaches with an automated second pass.
- **How it relates to this module:** Use as an automated cross-check after your manual bias audit — if Giskard's scanner flags an issue your checklist missed, that is a signal your gold set has a blind spot.

---

## great-expectations/great_expectations
- **URL:** https://github.com/great-expectations/great_expectations
- **Purpose:** A general-purpose data-quality/validation framework ("unit tests for your data") that lets you declare expectations about a dataset's shape, distribution, and content, and get automated validation reports.
- **Popularity tier:** Very popular / widely adopted (~11.7k stars), the de facto standard for data-quality validation across the broader data-engineering industry (not LLM-specific).
- **Why it matters:** Rather than manually eyeballing your gold set's category distribution before a freeze, you can encode the 80/20 normal/edge-case split, the category stratification targets, and label-distribution constraints as Great Expectations "expectation suites" that run automatically in CI — turning the pre-freeze bias-audit checklist from Part 3 into an executable, repeatable gate instead of a manual review.
- **How it relates to this module:** A natural implementation target for making the 5-point pre-freeze audit checklist a CI-enforced gate rather than a manual review step, tying back into the CI/CD foundations from Modules 01–02.

---

## chiphuyen/aie-book
- **URL:** https://github.com/chiphuyen/aie-book
- **Purpose:** Official companion repository for Chip Huyen's *AI Engineering* (O'Reilly, 2025) book — chapter summaries, notes, and supplementary material (not a heavy code-tutorial repo).
- **Popularity tier:** Popular, tied to a bestselling recent book; growing steadily since the book's 2025 release.
- **Why it matters:** A free, quick-reference companion to the book chapter cited in `books.md`; useful to skim the chapter-summaries file for the exact structure of the book's evaluation and dataset-engineering content before deciding which sections to read in full.
- **How it relates to this module:** Supporting reference material for the conceptual foundations underlying this module's more tactical, pipeline-oriented content.

---

## openai/evals
- **URL:** https://github.com/openai/evals
- **Purpose:** OpenAI's framework and open registry of benchmarks for evaluating LLMs and LLM-based systems, supporting both no-code (YAML+JSONL) eval definitions and fully custom Python eval logic, with private/custom evals kept local while public ones can be contributed to the shared registry.
- **Popularity tier:** Very popular / widely adopted (~19.1k stars) — one of the earliest and most foundational open-source eval frameworks, predating the current wave of LLM-specific eval tooling.
- **Why it matters:** Its JSONL-based eval-data format (each line a `{"input": [...], "ideal": "..."}` style record) is one of the earliest standardized "gold set" row formats in the LLM ecosystem and is still a useful lowest-common-denominator export target if you need your dataset to be consumable by the broadest range of tools.
- **How it relates to this module:** A third reference schema (alongside DeepEval's `Golden` and Ragas's question/context/ground_truth) worth being aware of when deciding on your own gold_set export format — many teams' internal formats are direct descendants of this one.

---

## Notes on volatility
Star counts for all repositories above change daily; treat the figures as a rough popularity tier captured at time of writing (July 2026), not an exact live count. Verify current numbers directly on GitHub before quoting them in any external-facing material.
