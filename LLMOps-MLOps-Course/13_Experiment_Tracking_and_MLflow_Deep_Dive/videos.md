# Recommended Videos — Experiment Tracking and MLflow Deep Dive

All titles and URLs below were confirmed to exist via live search in July 2026. This sandbox's network blocks direct requests to youtube.com, so exact runtimes and some channel attributions could not be independently re-verified by fetching the video page itself — where that applies it's called out explicitly rather than guessed. Watch on YouTube directly to confirm current runtime before class use.

---

### ★★★★★ MLflow on Databricks End-to-End Tutorial — Experiments, Registry, Serving, Nested Runs
- **URL**: https://www.youtube.com/watch?v=9AenofD8GZ8
- **Difficulty**: Intermediate–Advanced
- **Why it's worth watching**: This is the single best video match for this module's scope — it walks the full MLflow lifecycle in one sitting: creating experiments, logging nested runs (parent/child runs, useful for hyperparameter sweeps or multi-step pipelines), comparing runs in the UI, registering the winning model, and serving it. Nested runs in particular are a pattern this module's `compare_runs.py` content builds on.
- **Complements**: The "MLflow Tracking + Model Registry deep dive" and "promoting the best model through the registry" sections of this chapter.

### ★★★★☆ Track Your ML Experiments with MLflow — Quick Tutorial
- **URL**: https://www.youtube.com/watch?v=Z4unUK0vn4k
- **Difficulty**: Beginner
- **Why it's worth watching**: A fast, focused walkthrough of the core `mlflow.log_param()` / `log_metric()` / `log_artifact()` loop and the Tracking UI — the right video to watch first if you've never opened `mlflow ui` before.
- **Complements**: The "logging experiments/params/metrics/artifacts" code section early in this chapter.

### ★★★☆☆ MLflow Experiment Tracking with Autologging in Databricks
- **URL**: https://www.youtube.com/shorts/puctvj_I7Mc
- **Difficulty**: Beginner
- **Why it's worth watching**: A short-form demo of `mlflow.autolog()` — worth a couple of minutes to see how much MLflow captures automatically for common frameworks with a single line, before you learn manual logging.
- **Complements**: The autologging vs. manual-logging comparison in the Tracking section.

### ★★★★☆ Welcome to Weights & Biases — Introduction Walkthrough
- **URL**: https://www.youtube.com/watch?v=91HhNtmb0B4
- **Creator/Channel**: Weights & Biases (official channel — see channel link below)
- **Difficulty**: Beginner
- **Why it's worth watching**: The official W&B product walkthrough — dashboard tour, run comparison, and the visualization panels (including parallel coordinates) that this module's source transcripts reference directly.
- **Complements**: The "W&B parallel-coordinates views" section of this chapter and the MLflow-vs-W&B comparison table.

### ★★★☆☆ Weights & Biases End-to-End Demo
- **URL**: https://www.youtube.com/watch?v=tHAFujRhZLA
- **Difficulty**: Intermediate
- **Why it's worth watching**: A longer, applied demo showing W&B across a realistic training loop rather than just the dashboard tour — useful for seeing how config/tags map to comparable runs in practice, which is the same tagging-strategy problem this chapter tackles for MLflow.
- **Complements**: The tagging-strategy and run-comparison sections.

### ★★★☆☆ Weights & Biases: Quick Start Tutorial (Log Metrics & Models Fast)
- **URL**: https://www.youtube.com/watch?v=iq8NFthBffM
- **Creator/Channel**: Community MLOps-focused channel (title includes "Master MLOps" branding; independent creator, not the official W&B channel)
- **Difficulty**: Beginner
- **Why it's worth watching**: A minimal, code-first "first W&B run in five minutes" walkthrough — good as a companion to the official intro video above if you want to see someone type the actual `wandb.init()` / `wandb.log()` calls line by line.
- **Complements**: The Python logging code sections of this chapter.

### ★★★☆☆ Weights & Biases — Official YouTube Channel
- **URL**: https://www.youtube.com/channel/UCBp3w4DCEC64FZr4k9ROxig
- **Difficulty**: Beginner–Advanced (mixed)
- **Why it's worth watching**: Not a single video but the channel itself — browse it for sweeps (hyperparameter search), Tables (interactive data/prediction inspection), and reports content that goes beyond what's covered in this chapter, if you want to go deeper into W&B specifically after finishing this module.
- **Complements**: Optional deep-dive material beyond the chapter's W&B coverage.

---

## University course video (bonus, advanced)

### ★★★★★ Full Stack Deep Learning — "Experiment Management" Lab (Lab 4)
- **Course page**: https://fullstackdeeplearning.com/course/2022/
- **Difficulty**: Advanced
- **Why it's worth watching**: FSDL (taught by ML engineers/founders, not a random creator) has a lecture + lab specifically titled **Experiment Management**, described as: "We run, track, and manage model development experiments with Weights & Biases." It sits inside a broader "Development Infrastructure & Tooling" lecture that positions experiment tracking as one piece of a larger production ML tooling stack — exactly the framing this module needs before diving into MLflow specifics.
- **Complements**: The chapter's opening motivation on *why* experiment tracking matters at production scale, before the MLflow-specific mechanics.
