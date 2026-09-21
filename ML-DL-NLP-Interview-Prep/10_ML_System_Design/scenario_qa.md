# ML System Design — Scenario-Based Q&A

**Situation (full design walkthrough): "Design a recommendation system for a video-streaming homepage."** What would you do and why?

Model answer (structured):
- **Objective and constraints:** Clarify first — is the goal engagement (watch time), retention, or revenue (ads/subscriptions)? Assume long-term engagement without degrading content diversity or trust (avoid pure clickbait optimization). Scale: tens of millions of users, catalog of hundreds of thousands of videos, homepage must render in under ~200ms.
- **Metric:** Offline — recall@k at retrieval, NDCG@k at ranking against held-out watch history. Online — A/B test on watch time and session return rate, with a guardrail metric on content diversity to catch a model that over-narrows recommendations.
- **Baseline:** Popularity-based "trending now" list — cheap, immediately deployable, sets the floor any ML system must beat.
- **Architecture:** Two-stage. Retrieval stage: a two-tower embedding model (user tower encodes watch history/profile, item tower encodes video metadata) with approximate nearest-neighbor search, narrowing the catalog to ~500 candidates. Ranking stage: a model with richer cross-features (user-item interaction features, recency, session context) scoring and ordering the final list. Batch-precompute user/item embeddings nightly; re-rank in real time using current-session signals (what the user clicked in the last 10 minutes).
- **Cold start:** New users get content-based recommendations from onboarding preferences; new videos get boosted via exploration (small forced impressions) until enough interaction data accumulates.
- **Evaluation and monitoring:** Roll out via shadow deployment, then canary, then A/B test before full rollout. Monitor prediction/engagement distribution for drift and set an automatic rollback trigger if watch time or diversity guardrail metrics drop sharply post-deployment.

---

**Situation:** You're asked to design a search-ranking system for an e-commerce site and the interviewer pushes back: "why not just optimize for click-through rate directly?" What would you do and why?

Model answer: Explain that CTR alone rewards clickbait-style titles/thumbnails and can be gamed by sensational-but-irrelevant results, without reflecting whether the user actually found what they wanted. Propose session success rate (did the user complete a purchase or stop searching, indicating satisfaction) as a better proxy alongside CTR, and use human-labeled relevance judgments (NDCG/MRR) offline as a metric CTR can't corrupt. Frame the final answer as a multi-metric evaluation: offline relevance judgment, plus online CTR and conversion, monitored together so no single gameable metric drives the whole system.

---

**Situation:** Your recommendation system's retrieval stage returns a candidate set that the ranking stage consistently scores low, hurting overall recommendation quality. How would you diagnose and fix this?

Model answer: This is a retrieval recall problem — the ranking stage can only be as good as the candidates it receives. Measure recall@k of the retrieval stage against a held-out set of items the user actually engaged with, independent of the ranking stage's score. If recall is low, the retrieval embedding model or ANN index (e.g., approximation quality, stale embeddings) is the actual bottleneck, not the ranker. Fix by improving retrieval-stage training data/features, increasing candidate set size (with a latency-budget tradeoff), or refreshing embeddings more frequently, and re-measure both stages independently rather than tuning the ranker to compensate.

---

**Situation:** A fraud-detection model interviewer scenario: "Your model's offline precision-recall AUC looks great, but the finance team says actual dollar losses from fraud haven't gone down since deployment." What would you do and why?

Model answer: Point out the mismatch between the ML metric and the business metric — precision-recall AUC treats all frauds as equal, but dollar loss is dominated by a small number of high-value fraud cases. Check whether the model is catching many small, cheap frauds while missing the rare large ones, and consider a cost-weighted evaluation metric that reflects actual dollar impact, not just detection rate. Also verify the decision threshold/policy (what score triggers a block vs a review) is actually tuned against expected loss, not just against a generic F1-style tradeoff.

---

**Situation:** You need to decide whether a new large ranking model should be trained with data parallelism or model parallelism, and the team is unsure which to reach for. What would you do and why?

Model answer: The deciding question is whether the model fits in a single device's memory. If yes, use data parallelism (replicate the model, shard the data, synchronize gradients) — it's simpler to implement and sufficient for most ranking models, which are typically far smaller than large language models. Only reach for model/tensor/pipeline parallelism if the model itself (parameters + activations + optimizer states) exceeds single-device memory, since that adds real engineering complexity (communication overhead between split model parts) that isn't worth it unless memory forces the issue.

---

**Situation:** Six months after launch, your recommendation system's engagement metrics have plateaued, and a stakeholder suspects the model has converged onto a narrow set of "safe" popular recommendations, reducing content discovery. What would you do and why?

Model answer: This is a classic feedback-loop / popularity-bias problem — the model learns from data generated by its own past recommendations, so it reinforces whatever it already recommends and starves less-popular items of the interaction data needed to ever surface. Diagnose by measuring recommendation diversity/coverage over time (are fewer unique items receiving most impressions than 6 months ago). Fix with deliberate exploration (bandit-style forced impressions for under-exposed items), diversity-aware re-ranking (penalize over-concentration in the final list), and adding a diversity guardrail metric to the standard evaluation so this doesn't silently recur.

---

**Situation:** A junior engineer proposes using standard random k-fold cross-validation to evaluate a new demand-forecasting model. What would you do and why?

Model answer: Flag this as a temporal leakage risk — random k-fold CV can train on future data and validate on past data, giving an optimistic and misleading estimate of real-world performance, since forecasting is inherently time-ordered. Recommend a time-based split instead (e.g., rolling-origin / walk-forward validation, training only on data before each validation window), which correctly simulates the real deployment condition of only ever having past data available to predict the future.

---

**Situation:** You're asked "design Google's autocomplete/search-suggestion feature" and need to handle both scale and personalization. What would you do and why?

Model answer: Frame the core constraint as extreme latency sensitivity (must respond within tens of milliseconds per keystroke) at massive scale, which rules out any heavy per-request model. Architecture: a precomputed trie/prefix-index of popular queries for the base candidate generation (near-instant lookup), personalized re-ranking layered on top using lightweight, precomputed user-context features (recent searches, location) rather than a full model inference per keystroke. Offline metric: whether the eventually-chosen query appeared and how high it ranked among suggestions (MRR-style). Online metric: suggestion acceptance rate and overall search latency, with strict monitoring since latency regressions here are immediately visible to every user on every keystroke.
