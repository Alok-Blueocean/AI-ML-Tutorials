# Videos — Module 08: LLM Latency, Cost, and Deployment Strategy

A note on this list: video platforms were not directly reachable from the research environment used to compile this module, so titles/durations below are limited to talks whose existence could be confirmed through the hosting organization's own website (conference session catalogs, official engineering blogs that embed or describe the talk) rather than guessed. Where a duration could not be confirmed, it is omitted rather than invented, per this module's sourcing standard. Treat this list as a starting point, not exhaustive — search the named channels directly for the newest recorded talks, since conference archives are updated continuously.

---

### ★★★★★ Speeding Up LLM Inference With TensorRT-LLM
**Creator/Channel:** NVIDIA (GTC session S62031, GTC San Jose 2024; distributed via NVIDIA On-Demand and the NVIDIA Developer YouTube channel)
**Session page:** https://www.nvidia.com/en-us/on-demand/session/gtc24-s62031/
**Difficulty:** Advanced
**Why it's worth watching:** The official NVIDIA walkthrough of how TensorRT-LLM achieves its throughput advantage over general-purpose serving frameworks — in-flight (continuous) batching, custom fused kernels, and the compile-time optimization workflow. Gives the "why would a team accept vendor lock-in and a slower iteration loop" argument directly from the team that built the trade-off, which is more convincing than any third-party summary.
**Which part of the tutorial it complements:** Part 3, the vLLM vs. TensorRT-LLM vs. Ollama vs. hosted-API comparison — specifically the "maximum throughput, single model, NVIDIA-only hardware" row of that table.

---

### ★★★★★ vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention
**Creator/Channel:** Woosuk Kwon (UC Berkeley Sky Computing Lab), presented at Ray Summit 2023; recording distributed via the Anyscale / Ray Summit YouTube channel
**Difficulty:** Advanced
**Why it's worth watching:** The original conference presentation of the PagedAttention idea and the vLLM system, delivered by the paper's lead author, alongside the Anyscale engineering blog post on continuous batching referenced in `references.md`. Watching the talk after reading the paper is a good way to consolidate the "why fragmentation happens in naive KV-cache allocation, and what paging fixes" intuition, since the talk walks through the memory-layout diagrams live rather than as static figures.
**Which part of the tutorial it complements:** Part 1 (why batching and memory management drive cost) and Part 3 (why vLLM became the default self-hosting choice).
**Note:** Search "vLLM Ray Summit 2023 Woosuk Kwon" on the Anyscale or Ray Summit YouTube channels if this specific recording has been reorganized into a different playlist since publication — conference video hosting occasionally migrates channels.

---

### ★★★★☆ Redis semantic caching / RedisVL and LangCache demos
**Creator/Channel:** Redis, Inc. (official Redis YouTube channel and Redis Developer relations content, accompanying the LangCache and RedisVL documentation)
**Difficulty:** Intermediate
**Why it's worth watching:** Redis's own developer relations team has published walkthroughs of building semantic caches with RedisVL's `SemanticCache` and the managed LangCache product, demonstrating the same Hash + vector-index + TTL pattern documented in the official Redis semantic-cache tutorial referenced in `references.md`. Useful as a visual companion to that written tutorial, especially for seeing the cache-hit-rate dashboard and threshold-tuning behavior demonstrated live rather than described in prose.
**Which part of the tutorial it complements:** Part 2, the Redis semantic-caching implementation section.
**Note:** Search "Redis semantic cache LangCache" or "RedisVL SemanticCache" on the official Redis YouTube channel (youtube.com/@Redis) for the current set of demo recordings — Redis has published and updated several of these as the LangCache product has matured through 2025–2026.

---

## Recommended channels to follow (rather than single pinned videos)

Because LLM serving is one of the fastest-moving areas in this entire course, the single most durable recommendation for this module is to **follow channels and conference archives directly** rather than rely on any one video staying current:

- **NVIDIA Developer** (YouTube) and the **NVIDIA GTC on-demand catalog** (nvidia.com/gtc/session-catalog) — new TensorRT-LLM, Triton, and Dynamo sessions are published after every GTC (twice yearly).
- **Anyscale** (YouTube/blog) — Ray Summit talks and engineering blog posts on vLLM, continuous batching, and serving cost at scale.
- **Redis** (YouTube/blog) — official RedisVL and LangCache tutorials and webinars on semantic caching.
- **AI Engineer** (ai.engineer, YouTube channel @aiDotEngineer) — a conference series specifically aimed at practicing AI/LLM engineers, with recurring tracks on inference cost, evaluation, and deployment; browse the latest World's Fair or Summit playlist for current-year talks on cost and latency optimization, since specific talk titles change every event.

If you want a single next action after this module: open the NVIDIA GTC session catalog and the Anyscale blog/YouTube presence and search each for "inference," "serving," or "TensorRT-LLM" / "vLLM" filtered to the last 6–12 months, since both organizations publish new, directly relevant material on a rolling basis and a static list here will age faster than the module's written references will.
