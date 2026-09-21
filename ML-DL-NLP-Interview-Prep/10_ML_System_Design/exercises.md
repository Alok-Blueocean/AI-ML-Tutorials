# ML System Design — Exercises

Ordered easy to hard. Mix of hands-on tasks and open-ended design questions (the norm for this topic).

1. **Frame a problem.** Take "design a system to recommend friends on a social network" and write down: the objective, 3 candidate offline metrics, and 3 candidate online metrics, before proposing any model.

2. **Baseline first.** For the friend-recommendation problem above, propose the simplest possible baseline (no ML) and explain what it would get right and wrong.

3. **Conceptual: retrieval vs ranking.** Explain why you cannot just run your best ranking model directly over the entire item catalog for every request, using concrete numbers (e.g., catalog size, per-item scoring cost, latency budget).

4. **Build a toy two-stage system.** Using a public dataset (e.g., MovieLens), implement a simple two-tower retrieval model (or just cosine similarity over precomputed embeddings) to get top-200 candidates, then a gradient-boosted tree ranker to reorder them. Measure recall@200 for retrieval and NDCG@10 for the final ranking.

5. **Conceptual: cold start.** For a new podcast app with zero listening history for new users, propose two concrete mitigations for user cold start and two for item (new podcast) cold start.

6. **Design a batch vs real-time split.** For a food-delivery app's "estimated delivery time" feature, decide which parts of the computation should be batch-precomputed and which must be real-time, and justify each choice.

7. **Distributed training decision.** Given a model that is 500M parameters and a model that is 70B parameters, explain which parallelism strategy (data vs model/tensor/pipeline) you'd reach for in each case and why.

8. **Conceptual: proxy labels.** For a churn-prediction system where "true churn" is only confirmed after 90 days of inactivity, propose a proxy label usable within 14 days, and explain what bias this proxy might introduce.

9. **Design a search-ranking system.** Full walkthrough: query understanding, retrieval (name the hybrid approach), ranking features, offline metric, online metric, and one failure mode you'd specifically monitor for.

10. **Design a fraud-detection system.** Full walkthrough: objective and business tradeoff, why accuracy is the wrong metric, feature types needed in real time vs batch, latency constraint, and how you'd handle label delay.

11. **Design a recommendation system end-to-end.** Full walkthrough for a video-streaming homepage: objective (careful — not just "maximize clicks"), two-stage architecture, cold start handling, and one concrete guardrail metric to prevent an engagement-optimizing model from degrading long-term user trust (e.g., over-recommending outrage content).

12. **Handling adversarial drift.** For the fraud-detection system in exercise 10, explain why "concept drift" here is fundamentally different from concept drift in, say, demand forecasting, and what monitoring/retraining cadence that difference implies.

13. **Scale estimation.** For a search system handling 50,000 queries per second globally with a 100ms end-to-end latency budget, sketch a rough latency budget breakdown across query understanding, retrieval, and ranking, and identify where you'd need approximate/pre-computed shortcuts.

14. **Multi-objective ranking.** Design a ranking system that must balance relevance, diversity, and business revenue (e.g., sponsored placement) in a single ranked list. Propose one concrete way to combine these objectives (e.g., a weighted scoring function or a constrained optimization) and explain the tradeoff.

15. **Full mock interview.** Pick one of: "design YouTube's recommendation system," "design Google's autocomplete," or "design an ATM/credit-card fraud detector," and write a complete structured answer (objective -> metric -> baseline -> architecture -> evaluation -> monitoring) in the same depth as the worked examples in tutorial.md, without referring back to them.
