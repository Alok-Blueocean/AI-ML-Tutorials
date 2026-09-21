# MLOps and Model Deployment — References

All links below were checked this session (fetched and content verified against the claim) unless explicitly marked otherwise.

## Book

- Chip Huyen, *Designing Machine Learning Systems* (O'Reilly, 2022) — the standard reference for this whole topic; covers data, training, deployment, and monitoring for production ML. Companion summaries/resources repo: https://github.com/chiphuyen/dmls-book

## Official Docs / Whitepapers

- MLflow official documentation (experiment tracking, model registry). https://mlflow.org/docs/latest/
- DVC (Data Version Control) official repository and docs. https://github.com/iterative/dvc (note: this repository now resolves under the `treeverse` GitHub organization — same project, ownership/hosting moved; the link still works) — docs at https://doc.dvc.org/start
- Google Cloud, "MLOps: Continuous delivery and automation pipelines in machine learning" — defines the MLOps maturity levels (manual, pipeline automation, CI/CD automation) referenced in the tutorial. https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning
- PyTorch, general model export/serving background if optimizing for deployment: ONNX official site and docs (not URL-verified this session — https://onnx.ai/, well-known project, verify before distributing widely).

## GitHub Repositories

- `evidentlyai/evidently` — open-source ML/LLM observability library with 100+ built-in metrics; strong for data drift detection and monitoring reports/dashboards. https://github.com/evidentlyai/evidently
- `GokuMohandas/Made-With-ML` — free, extensive project-based MLOps course covering the full lifecycle (data, training, tracking, serving, CI/CD, monitoring) with real code. https://github.com/GokuMohandas/Made-With-ML

## Course

- Made With ML by Goku Mohandas (Anyscale) — project-based MLOps course, same material as the GitHub repo above, hosted at https://madewithml.com/courses/mlops/ (not URL-verified this session; the GitHub repo above was verified and links to the same course).

## Articles / Interview Prep

- DataCamp, "Top MLOps Interview Questions and Answers." https://www.datacamp.com/blog/mlops-interview-questions
- GrowthBook, "Why your AI model performs great offline but fails in production" — directly relevant to the "offline vs production" scenario question. https://www.growthbook.io/insights/why-your-ai-model-performs-great-offline-but-fails-production

## Notes on What to Prioritize

Interview questions consistently center on: data drift vs concept drift (and how you'd detect each without waiting for ground-truth labels), the offline-metric-looks-great-but-production-degrades scenario, canary/shadow/A-B deployment tradeoffs, and rollback design. Be ready to name concrete tools (MLflow, DVC, Evidently) and explain what problem each one actually solves, not just recite the name.
