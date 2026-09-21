# GitHub Repositories — Module 08: LLM Latency, Cost, and Deployment Strategy

All repositories below were verified to exist and confirmed for purpose/features at time of writing (mid-2026). Star counts are approximate popularity snapshots, not guarantees of current exact counts — check the repo directly for the live number.

---

## Inference Servers (self-hosted deployment)

### vLLM
**URL:** https://github.com/vllm-project/vllm
**Popularity tier:** Extremely popular / the de facto standard (~88k GitHub stars, 2,000+ contributors).
**Purpose:** A fast, easy-to-use library and OpenAI-API-compatible server for LLM inference, originating from UC Berkeley's Sky Computing Lab. Implements PagedAttention (efficient, fragmentation-free KV-cache memory management) and continuous batching, plus quantization (FP8/INT8/INT4/AWQ/GPTQ/GGUF), distributed inference (tensor/pipeline/data/expert/context parallelism), and broad hardware support (NVIDIA, AMD, Intel GPUs, TPUs, x86/ARM/PowerPC CPUs, Apple Silicon, Huawei Ascend).
**How it relates to this module:** This is the repository backing the module's default recommendation for self-hosted production LLM serving. The `serving-config.yaml` pattern taught in this module (max concurrent sequences, GPU memory utilization target, prefix caching toggle) maps directly onto vLLM's real CLI/server flags — reading the repo's `docs/serving` examples turns the module's illustrative YAML into something you could actually deploy.

### NVIDIA TensorRT-LLM
**URL:** https://github.com/NVIDIA/TensorRT-LLM
**Popularity tier:** Very popular within the NVIDIA/enterprise-GPU ecosystem (~14k GitHub stars).
**Purpose:** NVIDIA's library for compiling LLMs into optimized TensorRT engines for NVIDIA GPUs, with custom fused attention/GEMM/MoE kernels, prefill-decode disaggregation, speculative decoding, and multi-GPU/multi-node support. Recently moved to a PyTorch-native high-level Python API for defining and optimizing models rather than a purely engine-compilation workflow.
**How it relates to this module:** The "maximum throughput, willing to pay compile-time and vendor lock-in cost" option in the module's vLLM vs. TensorRT-LLM vs. Ollama vs. hosted-API comparison table. Also the engine most often paired with NVIDIA Triton Inference Server in enterprise hybrid-deployment case studies.

### Ollama
**URL:** https://github.com/ollama/ollama
**Popularity tier:** Extremely popular for local/desktop LLM usage (~177k GitHub stars — one of the most-starred AI infra projects on GitHub).
**Purpose:** A single-binary CLI and local server for running quantized (GGUF) open models with one command (`ollama run <model>`), plus a REST API and growing ecosystem of editor/IDE and chat-UI integrations. Explicitly optimized for local developer experience, not multi-tenant production throughput.
**How it relates to this module:** The module's "under 5 minutes to a running model, single-user or edge/offline deployment" corner of the deployment comparison table. Also the practical tool for prototyping the module's cost-modeling exercises locally before committing to a hosted-API or vLLM-cluster spend estimate.

### Hugging Face Text Generation Inference (TGI)
**URL:** https://github.com/huggingface/text-generation-inference
**Popularity tier:** Popular, historically significant (~11k GitHub stars).
**Purpose:** A Rust/Python/gRPC-based production inference server (originally powering HuggingChat and HF's hosted Inference API) with SSE token streaming, continuous batching, and an OpenAI-compatible Messages API.
**How it relates to this module:** Useful for seeing the server-side implementation of SSE token streaming referenced in the module's streaming section. **2026 currency note:** the project is now in maintenance mode — Hugging Face has shifted engineering focus to vLLM/SGLang, so present TGI as historically important rather than a first choice for a new deployment.

### NVIDIA Triton Inference Server
**URL:** https://github.com/triton-inference-server/server
**Popularity tier:** Very popular / widely adopted in enterprise MLOps (~11k GitHub stars).
**Purpose:** A general-purpose, multi-framework model-serving platform (TensorRT, PyTorch, ONNX, OpenVINO, Python backend) with dynamic batching, concurrent model execution, model ensembling, and HTTP/gRPC protocols — not LLM-specific, but frequently used to front a TensorRT-LLM engine in enterprise deployments.
**How it relates to this module:** The standard "how do we get unified observability and routing across many models, not just our LLM" layer referenced in the module's hybrid-architecture case study.

---

## Semantic Caching / LLM Response Caching

### GPTCache (zilliztech/GPTCache)
**URL:** https://github.com/zilliztech/GPTCache
**Popularity tier:** Very popular / widely adopted semantic-caching library (~8.1k GitHub stars).
**Purpose:** A modular semantic-caching library for LLM applications with pluggable embedding generators, vector stores, and cache backends (SQLite, PostgreSQL, MongoDB, Redis, DynamoDB, and more), plus first-class integrations with LangChain and LlamaIndex. Ships a Docker server for language-agnostic access. Markets itself around "cut costs 10x, boost speed 100x" via cache reuse.
**How it relates to this module:** The most mature, batteries-included open-source alternative to hand-rolling the module's Redis semantic-cache code sample — good "next step" repo for students who want a production-hardened library rather than a from-scratch implementation.

### RedisVL (redis/redis-vl-python)
**URL:** https://github.com/redis/redis-vl-python
**Popularity tier:** Growing / actively maintained official Redis project (smaller star count than GPTCache but backed directly by Redis, Inc.).
**Purpose:** The "AI-native Redis Python client," providing a `SemanticCache` class (embedding + distance threshold + TTL + metadata filtering built in), vector similarity search with hybrid (semantic + full-text) filtering, an `MessageHistory` LLM-memory helper, semantic routing, and an MCP server for exposing Redis indexes to MCP-compatible agent clients.
**How it relates to this module:** The higher-level, officially-supported alternative to the raw `FT.SEARCH`-based semantic cache code taught in this module — read after the from-scratch version to see what a production team would actually adopt instead of maintaining bespoke caching code.

### redis/docs — semantic cache reference implementation
**URL:** https://github.com/redis/docs (see `content/develop/use-cases/semantic-cache/redis-py/`)
**Popularity tier:** Official Redis documentation source repository.
**Purpose:** Contains the full runnable source (`cache.py`, `mock_llm.py`, `seed_cache.py`, `demo_server.py`) for the official Redis semantic-caching tutorial: a Redis Hash schema with a raw float32 embedding field, a combined `FT.SEARCH ... KNN` query with TAG metadata pre-filtering, tunable cosine-distance thresholds, TTL-based expiry, and a small interactive web demo for sweeping the threshold and watching hit/miss behavior live.
**How it relates to this module:** This is the closest real-world sibling to the module's own semantic-caching code sample — same core pattern (Hash + vector index + metadata TAG filter + distance threshold + TTL), useful as a reference to diff your own implementation against.

---

## Books / Courses with Companion Code

### LLM Engineer's Handbook — companion repository
**URL:** https://github.com/PacktPublishing/LLM-Engineers-Handbook
**Popularity tier:** Popular companion-code repo (~5.3k GitHub stars).
**Purpose:** Official code for Paul Iusztin and Maxime Labonne's *LLM Engineer's Handbook* (Packt), covering an end-to-end LLM system: data collection, training pipelines, RAG, AWS deployment, and production monitoring, built with Domain-Driven Design principles and tools like ZenML, Hugging Face, Comet ML, and Opik.
**How it relates to this module:** Good source of realistic, non-toy deployment and monitoring code to compare against the module's own patterns — particularly its AWS deployment and production-monitoring chapters, which intersect with this module's API-vs-self-hosted cost tradeoffs.

---

## What to skip / deprioritize in mid-2026

Several once-popular repos in this space (early quantization-focused runtimes, first-generation LangChain caching wrappers pinned to since-deprecated APIs) have either stagnated or been superseded by the vLLM/SGLang/TensorRT-LLM trio above plus the official provider caching APIs (OpenAI/Anthropic/Gemini). When recommending further reading to students, prefer repos with commits and releases in the last 6 months over older, more-starred-but-now-dormant alternatives — star count is a lagging indicator of adoption, not current relevance, and is especially misleading in the fast-moving LLM-serving space.
