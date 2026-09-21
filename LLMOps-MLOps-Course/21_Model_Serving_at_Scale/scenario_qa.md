# Model Serving at Scale — Scenario-Based Q&A

**Situation:** A prototype that calls `.generate()` in a simple Python loop works great with one user at a time, but falls over as soon as it's opened up to 500 concurrent users, with requests timing out and GPU memory errors appearing. What would you do and why?

Model answer: Diagnose this as a batching/memory-management problem, not something to fix by throwing more GPUs at it blindly. A naive per-request generation loop leaves most of the GPU's parallelism unused and doesn't manage the KV cache efficiently across concurrent requests, which is exactly the gap purpose-built serving engines close. Replace the hand-rolled loop with a dedicated serving engine like vLLM, which implements continuous batching (swapping finished requests out and new ones in on every generation step) and PagedAttention (managing KV cache in fixed-size pages instead of one large contiguous block per request), dramatically increasing how many concurrent requests the same GPU can serve.

---

**Situation:** Your team notices that GPU memory usage climbs steadily throughout the day and eventually causes out-of-memory errors, even though the model weights themselves are a fixed, known size. What would you do and why?

Model answer: Suspect the KV cache before suspecting the model weights — the KV cache, which grows with every token generated per active conversation, can easily become the largest consumer of GPU memory, often exceeding the model weights themselves, especially under sustained concurrent load with long conversations. Check whether the serving engine is using an efficient KV-cache management scheme (like vLLM's PagedAttention, which avoids memory waste from fragmentation) rather than allocating one large contiguous block per request. If cache growth is inherent to genuinely long conversations, consider capping maximum context length or implementing cache eviction/summarization for very long-running sessions rather than assuming more GPU memory is the only fix.

---

**Situation:** A stakeholder asks why response latency gets noticeably worse during peak traffic hours even though the team increased the serving batch size to improve throughput. What would you do and why?

Model answer: Explain the batching tradeoff directly: larger batch sizes improve throughput (more requests processed per unit of GPU time) but can add latency for individual requests, since a request may wait longer for a batch to fill or for its turn within a larger batch. Check whether continuous/in-flight batching is enabled (which mitigates this compared to naive fixed-batch waiting) and whether batch size and timeout settings are tuned for the product's actual latency/throughput tradeoff rather than for raw throughput alone. If the product is latency-sensitive (an interactive chat UI), it may be worth accepting somewhat lower throughput per GPU for a batching configuration that keeps individual response times acceptable, and validating that choice with real time-to-first-token measurements at peak load.

---

**Situation:** Your infrastructure team wants to move your open-weight Llama-based service from a plain PyTorch deployment to TensorRT-LLM for maximum performance, but a teammate is worried about losing flexibility to quickly iterate on the model. What would you do and why?

Model answer: Validate the concern as real and weigh it explicitly rather than treating TensorRT-LLM as a strictly-better upgrade. TensorRT-LLM compiles the model into a highly optimized, GPU- and shape-specific execution engine, which gives a significant speed boost but adds a compilation step and makes the resulting engine less flexible to change on the fly than a plain PyTorch model — swapping in a new checkpoint or experimenting with a different configuration requires recompiling. Recommend TensorRT-LLM (often via Triton as the serving layer) once the model and its serving shape have stabilized and you need maximum NVIDIA-hardware throughput, but keep a faster-iterating engine like vLLM for active experimentation and any model still under frequent change.

---

**Situation:** Your team runs a RAG-heavy, agentic application where most requests share a large, identical system prompt, and inference cost per request is higher than expected given how much of the prompt is actually repeated. What would you do and why?

Model answer: Point out that a serving engine unaware of shared prefixes is redoing the same computation on the same system-prompt tokens for every single request, which is pure waste at this traffic shape. Evaluate SGLang specifically for this workload — its RadixAttention mechanism automatically detects shared prompt prefixes across requests and reuses their cached computation instead of recomputing it, which is exactly the pattern of "same system prompt across thousands of requests" this scenario describes. Benchmark the switch on your actual traffic mix (shared-prefix ratio, request concurrency) before committing, since the benefit scales with how much of the prompt is genuinely shared across requests.

---

**Situation:** Your serving infrastructure team is proposing autoscaling GPU replicas down to zero during low-traffic overnight hours to save cost, similar to how they autoscale a stateless web service. What would you do and why?

Model answer: Flag the difference between GPU-backed LLM serving and a typical stateless web service before approving this — loading a multi-gigabyte model onto a GPU takes real, non-trivial time, so scaling fully to zero means the first request after a quiet period pays a large cold-start latency penalty, which may violate a latency SLA even if traffic itself is genuinely low overnight. Recommend keeping a warm minimum pool of replicas (scaling down, not to zero) during low-traffic windows, sized against the acceptable worst-case latency for the first request in a burst, and reserve scale-to-zero for genuinely non-latency-sensitive or batch-style workloads where a cold start is acceptable.

---

**Situation:** A model serving deployment needs to support several different model types side by side — a classical PyTorch classifier, an ONNX-exported embedding model, and an LLM — and the team is debating whether to run three separate bespoke serving setups or unify them. What would you do and why?

Model answer: Recommend evaluating NVIDIA Triton Inference Server specifically for this mixed-model-type scenario, since it's a general-purpose model server (not LLM-specific) built to serve many model types side by side — PyTorch, ONNX, custom Python, and others — under one standardized API (HTTP/gRPC) with shared model versioning, dynamic batching, and monitoring. For the LLM component specifically, pair Triton with a specialized backend like TensorRT-LLM rather than running it through Triton's generic PyTorch path, since that's where the actual LLM-serving performance comes from — think of Triton as the unifying serving/orchestration layer and the backend as the engine doing the real inference work.

---

**Situation:** After enabling INT8 quantization on your serving deployment to cut costs, latency and throughput both improved significantly, but a stakeholder asks whether this could be silently hurting response quality. What would you do and why?

Model answer: Take the concern seriously and treat quantization as a real accuracy tradeoff requiring measurement, not a free win. Quantization (FP16/INT8/FP8) is genuinely one of the cheapest ways to cut latency and cost, but it typically comes with a small, measurable quality tradeoff that varies by model and task. Before shipping it broadly, run the quantized model against your existing evaluation suite (Module 09-11's evaluation datasets and judge) and compare quality metrics directly against the unquantized baseline, not just eyeballing a few outputs. If the measured quality drop is within an acceptable tolerance for the use case, ship it with that evidence documented; if not, consider a less aggressive quantization level (FP16 instead of INT8) as a middle ground.

---

**Situation:** Your serving fleet's load balancer routes requests to GPU replicas using simple round-robin, and you notice some replicas are consistently overloaded while others sit idle, even though request counts are evenly distributed. What would you do and why?

Model answer: Diagnose this as a load-balancing strategy mismatch with LLM traffic's actual cost profile — round-robin assumes each request costs roughly the same to serve, but LLM requests vary wildly in cost (a 20-token reply versus a 4,000-token essay), so evenly distributing request *count* does not evenly distribute actual GPU *load*. Switch to a load-balancing strategy based on queue depth or estimated load per replica rather than simple round-robin, so a replica currently processing several long-running generations receives fewer new requests than one that's mostly idle. Validate the fix by checking that GPU utilization, not just request count, becomes more evenly distributed across replicas afterward.

---

**Situation:** A large model doesn't fit in a single GPU's memory, and an engineer proposes simply reducing the batch size to 1 to make it fit, accepting whatever throughput results. What would you do and why?

Model answer: Push back on batch-size-1 as a workaround for a memory-fitting problem — this sacrifices most of the throughput benefit of running on a GPU at all and doesn't scale to real production traffic. The correct fix for a model too large for one GPU is sharding across multiple GPUs (tensor or pipeline parallelism), which lets the model run at a reasonable batch size and throughput by splitting the model itself across hardware rather than starving it of batching. Evaluate which serving engine and sharding strategy fit your model and available GPU topology, and benchmark actual throughput and latency under realistic concurrent load before committing to a sharding configuration.

---

**Situation:** Your team monitors GPU utilization and request-per-second as the primary health signals for the serving fleet, but users are reporting the service "feels slow" even when both metrics look normal. What would you do and why?

Model answer: Point out that GPU utilization and raw RPS don't directly capture the user-facing experience of LLM latency, and recommend monitoring the metrics that actually explain "feels slow": time-to-first-token (how long before any output appears, which dominates perceived responsiveness in a streaming chat UI), tokens-per-second during generation, and queue wait time before a request even starts processing. A service can show healthy GPU utilization and RPS while time-to-first-token or queue wait time has quietly regressed — for example, if batching settings favor throughput at the cost of an individual request waiting longer to be picked up. Add these user-facing latency signals to the dashboard alongside the infrastructure-level metrics, since they're the ones that actually correlate with the complaint.

---

**Situation:** A cost-cutting proposal suggests switching your production LLM serving fleet from a more expensive current-generation GPU to a cheaper, older GPU model to reduce cloud spend, based purely on the hourly rental price difference. What would you do and why?

Model answer: Push back on comparing hourly price alone — the metric that actually matters is cost per token (or cost per request) at a given latency target, not raw hourly GPU price, since an older, cheaper GPU may need more replicas (and more total GPU-hours) to hit the same throughput and latency SLA, potentially costing more overall despite a lower sticker price. Benchmark actual throughput and latency on the proposed GPU with your specific model and serving engine before deciding, and compute the fully-loaded cost per request at the required latency bar on both options, rather than comparing list prices in isolation.

---

**Situation:** Your team wants to serve a new fine-tuned variant of an existing model, and someone suggests treating it exactly like deploying an entirely new model from scratch, including a fresh TensorRT-LLM compilation and full capacity-planning cycle. What would you do and why?

Model answer: Evaluate whether that level of ceremony is actually warranted given how similar the fine-tuned variant is to the existing deployed model in architecture and expected traffic shape. If the underlying architecture and shape are unchanged, a lighter-weight rollout (same serving engine configuration, a canary deployment routing a small percentage of traffic to the new variant, comparing latency/throughput/quality against the existing baseline) is usually sufficient and faster than treating it as an entirely new capacity-planning exercise. Reserve the full "from scratch" process for genuinely new architectures or significant shape changes (a much larger model, a different context length) where the old capacity assumptions may no longer hold.

---

**Situation:** A postmortem after a brief outage reveals that a sudden traffic spike triggered autoscaling, but new GPU replicas took over two minutes to come online, during which existing replicas were overwhelmed and latency spiked badly for users. What would you do and why?

Model answer: Point directly at the mismatch between GPU cold-start time and the speed of the traffic spike as the root cause — GPU nodes are slower to spin up and far more expensive to leave idle than typical web-service compute, so an autoscaling policy tuned like a stateless web service's will systematically under-react to fast spikes. Increase the warm minimum replica pool to absorb a larger burst before autoscaling has to kick in, and tune the autoscaling trigger to activate earlier (on a leading indicator like rising queue depth, not just current GPU utilization, which lags behind an incoming spike) so new capacity has more lead time to come online before existing replicas are overwhelmed.
