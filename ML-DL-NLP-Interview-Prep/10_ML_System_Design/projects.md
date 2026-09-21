# ML System Design — Projects

## Small: MovieLens Two-Stage Recommender

Build a two-stage recommendation pipeline on the public MovieLens dataset: a retrieval stage using collaborative-filtering embeddings or a simple two-tower model to generate top-200 candidates per user, and a ranking stage using a gradient-boosted tree model with richer features (genre match, recency, popularity) to produce a final top-10 list. Evaluate with recall@200 at the retrieval stage and NDCG@10 at the ranking stage, separately. This proves you understand why systems are split into stages and can measure each stage on its own appropriate metric, rather than treating recommendation as a single monolithic model.

## Medium: Search Ranking with Hybrid Retrieval

Using a public document/passage dataset (e.g., a subset of MS MARCO or a self-scraped small corpus with hand-labeled query-relevance pairs), build a search system combining BM25 keyword retrieval and dense embedding retrieval, merged via reciprocal rank fusion, followed by a learning-to-rank model trained on query-document features. Evaluate with NDCG and MRR against your labeled relevance judgments, and specifically test a handful of exact-match queries (product codes, exact titles) to confirm hybrid retrieval doesn't regress on cases pure BM25 would nail. This proves you can implement the retrieval-then-ranking pattern for search specifically, including the hybrid-retrieval nuance interviewers frequently probe.

## Large: Fraud-Detection System with Label-Delay Simulation

Build a fraud-detection pipeline on a public imbalanced transaction dataset (e.g., a Kaggle credit-card-fraud dataset), simulating the real-world constraint that true labels (confirmed chargebacks) arrive weeks after the transaction. Implement: a fast model usable within a tight simulated latency budget, a precision/recall-focused evaluation (not accuracy, given class imbalance), a proxy-label strategy for faster feedback (e.g., "flagged by rules engine" as an early proxy vs "confirmed chargeback" as the true delayed label), and a simple concept-drift simulation where you inject a change in fraud patterns partway through and show model performance degrading, then demonstrate a retraining response. This proves you can reason about the full set of constraints that make fraud detection uniquely hard among ML system design problems: class imbalance, label delay, adversarial drift, and strict latency.
