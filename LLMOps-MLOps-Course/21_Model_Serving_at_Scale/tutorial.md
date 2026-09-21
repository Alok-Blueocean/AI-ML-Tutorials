# Model Serving at Scale

## What is this about?

Training a model gets all the attention, but *serving* it — answering real user requests, fast, cheaply, and reliably, day after day — is where most of the engineering effort in production LLM systems actually goes. "Model serving at scale" means running inference for many concurrent users on expensive, memory-constrained GPUs while keeping latency low, throughput high, and cost under control.

A model that works fine in a notebook with one prompt at a time can fall over completely when 500 people hit it at once. This module looks at the tools and techniques built specifically to solve that problem: **vLLM**, **NVIDIA Triton Inference Server**, **TensorRT-LLM**, and **SGLang**, plus the basics of GPU scheduling and autoscaling for LLM workloads.

## Why it matters

- **GPUs are expensive and scarce.** Every idle GPU-second is wasted money. Efficient serving squeezes more requests out of the same hardware.
- **Latency directly affects user experience.** Chat apps, copilots, and agents all feel "slow" or "broken" if token generation stalls under load.
- **Naive serving doesn't scale.** Simply loading a model with a basic Python script and calling `.generate()` in a loop works for a demo, not for thousands of concurrent requests — it wastes GPU memory and can't batch requests efficiently.
- **Cost per token matters at scale.** Small serving inefficiencies multiply into large cloud bills once you're handling millions of requests a day.

Understanding serving infrastructure is what separates a working prototype from a production LLM system.

## Main concepts in plain terms

### 1. The core problem: batching and memory

GPUs are fast because they do massive amounts of work in parallel. If you serve requests one at a time, you leave most of that parallelism unused. The trick is **batching**: grouping multiple requests together so the GPU processes them simultaneously.

But LLM requests are awkward to batch because each one generates a different number of tokens and finishes at a different time. This is solved by **continuous (in-flight) batching** — instead of waiting for a whole batch to finish before starting a new one, the server swaps finished requests out and new requests in on every generation step, keeping the GPU constantly full.

The other big constraint is the **KV cache** — the memory each request needs to "remember" its own conversation so far. This cache grows with every token generated and can easily become the biggest consumer of GPU memory, often more than the model weights themselves.

### 2. vLLM

vLLM is an open-source serving engine built around a technique called **PagedAttention**, which manages the KV cache like an operating system manages virtual memory — in small fixed-size pages rather than one large contiguous block per request. This avoids memory waste from fragmentation and lets vLLM pack far more concurrent requests into the same GPU memory. Combined with continuous batching, it's become a popular default choice for serving open-weight LLMs (Llama, Mistral, Qwen, etc.) with an OpenAI-compatible API out of the box.

### 3. NVIDIA Triton Inference Server

Triton is a general-purpose model server — it isn't LLM-specific, and it can serve many model types (PyTorch, TensorFlow, ONNX, custom Python) side by side. Its strengths are things production teams care about: standardized APIs (HTTP/gRPC), model versioning, dynamic batching, multi-model and multi-GPU deployment, and metrics for monitoring. For LLMs specifically, Triton is often paired with a specialized backend (such as TensorRT-LLM) rather than running raw PyTorch, since that's where the performance actually comes from. Think of Triton as the "serving platform/orchestration layer" and the backend as the "engine" doing the actual inference.

### 4. TensorRT-LLM

TensorRT-LLM is NVIDIA's library for **compiling** LLMs into highly optimized GPU execution engines. Instead of running the model through a general Python/PyTorch execution path, it fuses operations, applies precision optimizations (like FP16/INT8/FP8 quantization), and generates a specialized engine tailored to a specific GPU and model shape. The trade-off: you get a significant speed boost, but the compilation step adds complexity, and engines are less flexible to change on the fly than a plain PyTorch model. It's commonly used as the backend inside Triton for maximum throughput on NVIDIA GPUs.

### 5. SGLang

SGLang is a newer serving framework that focuses on efficient handling of *structured* and *complex* generation patterns — things like agent workflows, multi-turn conversations, JSON-constrained output, and prompts that share large common prefixes (e.g., the same system prompt across thousands of requests). Its standout feature is **RadixAttention**, an automatic KV-cache reuse mechanism that detects shared prompt prefixes across requests and reuses their cached computation instead of recomputing it. This makes it especially strong for RAG pipelines and agentic applications with repeated or templated prompts.

### 6. GPU scheduling and autoscaling basics

Once a single GPU is serving efficiently, the next problem is fleet-level: how many GPU instances do you run, and how do requests get distributed across them?

- **Request scheduling / load balancing**: incoming requests get routed across multiple model replicas, often using queue depth or estimated load rather than simple round-robin, since LLM requests vary wildly in cost (a 20-token reply vs. a 4,000-token essay).
- **Horizontal autoscaling**: adding or removing GPU replicas based on demand (e.g., queue length, GPU utilization, or requests-per-second) — much like autoscaling a normal web service, except GPU nodes are slower to spin up and far more expensive to leave idle.
- **Cold starts**: loading a multi-gigabyte model onto a GPU takes real time, so autoscaling for LLMs often keeps a warm minimum pool of replicas rather than scaling fully to zero.
- **Model/GPU packing**: smaller models may be packed several-to-a-GPU, while large models may need to be *sharded* across multiple GPUs (tensor or pipeline parallelism) just to fit.

## A simple example: serving with vLLM

Here's roughly what stands between "I have model weights" and "I have an API":

```bash
pip install vllm

# Launch an OpenAI-compatible server for a model
vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --port 8000 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.90
```

```python
# Call it just like the OpenAI API
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-needed")

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Explain KV cache in one sentence."}],
)
print(response.choices[0].message.content)
```

Behind that one `vllm serve` command, continuous batching, PagedAttention memory management, and request queuing are all handled automatically — which is the whole point of using a dedicated serving engine instead of writing your own loop.

## Key takeaways / best practices

- **Don't serve LLMs with a naive for-loop.** Use a purpose-built engine (vLLM, TensorRT-LLM, SGLang) that implements continuous batching and efficient KV-cache management.
- **Match the tool to the job**: vLLM for a fast, flexible open-source default; TensorRT-LLM (often via Triton) when you need the absolute best NVIDIA-hardware performance and can afford a compilation step; Triton when you need a unified serving platform across multiple model types; SGLang when your workload has shared prefixes or complex/agentic generation patterns.
- **The KV cache is usually your real memory bottleneck**, not the model weights — plan capacity around it.
- **Batching improves throughput but can add latency** for individual requests; tune batch size and timeout settings to your latency/throughput trade-off.
- **Autoscale on the right signal** (queue depth or GPU utilization, not just CPU/RPC count) and keep a warm minimum pool, since GPU cold starts are slow and expensive to hide from users.
- **Quantization (FP16/INT8/FP8) is one of the cheapest ways to cut latency and cost**, usually with a small, measurable quality trade-off — always benchmark before shipping.
- **Monitor real production signals**: time-to-first-token, tokens/sec, queue wait time, and GPU utilization tell you far more than raw request count.
