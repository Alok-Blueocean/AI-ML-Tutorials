# Data Layer: Feature Stores and Versioning

## What This Is About

Every ML or LLM system is only as good as the data feeding it. This module looks at the "data layer" of MLOps — the tools and practices that make data and features **traceable, reproducible, and reusable** across a team. Two problems dominate this space:

1. **Dataset versioning** — knowing exactly which version of your data produced which model.
2. **Feature stores** — sharing and reusing computed features consistently between training and production (serving).

We'll cover three widely used tools that represent each part of this landscape: **DVC** and **LakeFS** (data/dataset versioning), and **Feast** (feature store).

## Why It Matters

Code is versioned with Git almost by default now. Data usually isn't — and that's where things quietly break:

- A model performs well in a notebook, but nobody can reproduce it six months later because the training CSV was overwritten.
- A feature is computed one way during training (in a batch job) and slightly differently during real-time serving (in an API), causing **training-serving skew** — a silent source of bad predictions.
- Two data scientists on the same team each recompute the same expensive feature in incompatible ways.
- An auditor or regulator asks "what data trained this model?" and nobody has a clean answer.

Without a data layer strategy, ML pipelines look reproducible on paper (code is versioned, experiments are logged) but are actually unreproducible in practice because the data underneath keeps shifting. This matters even more for LLM pipelines, where "data" includes fine-tuning sets, retrieval corpora, and embeddings that also need versioning.

## Main Concepts in Plain Terms

### Dataset Versioning

The core idea: treat large data files like you treat code — with commits, history, and the ability to roll back or branch. But Git itself is bad at handling large binary files (images, parquet files, model weights), so specialized tools exist.

**DVC (Data Version Control)**
- Works alongside Git: your Git repo stores small pointer files (metadata, hashes), while the actual large data lives in cheap storage (S3, GCS, Azure Blob, or even a local drive).
- You get familiar commands — `dvc add`, `dvc push`, `dvc pull`, `dvc checkout` — that mirror Git, so a data change is just another commit.
- It also supports lightweight **pipelines** (`dvc.yaml`), letting you version the steps that turn raw data into features or a trained model, and re-run only the steps whose inputs changed.
- Mental model: "Git for data, using Git for the bookkeeping."

**LakeFS**
- Gives you Git-like branching, committing, and merging directly **on top of your data lake** (e.g., an S3 bucket), without duplicating the data.
- You can create a "branch" of your entire dataset (petabytes, conceptually), experiment or run a pipeline against it, and merge or discard changes — all through cheap, copy-on-write style operations, not physical copies.
- Great fit when many teams read/write a shared data lake and you need isolation ("let me test my ETL job on a branch without touching production data") plus rollback if a bad pipeline run corrupts data.
- Mental model: "Git for your entire data lake, as an infrastructure layer."

**DVC vs. LakeFS, in short:** DVC is lightweight and file/repo-centric, ideal for individual projects or small teams tracking specific datasets alongside code. LakeFS operates at the infrastructure level, versioning the whole data lake for larger organizations with heavier data engineering needs. They solve the same underlying problem — reproducible, rollback-able data — at different scales.

### Feature Stores

A **feature** is a computed input to a model — e.g., "average order value in the last 30 days" for a customer. Computing this correctly and consistently is surprisingly hard once you have many models and many engineers.

A **feature store** is a central system that:
- **Computes features once** (batch or streaming) and stores them so they aren't recalculated inconsistently by every team.
- Serves the **same feature values** to both training (usually from an "offline store" — a data warehouse or files) and real-time inference (from an "online store" — a fast key-value store like Redis or DynamoDB), preventing training-serving skew.
- Adds a registry/catalog so features are discoverable and reusable — instead of every project quietly reinventing "average order value."
- Often tracks point-in-time correctness — pulling the feature value as it was known *at the time*, not the current value, to avoid leaking future information into training data.

**Feast** is a popular open-source feature store that plugs into your existing infrastructure (it doesn't force you into a specific database). You define features in code, Feast keeps offline and online stores in sync, and both your training script and your production API pull features through the same interface.

## A Simple Example (Feast-style feature definition)

```python
# feature_repo/features.py
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32
from datetime import timedelta

customer = Entity(name="customer_id")

order_stats_source = FileSource(
    path="data/order_stats.parquet",
    timestamp_field="event_timestamp",
)

order_stats_fv = FeatureView(
    name="order_stats",
    entities=[customer],
    ttl=timedelta(days=90),
    schema=[Field(name="avg_order_value_30d", dtype=Float32)],
    source=order_stats_source,
)
```

Once registered (`feast apply`), both a training pipeline and a live prediction service can request `avg_order_value_30d` for a given `customer_id` and get consistent, point-in-time-correct values — one from the offline store for historical training data, one from the online store for low-latency serving.

## Key Takeaways / Best Practices

- **Version your data, not just your code.** If you can't answer "what data trained this model?", you can't truly reproduce or debug it.
- **Match the tool to the scale.** DVC for project-level dataset tracking with Git-like simplicity; LakeFS when you need Git-like operations across an entire shared data lake.
- **Use a feature store to kill training-serving skew.** The same feature-computation logic and values should feed both training and production.
- **Think in offline vs. online stores.** Offline for large historical training data, online for fast single-record lookups at inference time — a feature store keeps both consistent.
- **Watch out for point-in-time correctness.** Naively joining "current" feature values into historical training data leaks future information and inflates model performance.
- **Catalog features for reuse.** A registry prevents duplicated effort and encourages standardized, well-understood features across teams.
- **For LLM pipelines, extend the same thinking to text/embeddings.** Version your fine-tuning datasets and retrieval corpora just as rigorously as you would tabular features.
