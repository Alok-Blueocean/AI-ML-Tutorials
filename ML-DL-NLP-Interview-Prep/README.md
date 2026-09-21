# ML / DL / NLP Interview Prep

A condensed, interview-focused companion to the [ML interview prep doc](https://claude.ai/code/artifact/7159b7da-d42f-4313-bc5a-927ee253b980) built earlier — this is the local, file-based version, scoped down on purpose: short concept notes with real-world examples for every topic, but exercises / references / projects / scenario Q&A only on the topics that actually carry weight in senior ML interviews.

Sibling to `../LLMOps-MLOps-Course/` (same numbered-folder pattern), but each topic here gets at most 5 files instead of that course's 10, and 4 lighter foundational topics get only a `tutorial.md`.

## How this maps to interview importance

**Full kit** (`tutorial.md` + `exercises.md` + `references.md` + `projects.md` + `scenario_qa.md`) — these are the topics search results and interview-question aggregators consistently flag as most-asked:

| # | Topic | Why it's "most important" |
|---|---|---|
| 01 | [Regression & Regularization](01_Regression_and_Regularization/) | Ridge/Lasso tradeoffs and multicollinearity are near-universal screening questions |
| 02 | [Classification Algorithms](02_Classification_Algorithms/) | Logistic regression, SVM kernels, and tree splits come up in almost every ML round |
| 03 | [Model Evaluation & Overfitting](03_Model_Evaluation_and_Overfitting/) | The single most commonly asked practical topic — bias-variance, CV, metric choice under imbalance |
| 04 | [Ensemble Methods](04_Ensemble_Methods_Bagging_Boosting/) | XGBoost/LightGBM internals are a default senior-IC follow-up question |
| 05 | [Neural Network Fundamentals](05_Neural_Network_Fundamentals/) | Backprop, activations, and **architecture design** — includes deep scenario coverage of reducing overfitting and choosing layer depth/width |
| 06 | [CNN & RNN Architectures](06_CNN_and_RNN_Architectures/) | ResNet skip connections, LSTM gating — standard deep-learning-round material |
| 07 | [Attention & Transformers](07_Attention_and_Transformers/) | Self-attention mechanics are now asked even outside dedicated NLP roles |
| 08 | [NLP & LLMs](08_NLP_and_LLMs/) | Embeddings, BERT vs GPT, RAG, LoRA — the current wave of NLP interview questions |
| 09 | [MLOps & Model Deployment](09_MLOps_and_Model_Deployment/) | Drift detection and "great offline, bad in production" is the classic senior differentiator |
| 10 | [ML System Design](10_ML_System_Design/) | The open-ended whiteboard round for senior/staff loops |
| 15 | [Deep Learning MLOps at Scale](15_Deep_Learning_MLOps_at_Scale/) | SageMaker, MLflow for DL, and large-team deployment governance — the senior/staff differentiator on the infra side |

**Brief primer only** (`tutorial.md`, concept + real-world example, no exercises/projects/scenario Q&A — prerequisite or lower interview-frequency material):

| # | Topic |
|---|---|
| 11 | [Python & Statistics Foundations](11_Python_Stats_Foundations/) |
| 12 | [Feature Engineering & Data Preprocessing](12_Feature_Engineering_and_Data_Preprocessing/) |
| 13 | [Unsupervised Learning — Clustering & PCA](13_Unsupervised_Learning_Clustering_PCA/) |
| 14 | [Bias, Fairness & Explainability](14_Bias_Fairness_and_Explainability/) |

## Suggested order

1. **11 → 12** (foundations, skim if already solid)
2. **01 → 04** (classical ML — regression, classification, evaluation/overfitting, ensembles)
3. **13** (unsupervised, brief)
4. **05 → 07** (neural nets → CNN/RNN → attention/transformers)
5. **08** (NLP/LLMs)
6. **09 → 10 → 15** (MLOps, then system design, then DL-specific MLOps at scale — the material that most separates mid-level from senior/staff answers)
7. **14** (bias/fairness, brief — closing sanity check)

## A note on `references.md` accuracy

Every link in a `references.md` file was checked this session via web search, and the load-bearing ones (papers, docs, GitHub repos) were fetched to confirm the content matches the claim. A small number of resources are well-known but weren't individually re-fetched this session (e.g. a textbook with no single canonical URL) — those are explicitly marked `(not URL-verified this session)` inline, with enough detail (author/title) to find them yourself. Nothing was fabricated; treat the flagged items as "very likely real, just double-check the exact link before relying on it."

## Scenario Q&A — what's covered

Per-topic `scenario_qa.md` files use a "Situation → what would you do and why" format. Two areas get explicitly deep treatment across multiple angles, as requested:
- **Reducing overfitting** — see `03_Model_Evaluation_and_Overfitting/scenario_qa.md` (the general toolkit) and `05_Neural_Network_Fundamentals/scenario_qa.md` (neural-net-specific: too little data, over-capacity architecture, diagnosing overfitting vs. a bad validation split).
- **Designing neural network layers/architecture** — see `05_Neural_Network_Fundamentals/scenario_qa.md` (tabular vs. small-image cases, when to stop adding capacity, activation choice per layer).

`09_MLOps_and_Model_Deployment/scenario_qa.md` and `10_ML_System_Design/scenario_qa.md` carry the equivalent depth for production-failure and whiteboard-design scenarios; `15_Deep_Learning_MLOps_at_Scale/scenario_qa.md` covers the SageMaker/MLflow/large-team-deployment angle specifically.

## Update log

- **05_Neural_Network_Fundamentals** was enriched with a practical architecture-design method, Flatten vs. Global Average Pooling, exactly where/how to place regularization (L1/L2, Dropout, BatchNorm), a hyperparameter-role table, full loss-function and activation-function catalogs, and a "which use case → which choice" decision cheat-sheet — plus 10 more scenario Q&As and 13 more design-decision exercises, all on "when to use what."
- **15_Deep_Learning_MLOps_at_Scale** was added new: SageMaker for DL (training, distributed training, endpoints, Pipelines, Model Registry — including the honest note that SageMaker Model Monitor is now closed to new customers), MLflow for DL (autologging, the shift from Staging/Production stages to aliases), DL-specific serving (Triton, TorchServe — flagged as archived/unmaintained since Aug 2025, TensorFlow Serving, ONNX/TensorRT), and large-team governance (registry naming, reproducibility, FinOps, rollback). All references in this folder were individually fetched and verified this session — none needed the "not URL-verified" disclaimer.
