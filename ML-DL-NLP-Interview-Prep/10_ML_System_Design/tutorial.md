# ML System Design — Condensed Study Notes

## How to Structure an Open-Ended ML System Design Answer

A repeatable framework beats improvising:

1. **Clarify objective and constraints.** What business problem, what scale (users, QPS, data volume), what latency budget, what's out of scope. Never start designing before this.

2. **Define the metric.** Business metric (e.g., revenue, fraud loss avoided) vs offline ML metric (e.g., AUC, NDCG) vs online metric (e.g., CTR lift in an A/B test) — state all three and how they relate.

3. **Propose a baseline.** A simple, cheap solution first (heuristic or simple model) — this anchors the conversation and shows judgment before jumping to complexity.

4. **Iterate.** Add complexity where it earns its keep: better features, a stronger model, a second stage, more data.

5. **Evaluate.** Offline metrics, then online A/B testing plan, then a monitoring/retraining plan.

```
clarify -> metric -> baseline -> architecture -> offline eval -> online eval -> monitor/retrain
```

Real-world example: asked to "design a system to detect fake reviews," a strong answer starts with "what does fake mean here — bot-generated, paid, or duplicate content — and what's the cost of a false positive (hiding a real review) vs a false negative," before naming a single model.

## Two-Stage Retrieval-then-Ranking Architecture

Used everywhere at scale: search, recommendations, ads. Problem: you cannot run an expensive, accurate model over millions/billions of candidates within a latency budget.

- **Stage 1 — Retrieval (candidate generation):** cheap, high-recall method narrows millions of items down to hundreds/thousands of plausible candidates. Often a two-tower embedding model (user tower + item tower, dot-product similarity) with approximate nearest neighbor search, or simple heuristics (popular items, collaborative filtering).

- **Stage 2 — Ranking:** an expensive, high-precision model (gradient-boosted trees or a deep network with rich cross-features) scores and orders the smaller candidate set precisely.

```
candidates = retrieve(user_embedding, item_index, top_k=500)   # cheap, ANN search
ranked     = rank(candidates, rich_features_model)             # expensive, precise
final_list = ranked[:20]
```

- Origin reference: Covington et al., "Deep Neural Networks for YouTube Recommendations" (2016) — canonical description of this pattern in production.

- Real-world example: a video platform's retrieval stage narrows 10 million videos to 500 candidates per user request using a two-tower embedding model and approximate nearest-neighbor search in milliseconds; the ranking stage then scores those 500 with a much heavier model to pick the final top-20 shown to the user — running the heavy model on all 10 million videos per request would never fit the latency budget.

## The Cold-Start Problem

- **User cold start:** a new user has no interaction history, so personalization has nothing to learn from.

- **Item cold start:** a new item has no interactions yet, so collaborative signals don't exist for it.

- Mitigations: content-based features (item metadata, user profile attributes) as a fallback until interaction data accumulates; popularity/trending baselines; onboarding flows that elicit explicit preferences; exploration strategies (bandits) that deliberately show new items to a small traffic slice to gather signal.

- Real-world example: a new user on a music app gets recommendations based on 3 genres they picked at signup (content-based) rather than collaborative filtering, since collaborative filtering has zero signal for them on day one.

## Batch vs Real-Time Architecture Tradeoffs

- **Batch:** precompute predictions on a schedule (hourly/nightly). Cheaper, simpler, no latency pressure at request time, but predictions can be stale.

- **Real-time:** compute predictions per request using the freshest features. Handles freshness-sensitive cases (a user's last 2 minutes of clicks) but adds serving complexity, latency budget pressure, and infrastructure cost.

- Most production systems are hybrid: batch-precompute the expensive/stable part, real-time-adjust with recent signals.

- Real-world example: an e-commerce homepage precomputes a personalized "recommended for you" shelf nightly (batch), but re-ranks it in real time based on what the user clicked in the current session, since session-level intent (browsing hiking boots right now) is far more predictive than yesterday's batch job could know.

## Distributed Training: Data Parallelism vs Model Parallelism

- **Data parallelism:** the full model is replicated across multiple devices; each device processes a different data shard and gradients are synchronized/averaged across devices. Used when the model fits on one device but training needs to go faster or use more data.

- **Model parallelism:** the model itself is too large to fit on one device, so different layers/parts of the model are split across devices (or techniques like tensor/pipeline parallelism split individual layers/stages). Used for very large models (e.g., large LLMs).

```
# data parallelism (conceptual)
for shard in split(data, num_devices):
    grads[device] = forward_backward(model_replica[device], shard)
avg_grads = mean(grads across devices)
update(model, avg_grads)   # every replica stays identical

# model parallelism (conceptual): layers 1-20 on GPU0, layers 21-40 on GPU1, ...
```

- Reference: PyTorch Distributed Overview documentation covers both (DDP for data parallelism; FSDP/tensor/pipeline parallel for model-scale parallelism).

- Real-world example: a mid-size recommendation model trains via standard data-parallel DDP across 8 GPUs to cut training time from days to hours; a 70B-parameter LLM instead needs model/tensor parallelism because it simply doesn't fit in one GPU's memory regardless of how much time you're willing to spend.

## Label Scarcity, Label Delay, and Proxy Labels

- Many real-world problems don't have clean, immediate labels: fraud outcomes may take weeks to confirm (chargebacks), churn labels require waiting a full observation window, ad conversions may be delayed.

- **Proxy labels** approximate the true label with something available sooner (e.g., "user didn't open the app in 7 days" as an early churn proxy while the "official" 30-day churn label is still pending), accepting some noise for faster feedback loops.

- Real-world example: a fraud model can't wait 60 days for confirmed chargebacks to retrain — it uses "customer disputed the transaction" or "transaction manually flagged by an analyst" as faster proxy labels, refining against true chargeback outcomes later.

## Worked Framing: Recommendation System

- Objective: maximize long-term engagement/revenue, not just clicks (clickbait optimization is a known failure mode).

- Metric: offline — NDCG/recall@k on held-out interactions; online — A/B test on engagement/revenue with guardrails (diversity, session length).

- Architecture: two-stage retrieval (collaborative filtering / two-tower embeddings) then ranking (gradient boosted trees or deep model with rich features: user history, item metadata, context).

- Key challenges to name: cold start, popularity bias/feedback loops, diversity vs relevance tradeoff, freshness (batch vs real-time features).

## Worked Framing: Search-Ranking System

- Objective: return the most relevant results for a query, fast.

- Metric: offline — NDCG, MRR against human-labeled relevance judgments; online — click-through rate, and more importantly, session success rate (did the user stop searching, i.e., find what they wanted) since raw CTR alone rewards clickbait titles.

- Architecture: query understanding (spell correction, intent classification) -> retrieval (BM25 keyword + dense embedding hybrid) -> ranking (learning-to-rank model using query-document features: text match, popularity, freshness, personalization).

- Key challenges to name: query understanding for ambiguous/short queries, handling long-tail queries with sparse click data, balancing exact-match (SKUs, names) with semantic match.

## Worked Framing: Fraud-Detection System

- Objective: minimize financial loss from fraud while minimizing false positives that block legitimate customers (a real business tradeoff, not a pure accuracy problem).

- Metric: offline — precision/recall and precision-recall AUC (fraud is heavily imbalanced, so plain accuracy is meaningless); online — actual dollar loss prevented vs false-positive customer friction rate.

- Architecture: real-time feature computation (transaction velocity, device/location anomalies) -> a fast model (or two-tier: rules engine for obvious cases, ML model for the rest) scoring within a strict latency budget (checkout can't wait seconds) -> human review queue for borderline scores.

- Key challenges to name: severe class imbalance, label delay (true fraud confirmed weeks later via chargebacks, so proxy labels are needed for fast iteration), adversarial adaptation (fraudsters actively change behavior in response to your defenses, unlike most ML domains), strict latency budget at checkout.

## Quick Gotchas Worth Naming in an Interview

- Never propose an architecture before stating the metric — interviewers specifically probe whether you'd optimize the right thing, and "maximize clicks" is almost always a trap answer for recommendation/search systems.

- Retrieval and ranking should be evaluated on different metrics (recall@k for retrieval, NDCG/precision for ranking) — conflating them hides which stage is actually the bottleneck.

- Class imbalance (fraud, rare-event prediction) makes accuracy meaningless; always reach for precision/recall, PR-AUC, or cost-weighted metrics instead.

- Label delay is not solved by "just wait for the real label" in a live system — proxy labels plus periodic recalibration against true labels is the standard real-world compromise.
