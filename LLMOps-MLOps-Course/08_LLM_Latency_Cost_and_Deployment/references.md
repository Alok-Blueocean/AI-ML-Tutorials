# References — Module 08: LLM Latency, Cost, and Deployment Strategy

Curated, verified external references: official docs, papers, and engineering blogs. Organized to follow the module's three parts: (1) latency & cost drivers, (2) batching/caching/streaming optimization, (3) API vs. local deployment strategy.

---

## Part 1 — Latency and Cost Drivers (TTFT, P95, token economics)

### vLLM Metrics design doc
**URL:** https://docs.vllm.ai/en/latest/design/metrics/
**What it teaches:** The canonical, precise definitions of the latency metrics you will be quoting in every capacity-planning conversation for the rest of your career: `time_to_first_token_seconds` (TTFT), inter-token latency / time-per-output-token (ITL/TPOT), and end-to-end request latency. It explains *why* TTFT is measured from `arrival_time` (when the frontend receives the request, before tokenization) rather than from when the GPU starts computing — a distinction that matters when you're debugging why your measured TTFT doesn't match your load-balancer's numbers. Also documents the Prometheus histogram buckets vLLM ships (1ms to tens of seconds) and points to the reference Grafana dashboard.
**Difficulty:** Intermediate. **Reading time:** 15–20 minutes.
**Why it matters for this module:** This is the ground-truth vocabulary for the module's core distinction between TTFT (perceived responsiveness, dominated by prefill + queueing) and total/P95 latency (dominated by output length + decode speed). Read this before building any latency dashboard or SLO.

### Databricks (MosaicML) — "LLM Inference Performance Engineering: Best Practices"
**URL:** https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices
**What it teaches:** One of the most-cited practitioner explanations of *why* LLM inference is memory-bandwidth-bound rather than compute-bound, and what that implies for cost. Introduces **Model Bandwidth Utilization (MBU)** as the metric that actually predicts your tokens/sec/dollar, distinct from the more famous MFU (Model FLOPs Utilization) borrowed from training. Gives concrete measured numbers: continuous batching yields roughly 10–20x throughput over naive per-request batching; H100 vs A100 latency deltas; diminishing returns from scaling GPU count per replica.
**Difficulty:** Intermediate/Advanced. **Reading time:** 25–30 minutes.
**Why it matters for this module:** This is the deepest "why" behind the module's three cost levers (model size, context length, output length) — it explains at the hardware level why output length is the most expensive lever (every output token is a full sequential decode step) while input/context tokens are comparatively cheap to process (parallel prefill).

### ClickHouse Engineering — "LLM inference latency: TTFT, tokens per second, and what to measure"
**URL:** https://clickhouse.com/resources/engineering/llm-inference-latency
**What it teaches:** A vendor-neutral breakdown of which latency numbers to actually put in a dashboard and why raw "tokens/sec" is a misleading headline metric once you account for streaming and concurrent requests. Good complement to the vLLM metrics doc because it's written from the consumer/observability side rather than the server-internals side.
**Difficulty:** Intermediate. **Reading time:** 10–15 minutes.

### Anyscale Engineering — "How continuous batching enables 23x throughput in LLM inference while reducing p50 latency"
**URL:** https://www.anyscale.com/blog/continuous-batching-llm-inference
**What it teaches:** The blog post that made continuous (iteration-level) batching mainstream knowledge. Explains the failure mode of static/naive batching (a full batch stalls until its longest sequence finishes, wasting GPU slots on already-finished short sequences), and quantifies the throughput and p50 latency wins from letting new requests fill freed slots mid-batch. Cites concrete multipliers: ~8x over naive batching for standard continuous batching, ~23x when combined with vLLM's PagedAttention memory management.
**Difficulty:** Intermediate. **Reading time:** 15 minutes.
**Why it matters for this module:** This is the mechanism underneath the module's "batching" optimization lever — read it alongside the PagedAttention paper below to understand *both* halves (scheduling + memory management) of why modern inference servers are so much cheaper per request than a naive `for` loop calling `model.generate()`.

---

## Part 2 — Optimization: Batching, Caching, Streaming

### Anthropic — "Prompt caching with Claude" (official announcement + docs)
**URL (blog):** https://claude.com/blog/prompt-caching
**URL (docs):** https://docs.claude.com/en/docs/build-with-claude/prompt-caching
**What it teaches:** The mechanics and pricing model of Anthropic's prompt caching: cache writes cost 1.25x (5-minute TTL) or 2x (1-hour TTL) the base input rate, cache reads cost roughly 0.1x base input rate (a ~90% discount). Documents the break-even logic explicitly — a 5-minute cache write pays for itself after a single subsequent read; a 1-hour write after two reads. Shows real before/after numbers: a 100K-token cached document ("chat with a book") measured a 79% latency reduction and 90% cost reduction; a 10K-token many-shot prompt saw 86% cost savings. Explains you can place up to four `cache_control` breakpoints per request, so a system prompt, a tool schema, and a mid-conversation retrieved document can each cache independently.
**Difficulty:** Beginner/Intermediate. **Reading time:** 15 minutes.
**Why it matters for this module:** This is the primary source for the module's "prompt caching achieving large savings" claim — use the write/read multipliers here to build the cost model spreadsheet's caching row precisely, rather than approximating.

### OpenAI — "Prompt Caching" (official API guide)
**URL:** https://developers.openai.com/api/docs/guides/prompt-caching
**Cookbook walkthroughs:** https://cookbook.openai.com/examples/prompt_caching101 and https://developers.openai.com/cookbook/examples/prompt_caching_201
**What it teaches:** OpenAI's caching is automatic (no explicit cache-write cost on older models) once a prompt prefix exceeds 1,024 tokens, matched in 128-token increments and routed by a hash of the first ~256 tokens to a server that recently saw that prefix. Explains the `prompt_cache_key` parameter for steering routing when you have multiple distinct long-prefix workloads sharing one deployment, and the newer explicit-breakpoint mode on GPT-5.6-class models where cache writes do carry a small (1.25x) premium.
**Difficulty:** Beginner/Intermediate. **Reading time:** 15–20 minutes.
**Why it matters for this module:** Read alongside the Anthropic doc to build a provider-agnostic mental model of prompt caching — the module should teach the *pattern* (put static content first, keep breakpoints stable, measure `cached_tokens`), which is close to identical across OpenAI, Anthropic, and Gemini even though the discount math differs slightly.

### Google — Gemini API "Context Caching" docs
**URL:** https://ai.google.dev/gemini-api/docs/caching
**What it teaches:** Gemini's implicit caching (on by default for Gemini 2.5+ models, no code changes required) versus explicit caching (developer creates and manages a cache object with its own TTL, useful for very large fixed contexts like an entire codebase or manual). Minimum-token thresholds differ from OpenAI's (2,048–4,096 tokens depending on model family).
**Difficulty:** Beginner. **Reading time:** 10 minutes.
**Why it matters for this module:** Completes the three-provider comparison table for prompt/context caching that the module's cost framework should include, since a production system routing across providers needs to know these thresholds differ.

### Redis — "Redis semantic cache" (official use-case docs)
**URL:** https://redis.io/docs/latest/develop/use-cases/semantic-cache/
**Runnable code walkthrough (Python/redis-py):** https://redis.io/docs/latest/develop/use-cases/semantic-cache/redis-py
**Managed offering:** https://redis.io/docs/latest/develop/ai/context-engine/langcache (Redis LangCache)
**What it teaches:** This is the most directly useful reference in the whole module. It explains precisely why semantic caching is a *different problem* from both exact-match caching (misses paraphrases) and a bare vector database (no first-class TTL/eviction/metadata filtering baked in for cache semantics). The redis-py walkthrough gives a complete, runnable reference implementation: a Redis Hash schema (prompt, response, raw float32 embedding bytes, tenant/locale/model-version metadata, TTL, hit_count), a combined `FT.SEARCH ... KNN` query that does metadata pre-filtering and vector similarity in a single round trip, and a full hit/miss decision flow with a tunable cosine-distance threshold. It is close in spirit to the module's own semantic-caching code sample and is the primary source to point students at for going deeper.
**Difficulty:** Intermediate. **Reading time:** 30–40 minutes including code.
**Why it matters for this module:** This is the reference implementation for Part 2's Redis semantic caching section — the module's own code sample should be read as a simplified, task-specific derivative of this pattern (same core idea: Hash + vector index + TAG pre-filter + distance threshold + TTL/eviction).

### RedisVL (Redis Vector Library) — `SemanticCache` API
**URL:** https://github.com/redis/redis-vl-python
**Docs:** https://docs.redisvl.com
**What it teaches:** A higher-level Python client that wraps the raw `FT.SEARCH` pattern above into a `SemanticCache` class with built-in embedding, distance thresholds, TTL, and metadata filters — the "don't hand-roll it" alternative once you understand the underlying mechanics from the raw redis-py walkthrough.
**Difficulty:** Intermediate. **Reading time:** 15 minutes to skim the README + LLM cache guide.

### FastAPI — "Server-Sent Events (SSE)" official tutorial
**URL:** https://fastapi.tiangolo.com/tutorial/server-sent-events/
**What it teaches:** The official, current (2026-era) pattern for streaming responses from a FastAPI endpoint using `EventSourceResponse` and the `ServerSentEvent` model, including keep-alive pings, cache-prevention headers, proxy-buffering prevention, and resuming an interrupted stream via `Last-Event-ID` — all handled for you rather than hand-rolled, which is the biggest change from older (pre-2025) tutorials that manually set `text/event-stream` headers on a raw `StreamingResponse`. Also documents SSE-over-POST, needed because most chat endpoints are POST, not GET.
**Difficulty:** Beginner/Intermediate. **Reading time:** 15–20 minutes.
**Why it matters for this module:** This is the current official pattern to teach for the module's FastAPI SSE streaming implementation — prefer `EventSourceResponse` over hand-rolling `StreamingResponse` with manual SSE framing, since the framework now solves the edge cases (proxy buffering, keep-alives) that used to require bespoke code.

### Hugging Face — Text Generation Inference (TGI) documentation
**URL:** https://github.com/huggingface/text-generation-inference
**What it teaches:** TGI documents token streaming via SSE and continuous batching from the server side (as opposed to FastAPI's client-facing SSE tutorial above) — useful for seeing the same SSE protocol implemented in Rust at the inference-server layer.
**Difficulty:** Intermediate.
**2026 currency note:** As of mid-2026 TGI is officially in **maintenance mode** (bug fixes and doc updates only) — Hugging Face has redirected engineering focus toward vLLM and SGLang. Teach this as a "know it exists, historically important, but don't default to it for new production deployments" reference rather than a first recommendation.

---

## Part 3 — API vs. Local/Self-Hosted Deployment Strategy

### vLLM project — official documentation and OpenAI-compatible server
**URL:** https://docs.vllm.ai/
**GitHub:** https://github.com/vllm-project/vllm
**What it teaches:** How to stand up an OpenAI-API-compatible inference server backed by PagedAttention and continuous batching; covers quantization (FP8/INT8/INT4/AWQ/GPTQ/GGUF), tensor/pipeline/data/expert parallelism, and the serving-config surface (`--max-num-seqs`, `--gpu-memory-utilization`, `--enable-prefix-caching`, etc.) that a `serving-config.yaml` in a real deployment maps onto.
**Difficulty:** Intermediate/Advanced. **Reading time:** Several hours to work through the quickstart + serving guide.
**Why it matters for this module:** vLLM is the default production self-hosting choice referenced throughout Part 3 — read the docs before treating any vLLM-vs-alternatives comparison table as settled, since flags and defaults change across releases.

### NVIDIA — TensorRT-LLM documentation and GTC session "Speeding up LLM Inference With TensorRT-LLM" (Session S62031, GTC 2024)
**GitHub/docs:** https://github.com/NVIDIA/TensorRT-LLM
**Session:** https://www.nvidia.com/en-us/on-demand/session/gtc24-s62031/
**What it teaches:** TensorRT-LLM's compiled-engine approach (custom fused kernels, in-flight batching, speculative decoding, prefill/decode disaggregation) that trades a longer build/compile step and NVIDIA-only hardware lock-in for typically 15–30% higher throughput than vLLM on the same GPU, per independent benchmarks. The GTC session is the official NVIDIA walkthrough of the same material in talk form.
**Difficulty:** Advanced.
**Why it matters for this module:** The canonical "maximum throughput, single model, NVIDIA-only, willing to pay engineering cost" corner of the deployment-choice comparison table.

### Ollama — official GitHub repository
**URL:** https://github.com/ollama/ollama
**What it teaches:** The single-command local model runner (`ollama run llama3.3`) with a REST API, GGUF-based quantized models, and first-class support for CPU-only and consumer-GPU machines. It is explicitly *not* built for multi-tenant, high-concurrency production serving — no continuous batching or PagedAttention-style KV cache sharing across concurrent requests.
**Difficulty:** Beginner.
**Why it matters for this module:** The canonical "prototype on a laptop, single-user desktop assistant, or compliance-mandated fully-offline edge deployment" corner of the comparison table — explicitly the wrong tool once you need multi-user throughput, which the module should state plainly rather than let students infer.

### NVIDIA Triton Inference Server
**URL:** https://github.com/triton-inference-server/server
**What it teaches:** A general-purpose multi-framework model server (not LLM-specific) that can host a TensorRT-LLM engine, an ONNX model, and a PyTorch model side by side behind one gRPC/HTTP endpoint, with dynamic batching and model ensembling/Business Logic Scripting for multi-step pipelines.
**Difficulty:** Advanced.
**Why it matters for this module:** Relevant to the "hybrid architecture" case study — large enterprises frequently front a TensorRT-LLM-compiled model with Triton to get standardized observability/routing across many different model types, not just LLMs.

### Efficient Memory Management for Large Language Model Serving with PagedAttention (paper)
**Authors:** Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph E. Gonzalez, Hao Zhang, Ion Stoica (UC Berkeley Sky Computing Lab)
**Venue:** SOSP 2023
**URL:** https://arxiv.org/abs/2309.06180
**What it teaches:** The systems paper behind vLLM. Frames LLM serving's core inefficiency as KV-cache memory fragmentation (analogous to external fragmentation in naive memory allocators) and solves it with an OS-virtual-memory-inspired paging scheme, enabling flexible KV cache sharing across requests (e.g., shared system-prompt prefixes). Reports 2-4x throughput improvement over then-state-of-the-art systems (FasterTransformer, Orca) at equivalent latency.
**Difficulty:** Advanced (systems paper; assumes familiarity with attention mechanics and OS memory management).
**Reading time:** 45–60 minutes.
**Why it matters for this module:** This is the paper every "why is vLLM fast" claim in the module traces back to — worth reading in full at least once for a senior MLOps engineer, since interviewers occasionally probe whether candidates actually understand PagedAttention or are just repeating the marketing line.

---

## Cross-cutting: Cost Modeling and Case-Study Pattern

### OpenAI / Anthropic / Google published API pricing pages
These are the primary sources for any cost-modeling spreadsheet — always check the live pricing page rather than caching numbers in a static doc, since per-token prices for a given model tier have historically moved (usually down) multiple times per year.
- OpenAI: https://openai.com/api/pricing/
- Anthropic: https://www.anthropic.com/pricing
- Google Gemini: https://ai.google.dev/gemini-api/docs/pricing

**Why it matters for this module:** The module's cost-modeling framework should be built as a *formula* (model tier price × tokens, minus cache-hit discount, minus batch-hit rate) that engineers re-run against current prices, not a framework with 2024-era prices hardcoded into it — treat any specific dollar figure from an older source (including the "36 cents to 11 cents" case study in this module's transcripts) as illustrative of the *mechanism*, not as a number to still expect today.
