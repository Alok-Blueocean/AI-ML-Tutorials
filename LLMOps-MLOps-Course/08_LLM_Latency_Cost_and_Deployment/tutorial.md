# Module 08 — LLM Latency, Cost, and Deployment Strategy

> Module 08 of the LLMOps/MLOps Engineering Course
> Prerequisites: Modules 01-06 (CI/CD foundations, versioning/registries/rollback, reproducibility/environments, Docker, Kubernetes)

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Measure and reason about the four core LLM production metrics — TTFT, total latency (P50/P95/P99), token cost, and context cost — and explain why looking at only one gives a false picture of system health.
2. Identify and manipulate the three cost levers of any LLM system — model size, context length, output length — and build a request-level cost model you can drop into a code review or an architecture doc.
3. Implement the three throughput/latency optimization techniques — batching, caching, streaming — plus prompt caching, with working Python and Redis code, and know which one to reach for given a specific symptom (high cost vs. slow start vs. low throughput).
4. Build a real semantic cache with Redis (vector similarity + metadata filtering) and a real token-streaming HTTP endpoint with FastAPI Server-Sent Events (SSE).
5. Make and defend a build-vs-buy deployment decision (hosted API vs. self-hosted vLLM/TensorRT-LLM/Ollama) using a quantitative framework: request volume, compliance constraints, latency SLOs, and operational maturity — not vibes.
6. Design a hybrid, tiered-routing architecture that mixes a premium hosted API, a self-hosted open model, and a fallback path, the way real high-scale AI products do it.
7. Recognize the common mistakes teams make when they treat latency/cost optimization as an afterthought rather than a first-class system design constraint, and apply the production checklist at the end of this module before shipping.

### Prerequisites

- Comfort with Python async/await and HTTP APIs (you'll write async FastAPI endpoints).
- Basic familiarity with vector embeddings and vector search (used for semantic caching) — if this is new, treat the code in section 3.2 as your first hands-on introduction; it is self-contained.
- Everything from Modules 01-06: you should already know how to containerize a service (Module 04), orchestrate it on Kubernetes (Module 05), version a model/config artifact, and roll it back. This module builds a *serving-layer* concern on top of that foundation — it assumes your deployment pipeline already exists and asks "now that it's deployed, is it fast enough and cheap enough?"
- No prior exposure to vLLM, TensorRT-LLM, or Redis is assumed; each is introduced from first principles.

### Key Terminology

| Term | Definition |
|---|---|
| **TTFT (Time-to-First-Token)** | Wall-clock time from request submission to the first generated token reaching the client. The dominant driver of *perceived* responsiveness. |
| **ITL (Inter-Token Latency)** | Time between successive streamed tokens after the first one. Determines how "smooth" a streaming response feels. |
| **Total / End-to-End Latency** | Time from request submission to the *last* token. What a non-streaming client experiences as "the wait." |
| **P50 / P95 / P99 latency** | The 50th/95th/99th percentile of a latency distribution. P95/P99 matter more than the mean in production because averages hide the worst-case tail that angry users and paging alerts are made of. |
| **Token cost** | The dollar cost of a request, driven by (input tokens × input price) + (output tokens × output price). |
| **Context cost** | The subset of token cost attributable specifically to input/prompt tokens — system prompt, chat history, retrieved documents. |
| **Continuous batching** (a.k.a. in-flight or dynamic batching) | A serving technique where new requests are folded into a GPU batch mid-generation, rather than waiting for a fixed batch to fully complete, dramatically raising GPU utilization. |
| **PagedAttention** | vLLM's memory-management technique that stores the KV cache in non-contiguous, page-like blocks (like OS virtual memory), eliminating fragmentation and enabling much higher batch concurrency. |
| **KV cache** | The cached key/value tensors from previous attention computations, reused so each new token doesn't require recomputing attention over the entire prior sequence. It is the single largest consumer of GPU memory during LLM inference. |
| **Semantic caching** | Caching keyed not by exact string match but by *embedding similarity* — a paraphrased question that means the same thing as a previously-answered one is served from cache. |
| **Prompt caching** (a.k.a. context caching) | A provider-side or server-side mechanism that caches the KV-cache state of a repeated prompt prefix (e.g., a long system prompt) so subsequent requests reusing that prefix skip re-processing it, at a steep discount. |
| **Server-Sent Events (SSE)** | A simple, unidirectional, HTTP-native streaming protocol (`text/event-stream`) well suited to token-by-token LLM output streaming; simpler than WebSockets when the client only needs to *receive*. |
| **Break-even point** | The request volume at which the fully-loaded cost of self-hosting (GPUs + ops engineering time) becomes cheaper than paying per-token API pricing. |
| **Data residency / compliance-driven deployment** | Deployment forced toward self-hosting not by cost economics but by legal/regulatory requirements that prohibit sending certain data to third-party API providers. |

---

## 2. Why This Topic Matters, and Where It Fits in the LLMOps Lifecycle

Modules 01-06 gave you the machinery to reliably *ship* a model: version it, containerize it, orchestrate it, roll it back if something breaks. That machinery answers the question "can we deploy this safely and repeatably?" This module answers a different, equally load-bearing question: **once it's deployed, is it good enough to keep running?**

"Good enough" in LLM production has three axes that trade off against each other constantly:

```
                      QUALITY
                        /\
                       /  \
                      /    \
                     /      \
                    /        \
                   /  PICK    \
                  /  ANY TWO   \
                 /  (mostly)    \
                /________________\
           SPEED                  COST
      (latency / TTFT)      ($ per request,
                              $ per month)
```

A model that is more accurate but takes 8 seconds to start responding will lose users to a "good enough" model that responds in 300ms. A model that is fast and accurate but costs $0.36 per request will bankrupt a product that needs to serve millions of free-tier users. This is not a hypothetical tension — it is the single most common reason LLM-powered features get killed after launch: they worked in the demo, and then the AWS/OpenAI bill or the P95 latency dashboard made them unshippable at scale.

Where this sits in the lifecycle:

```
 Module 01-02        Module 03            Module 04-05         Module 08 (HERE)        Module 09+
 CI/CD +          Reproducibility      Containerize +        Latency, Cost &      Observability,
 Versioning        (envs, deps,        Orchestrate            Deployment           Evaluation,
                    determinism)        (Docker, K8s)          Strategy             Guardrails...

 "Can I ship      "Will it behave    "Can it run          "Is it fast enough,   "Do I know when
  this reliably    the same way       reliably at          cheap enough, and     it's silently
  and reversibly?"  every time?"       scale?"              deployed the          degrading?"
                                                             right way?"
```

Notice that this module is deliberately placed *after* containerization/orchestration and *before* deep observability. That's intentional: you cannot optimize latency and cost until you can deploy and re-deploy safely (Modules 01-06), and you cannot know whether your optimizations are actually working in production without dashboards and tracing (later modules will build on the metrics defined here). This module is the bridge between "it runs" and "it runs well, cheaply, and predictably" — arguably the most senior-engineer-differentiating skill set in the entire LLMOps discipline, because it requires reasoning jointly about ML behavior, distributed systems, and unit economics.

---

## 3. Main Concepts

### 3.1 Latency and Cost Metrics — The Four Numbers That Define "Good"

#### Theory

**What it is.** Four measurements you must instrument on every LLM-backed endpoint before you can claim it is production-ready:

1. **TTFT (Time-to-First-Token)** — how long the user waits before *anything* appears.
2. **Total latency, measured at P95 (not average)** — how long the full response takes, focusing on the tail.
3. **Token cost** — dollars per request, driven by total tokens processed.
4. **Context cost** — the input-token slice of that cost specifically, because it's the slice most within your control at the prompt-engineering layer.

**What problem it solves.** Without these four numbers, "is the system good?" is a matter of opinion. With them, it's a measurable SLO you can alert on, budget against, and use to justify (or reject) an architecture change. They also decompose the problem: a system can be *fast but expensive* (small dense model, huge context), *cheap but slow* (large batch queue, small GPU pool), or *fast and cheap but wrong* (over-aggressive caching returning stale/irrelevant answers) — you need all four numbers plus a quality metric (covered in a later evaluation module) to tell these apart.

**Why P95, not average.** The vLLM project's own metrics design explicitly tracks latency histograms rather than simple means, because LLM serving latency is famously multimodal: most requests are fast, but a request that lands behind a large batch, or that triggers a KV-cache eviction, or that hits a cold model instance after autoscaling, can take 5-10x longer. An average of 400ms can hide a P99 of 4 seconds — and it's the P99 users tweet about. This is a general distributed-systems lesson (any queueing system has a long tail) but it is *especially* pronounced in LLM serving because generation time scales with output length and batch contention in a way that a stateless REST endpoint does not.

**Tradeoffs / when to use / when not to.**
- Track TTFT separately from total latency always — they answer different product questions (TTFT → "does it feel responsive," total latency → "can the user actually get the full answer in time for a hard SLA," e.g., a synchronous API caller with a timeout).
- Context cost tracking is worth the instrumentation effort specifically when your product has variable-length inputs (RAG, chat history, document upload). If every request has a near-identical fixed-size prompt, context cost is nearly constant and not worth over-engineering dashboards for.
- Don't over-invest in five-nines latency tracking (P99.99) for early-stage products — P95 is the industry-standard tail metric for a reason: it's stable enough to alert on without being dominated by single-digit outlier noise, and it correlates strongly enough with P99 for most sizing decisions.

#### Architecture

```
                         REQUEST LIFECYCLE — WHERE EACH METRIC IS MEASURED

  Client            Load Balancer /       Inference Server        Token Stream
  Request            API Gateway          (vLLM / TGI / API)      Back to Client
    |                     |                       |                     |
    |--- t0 ------------->|                       |                     |
    |                     |--- t1 --------------->|                     |
    |                     |                       |-- prefill/prompt -->|
    |                     |                       |   processing        |
    |                     |                       |-- 1st token ------->|
    |<---------------------------------- TTFT = (t_first_token - t0) ---|
    |                     |                       |-- token 2 -->|      |
    |                     |                       |-- token 3 -->|      |
    |                     |                       |    ...       |      |
    |                     |                       |-- token N -->|      |
    |<---------------- Total latency = (t_last_token - t0) --------------|

  Token cost   = (input_tokens  * price_in_per_1k / 1000)
               + (output_tokens * price_out_per_1k / 1000)

  Context cost = (input_tokens  * price_in_per_1k / 1000)     <- the piece you
                                                                   control via
                                                                   prompt design
```

#### Examples

**Beginner** — compute cost for a single request:

```python
def request_cost_usd(
    input_tokens: int,
    output_tokens: int,
    price_in_per_1k: float,
    price_out_per_1k: float,
) -> float:
    """Return the dollar cost of one LLM request."""
    input_cost = (input_tokens / 1000) * price_in_per_1k
    output_cost = (output_tokens / 1000) * price_out_per_1k
    return round(input_cost + output_cost, 6)


# Example from the module's baseline support-bot scenario:
# GPT-4-class pricing (illustrative — always check current provider pricing pages)
cost = request_cost_usd(
    input_tokens=12_000,
    output_tokens=500,
    price_in_per_1k=0.03,
    price_out_per_1k=0.06,
)
print(f"${cost:.4f} per request")  # ~$0.39 -- in the ballpark of the 36-cent baseline
```

**Intermediate** — a full request-level cost & latency model you can adapt into a spreadsheet or a monitoring dashboard query:

```python
from dataclasses import dataclass
from statistics import quantiles


@dataclass
class ModelPricing:
    name: str
    price_in_per_1k: float
    price_out_per_1k: float


@dataclass
class RequestSample:
    input_tokens: int
    output_tokens: int
    ttft_ms: float
    total_latency_ms: float


def cost_report(samples: list[RequestSample], pricing: ModelPricing) -> dict:
    total_cost = 0.0
    context_cost = 0.0
    ttfts, latencies = [], []

    for s in samples:
        in_cost = (s.input_tokens / 1000) * pricing.price_in_per_1k
        out_cost = (s.output_tokens / 1000) * pricing.price_out_per_1k
        total_cost += in_cost + out_cost
        context_cost += in_cost
        ttfts.append(s.ttft_ms)
        latencies.append(s.total_latency_ms)

    n = len(samples)
    p95_latency = quantiles(latencies, n=100)[94]     # 95th percentile
    p95_ttft = quantiles(ttfts, n=100)[94]

    return {
        "model": pricing.name,
        "n_requests": n,
        "avg_cost_usd": round(total_cost / n, 6),
        "context_cost_share_pct": round(100 * context_cost / total_cost, 1),
        "p95_ttft_ms": round(p95_ttft, 1),
        "p95_total_latency_ms": round(p95_latency, 1),
        "projected_monthly_cost_usd": round(total_cost / n * 20_000_000, 2),  # at 20M req/mo
    }
```

This is deliberately shaped like something you'd wire up to a Prometheus/Grafana pipeline (later modules cover the observability stack in depth) or run as a nightly batch job over logged request metadata — the point is that cost and latency reporting should be a first-class, queryable artifact, not a one-off spreadsheet someone built for a single meeting.

**Production-grade** — a cost-modeling "spreadsheet" you can hand to a team, expressed as a reusable module with named scenarios (this is the artifact a senior engineer produces during a capacity-planning or vendor-selection review):

```python
"""
cost_model.py — Adaptable LLM cost-modeling framework.

Usage: define one or more ScenarioConfig objects (baseline vs. optimized)
and diff them. This is the code-form of the "36 cents -> 11 cents" support-bot
optimization case study: model routing + context reduction + output capping.
"""
from dataclasses import dataclass, field


@dataclass
class ScenarioConfig:
    name: str
    monthly_requests: int
    avg_input_tokens: int
    avg_output_tokens: int
    price_in_per_1k: float
    price_out_per_1k: float
    cache_hit_rate: float = 0.0          # fraction served from semantic cache
    cache_hit_cost_usd: float = 0.0      # near-zero, e.g. Redis lookup cost
    prompt_cache_discount: float = 0.0   # fraction discount on cached input tokens
    prompt_cache_hit_share: float = 0.0  # fraction of input tokens that are cached prefix


def scenario_cost(cfg: ScenarioConfig) -> dict:
    cache_misses = cfg.monthly_requests * (1 - cfg.cache_hit_rate)
    cache_hits = cfg.monthly_requests * cfg.cache_hit_rate

    # Input cost, accounting for prompt-caching discount on the cached prefix share
    cached_in_tokens = cfg.avg_input_tokens * cfg.prompt_cache_hit_share
    uncached_in_tokens = cfg.avg_input_tokens - cached_in_tokens
    effective_in_cost_per_req = (
        (uncached_in_tokens / 1000) * cfg.price_in_per_1k
        + (cached_in_tokens / 1000) * cfg.price_in_per_1k * (1 - cfg.prompt_cache_discount)
    )
    out_cost_per_req = (cfg.avg_output_tokens / 1000) * cfg.price_out_per_1k
    full_cost_per_req = effective_in_cost_per_req + out_cost_per_req

    monthly_cost = (
        cache_misses * full_cost_per_req
        + cache_hits * cfg.cache_hit_cost_usd
    )
    return {
        "scenario": cfg.name,
        "cost_per_uncached_request_usd": round(full_cost_per_req, 5),
        "monthly_cost_usd": round(monthly_cost, 2),
        "monthly_requests": cfg.monthly_requests,
    }


if __name__ == "__main__":
    baseline = ScenarioConfig(
        name="baseline (GPT-4-class, no optimization)",
        monthly_requests=1_000_000,
        avg_input_tokens=12_000,
        avg_output_tokens=600,
        price_in_per_1k=0.03,
        price_out_per_1k=0.06,
    )
    optimized = ScenarioConfig(
        name="optimized (routing + semantic chunking + output cap + semantic cache)",
        monthly_requests=1_000_000,
        avg_input_tokens=4_000,          # semantic chunking: 12k -> 4k
        avg_output_tokens=150,           # capped output
        price_in_per_1k=0.0015,          # 80% of traffic routed to a cheaper model
        price_out_per_1k=0.002,
        cache_hit_rate=0.68,             # matches the 68% hit rate case study
        cache_hit_cost_usd=0.0001,
    )

    for cfg in (baseline, optimized):
        report = scenario_cost(cfg)
        print(report)
```

Running this prints a monthly-cost delta you can put directly into a design doc: the same shape of arithmetic that turns "36 cents per request" into "11 cents per request" in the source case study — model routing, context reduction, and output capping compound multiplicatively, not additively, which is why combining all three levers produces roughly a 70% reduction rather than three separate 20-30% reductions.

#### Comparison: What Each Metric Tells You (and What It Doesn't)

| Metric | Tells you | Doesn't tell you | Typical target |
|---|---|---|---|
| TTFT | Perceived responsiveness, "did it start" | Whether the full answer will finish in time | < 300-500ms for interactive chat |
| P95 total latency | Worst-case full completion time for the vast majority of users | Cost; a fast answer can still be expensive | < 2s for interactive; minutes are fine for batch/offline |
| Token cost | Dollar cost per request/month | Whether the spend is justified by conversion/retention | Set as a $/request budget threshold tied to unit economics |
| Context cost | Where your token spend is going (input vs. output) | Output quality — a small context can under-inform the model | Track as % of total cost; rising context-cost-share is an early warning sign of prompt bloat |

---

### 3.2 The Three Cost Levers — Model Size, Context Length, Output Length

#### Theory

Every dollar an LLM system spends decomposes into exactly three levers, and this decomposition is the single most useful mental model in this module because it turns "the AI is too expensive" — an unhelpful complaint — into three concrete, independently actionable engineering questions:

1. **Model size.** A bigger/more capable model costs more per token, generally because it requires more GPU-seconds (or a higher-priced API tier) to produce a token. The senior-engineer discipline here is **"right-sizing"**: start from the smallest model that clears your quality bar, and only escalate when you have evidence (an eval, not a hunch) that a bigger model is required for a specific task or slice of traffic. Do not default to the flagship model for every request — this is the single most common and most expensive mistake in LLM product engineering.
2. **Context length (input tokens).** Every token in the system prompt, chat history, and retrieved documents is priced and adds to prefill latency. Long contexts are a "silent" cost driver because they creep up gradually — someone adds "just one more example" to the system prompt, someone stops truncating chat history, a RAG pipeline starts retrieving 10 chunks instead of 3 — and nobody notices until the monthly bill spikes.
3. **Output length.** Tokens the model generates. Longer outputs cost more *and* take longer to stream to completion (unrelated to TTFT, which is unaffected by output length, but very relevant to total latency and to output-token spend). Capping `max_tokens` and instructing the model to be concise are cheap, high-leverage interventions.

**Why this framework, and not something else.** Anyone can say "reduce costs." The value of decomposing into exactly these three levers is that they map directly onto concrete engineering interventions:

| Lever | Concrete interventions |
|---|---|
| Model size | Task-based routing to smaller models; distillation; fine-tuning a small model to match a large model's quality on a narrow task; quantization |
| Context length | Semantic chunking/retrieval tuning; conversation summarization instead of full history replay; prompt caching for static prefixes; removing redundant few-shot examples |
| Output length | `max_tokens` caps; system-prompt instructions demanding concision; structured output schemas that bound response size (e.g., JSON with fixed fields vs. free-form prose) |

**When NOT to over-optimize each lever.** Right-sizing the model too aggressively (always defaulting to the cheapest model) risks quality regressions on the harder tail of your traffic — this is why routing (send *most* traffic to the cheap model, but *route* hard cases to the expensive one) beats blanket downgrading. Similarly, be careful capping output length on tasks that genuinely require long-form output (code generation, long-form writing) — an artificially low `max_tokens` there causes truncated, broken responses, which is a quality bug disguised as a cost optimization.

#ython Examples

**Beginner** — a routing function that picks model size based on a cheap complexity heuristic:

```python
def choose_model(query: str, retrieved_doc_count: int) -> str:
    """Very simple task-based router: cheap model by default, escalate on signals
    that suggest the task needs more capability."""
    long_query = len(query.split()) > 60
    many_docs = retrieved_doc_count > 5
    if long_query or many_docs:
        return "gpt-4-class"       # escalate: likely a complex, multi-step question
    return "gpt-3.5-class"          # default: fast, cheap, good enough for most FAQ-style asks
```

**Intermediate** — bounding context via summarization instead of raw history replay:

```python
def build_prompt(system_prompt: str, history: list[dict], user_msg: str,
                  max_history_tokens: int = 1000, summarizer=None) -> str:
    """Keep only the most recent turns verbatim; summarize the rest.
    This directly attacks the 'context length' cost lever."""
    recent, older = [], []
    running_tokens = 0
    for turn in reversed(history):
        turn_tokens = len(turn["content"]) // 4   # crude token estimate
        if running_tokens + turn_tokens <= max_history_tokens:
            recent.insert(0, turn)
            running_tokens += turn_tokens
        else:
            older.insert(0, turn)

    summary = ""
    if older and summarizer:
        summary = summarizer(older)   # a cheap, small-model summarization call

    history_block = (f"[Earlier conversation summary: {summary}]\n" if summary else "")
    history_block += "\n".join(f"{t['role']}: {t['content']}" for t in recent)

    return f"{system_prompt}\n\n{history_block}\n\nUser: {user_msg}"
```

**Production-grade** — a combined router + context-shrinker + output-cap enforcement layer, mirroring the "TURN / BALANCE / GUARDS / RESULT" cost-guardrail pattern from the source material:

```python
from dataclasses import dataclass


@dataclass
class RequestBudget:
    max_cost_usd: float = 0.02   # per-request budget threshold


def shrink_context(prompt_tokens: list[str], target_tokens: int) -> list[str]:
    """Placeholder for semantic-chunking/compression logic: in production this
    would call a retrieval re-ranker or a summarizer to compress `prompt_tokens`
    down to `target_tokens` while preserving the most relevant content."""
    return prompt_tokens[-target_tokens:]


def guarded_request(prompt_tokens: list[str], est_output_tokens: int,
                     price_in_per_1k: float, price_out_per_1k: float,
                     budget: RequestBudget) -> dict:
    def estimated_cost(n_in: int, n_out: int) -> float:
        return (n_in / 1000) * price_in_per_1k + (n_out / 1000) * price_out_per_1k

    # TURN: calculate cost per interaction, before sending anything
    cost = estimated_cost(len(prompt_tokens), est_output_tokens)

    # GUARDS: trigger controls when the request becomes too expensive
    if cost > budget.max_cost_usd:
        target = int(len(prompt_tokens) * (budget.max_cost_usd / cost) * 0.9)  # safety margin
        prompt_tokens = shrink_context(prompt_tokens, max(target, 256))
        cost = estimated_cost(len(prompt_tokens), est_output_tokens)

    # BALANCE + RESULT: cost is now a first-class decision input, not a
    # post-hoc report — this dict is what gets logged/dispatched downstream
    return {
        "final_prompt_tokens": len(prompt_tokens),
        "estimated_cost_usd": round(cost, 5),
        "within_budget": cost <= budget.max_cost_usd,
    }
```

The key production lesson embedded here: **cost control belongs inside the request path as a decision input, not in a nightly report someone reads after the money is already spent.** This is exactly the distinction between "engineering decisions and budget decisions are separate" (the anti-pattern) and "engineering manages latency, quality, and cost together" (the target state).

#### Comparison Table — The Three Levers

| Lever | Cost impact | Latency impact | Quality risk if over-tuned | Typical intervention |
|---|---|---|---|---|
| Model size | High — often 10-50x price difference between tiers | Larger models are typically slower per token too | Under-powered model fails complex/edge-case requests | Task-based routing + escalation path |
| Context length | Medium-high, scales linearly with tokens | Increases TTFT (longer prefill) | Truncating too aggressively loses needed information (RAG recall drops) | Semantic chunking, summarization, prompt caching |
| Output length | Medium, scales linearly with tokens | Increases total latency (more tokens to generate/stream) | Overly capped output truncates legitimate long-form answers | `max_tokens` caps + concision instructions + structured output |

---

### 3.3 Batching — Throughput Optimization for Grouped Workloads

#### Theory

**What it is.** Batching groups multiple inference requests so the GPU processes them together, amortizing fixed per-request overhead (kernel launch, memory movement) across more useful compute. **Continuous batching** (used by vLLM, TGI, and most modern serving stacks) goes further: instead of waiting for a fixed-size batch to fully complete before starting the next one, the server dynamically inserts new requests into GPU slots vacated by requests that finish generating, so GPU utilization stays high continuously rather than dipping every time a batch finishes at different times for different-length outputs (the "bubble" problem of static batching).

**What problem it solves.** Without batching, a GPU serving one request at a time leaves most of its compute capacity idle — LLM decoding is memory-bandwidth-bound, not compute-bound, at batch size 1, meaning the GPU can serve many concurrent decode steps for nearly the same wall-clock cost as one. Continuous batching alone is reported to yield roughly an order-of-magnitude throughput improvement over naive (static, one-at-a-time) batching, and combined with PagedAttention's efficient KV-cache memory management, gains compound further (see `references.md` for the specific published figures this course draws on).

**When to use it, when not to.** Batching is a serving-infrastructure-level concern (you get it largely "for free" by choosing a serving engine like vLLM that implements continuous batching, rather than hand-rolling it) *except* for genuinely offline/batch workloads — nightly report generation, bulk summarization, embedding backfills — where you explicitly group requests yourself and submit them as a batch job, because these workloads can tolerate queueing delay in exchange for much better $/token economics (many providers offer discounted "batch" API tiers for exactly this reason). Do NOT batch interactive, user-facing requests at the *application* layer by holding requests to accumulate a batch before sending — that reintroduces exactly the latency you're trying to eliminate. The right layering is: interactive traffic hits a continuously-batching inference server that batches invisibly and per-token; background/offline traffic is explicitly grouped and submitted as batch jobs.

#### Architecture

```
                    STATIC BATCHING (naive)                 CONTINUOUS BATCHING (vLLM-style)

  GPU timeline:                                        GPU timeline:
  [batch of 4 requests, all start together]             [request A starts]
  Req A: ####______  (short, finishes early,            [request B joins mid-flight]
                       GPU slot sits IDLE until           [request A finishes, slot freed]
                       the whole batch completes)         [request C immediately fills the
  Req B: ##########    (long)                              freed slot -- no idle bubble]
  Req C: #######___
  Req D: ####______
         |<-- next batch can't start until ALL 4    Result: GPU utilization stays near-
              finish, wasting the idle capacity      constant; new requests are admitted
              in the "_" gaps                        as soon as capacity frees up, not
                                                       only at fixed batch boundaries.
```

#### Examples

**Beginner** — explicit offline batch grouping for a non-interactive job:

```python
def batch_requests(items: list[str], batch_size: int = 8):
    """Group items for an offline summarization job -- fine to add latency here
    because there is no user waiting synchronously."""
    for i in range(0, len(items), batch_size):
        yield items[i:i + batch_size]

for batch in batch_requests(nightly_documents, batch_size=8):
    results = llm_client.batch_generate(batch)   # one round-trip per batch
```

**Intermediate/Production** — this is largely a *configuration*, not application-code, concern once you're on a serving engine with continuous batching built in:

```yaml
# serving-config.yaml
batching:
  max_batch: 8          # max requests grouped per scheduling step
  continuous: true       # enable continuous (in-flight) batching, not static
  max_wait_ms: 20        # cap on how long a request waits to join a batch

caching:
  semantic_cache: true

streaming:
  enabled: true
  first_token_target_ms: 350
```

This YAML is the literal shape referenced by the module's source material: each optimization technique is turned into an explicit, reviewable setting rather than left as an ambient property of the code. In production, this file (or its Kubernetes ConfigMap / Helm-values equivalent) is what an on-call engineer checks first when latency or cost drifts — "did someone change `max_batch` or disable the cache?" is a one-line diff to check, versus archaeology through application code.

---

### 3.4 Semantic Caching with Redis

#### Theory

**What it is.** Exact-match caching (same string in → same response out) rarely helps chat-style traffic because users phrase the same underlying question in dozens of different ways ("What's your refund policy?" / "Can I get my money back?" / "How do refunds work?"). **Semantic caching** instead embeds the incoming query into a vector, searches a vector store for previously-answered queries with high cosine/embedding similarity, and — if similarity clears a threshold — returns the cached answer without calling the LLM at all.

**What problem it solves.** For FAQ bots, support assistants, and any high-repeat-rate query surface, semantic caching converts a large share of traffic from "expensive LLM call" to "cheap vector lookup," attacking both latency (milliseconds vs. over a second) and cost (near-zero vs. full token price) simultaneously. The module's own worked example: a cached response returns in ~30ms vs. ~1200ms for a fresh generation, and after two weeks of production traffic the system reached a 68% cache hit rate, translating directly into a 68% cost reduction on that share of requests.

**Tradeoffs / when NOT to use it.** Semantic caching is a poor fit when: (a) responses must reflect fresh, request-specific state (e.g., "what's the status of *my* order #12345" — the answer is not reusable across users even if the *question* is semantically similar); (b) the similarity threshold is set too loosely, causing "confidently wrong" answers to near-miss queries — this is a subtle correctness bug, not just a performance tradeoff, so threshold tuning needs real evaluation, not a guess; (c) answers change frequently (pricing, policy that updates often) without a matching cache-invalidation strategy — a stale cached answer is worse than a slow correct one in many domains (legal, medical, financial).

#### Architecture

```
                         SEMANTIC CACHE LOOKUP FLOW

   User query: "Can I get my money back?"
        |
        v
  [Embedding model]  --embeds query into vector v--
        |
        v
  Redis: FT.SEARCH combined query
    - KNN vector similarity search over cached query embeddings
    - simultaneous TAG pre-filter on metadata (tenant_id, locale, model_version)
    - single round trip
        |
        v
  Best match found: "What's your refund policy?"  (similarity = 0.98)
        |
        +-- similarity >= threshold (0.95)? ------ YES -----> return cached
        |                                                       response
        |                                                       (~30ms)
        +-- NO -----> call LLM, generate fresh response (~1200ms)
                        |
                        v
                  write new entry to Redis Hash:
                    { prompt, response, embedding (float32 bytes),
                      tenant/locale/model_version tags, ttl, hit_count }
```

Each cache entry is stored as a single Redis **Hash** containing the raw prompt, the generated response, the raw float32 embedding bytes, tenant/locale/model-version metadata as tags, a TTL, and a hit counter — this is the reference pattern used by Redis's own official semantic-cache architecture, and it matters because it lets one `FT.SEARCH` call do vector similarity *and* metadata filtering (e.g., "only match cache entries for this tenant and this model version") in a single round trip, instead of a vector search followed by an application-side filter pass.

#### Code — Production-Grade Redis Semantic Cache

```python
"""
semantic_cache.py — Redis-backed semantic cache for LLM responses.

Requires: redis>=5.0 (redis-py with RediSearch/RedisJSON module support),
an embedding model client, and a Redis instance with the search module
(e.g., Redis Stack or Redis 8+ with search built in).
"""
import time
import numpy as np
import redis
from redis.commands.search.field import VectorField, TagField, TextField, NumericField
from redis.commands.search.query import Query
from redis.commands.search.indexDefinition import IndexDefinition, IndexType


class SemanticCache:
    INDEX_NAME = "idx:llm_semantic_cache"
    KEY_PREFIX = "cache:"

    def __init__(self, redis_client: redis.Redis, embed_fn, embedding_dim: int = 1536,
                 similarity_threshold: float = 0.95, ttl_seconds: int = 60 * 60 * 24 * 7):
        self.r = redis_client
        self.embed_fn = embed_fn
        self.embedding_dim = embedding_dim
        self.similarity_threshold = similarity_threshold
        self.ttl_seconds = ttl_seconds
        self._ensure_index()

    def _ensure_index(self):
        try:
            self.r.ft(self.INDEX_NAME).info()
            return  # index already exists
        except redis.exceptions.ResponseError:
            pass  # doesn't exist yet, create it

        schema = (
            TextField("prompt"),
            TextField("response"),
            TagField("tenant_id"),
            TagField("model_version"),
            NumericField("hit_count"),
            VectorField(
                "embedding",
                "HNSW",  # approximate nearest neighbor index
                {
                    "TYPE": "FLOAT32",
                    "DIM": self.embedding_dim,
                    "DISTANCE_METRIC": "COSINE",
                },
            ),
        )
        self.r.ft(self.INDEX_NAME).create_index(
            schema,
            definition=IndexDefinition(prefix=[self.KEY_PREFIX], index_type=IndexType.HASH),
        )

    def lookup(self, query: str, tenant_id: str, model_version: str) -> dict | None:
        vector = np.array(self.embed_fn(query), dtype=np.float32).tobytes()

        # Single round-trip: KNN vector search + TAG metadata pre-filter
        filter_expr = f"(@tenant_id:{{{tenant_id}}} @model_version:{{{model_version}}})"
        q = (
            Query(f"{filter_expr}=>[KNN 1 @embedding $vec AS score]")
            .sort_by("score")
            .return_fields("prompt", "response", "score", "hit_count")
            .dialect(2)
        )
        results = self.r.ft(self.INDEX_NAME).search(q, query_params={"vec": vector})

        if not results.docs:
            return None

        top = results.docs[0]
        similarity = 1 - float(top.score)  # COSINE distance -> similarity
        if similarity < self.similarity_threshold:
            return None

        self.r.hincrby(top.id, "hit_count", 1)
        return {"response": top.response, "similarity": round(similarity, 4), "cache_hit": True}

    def store(self, query: str, response: str, tenant_id: str, model_version: str):
        vector = np.array(self.embed_fn(query), dtype=np.float32).tobytes()
        key = f"{self.KEY_PREFIX}{tenant_id}:{int(time.time() * 1000)}"
        self.r.hset(key, mapping={
            "prompt": query,
            "response": response,
            "embedding": vector,
            "tenant_id": tenant_id,
            "model_version": model_version,
            "hit_count": 0,
        })
        self.r.expire(key, self.ttl_seconds)


# --- Usage ---
def handle_query(cache: SemanticCache, llm_client, query: str, tenant_id: str, model_version: str):
    hit = cache.lookup(query, tenant_id, model_version)
    if hit:
        return hit["response"]  # ~30ms path

    response = llm_client.generate(query)  # ~1200ms path
    cache.store(query, response, tenant_id, model_version)
    return response
```

Notice the metadata fields (`tenant_id`, `model_version`) are not decoration — without them, a multi-tenant system risks leaking Tenant A's cached answer to Tenant B, and a model upgrade risks serving stale answers generated by a retired model version. This is the most common bug in hand-rolled semantic caches: teams implement the vector-similarity half correctly and skip the metadata-scoping half, and it works fine in a single-tenant demo and silently misbehaves in production.

---

### 3.5 Prompt / Context Caching (Provider-Side)

#### Theory

Distinct from semantic caching (which caches *responses* to *similar* queries), **prompt caching** caches the *internal KV-cache state* of a repeated *exact* prompt prefix — typically a long system prompt, a fixed set of few-shot examples, or a large shared document — so that subsequent requests reusing that prefix skip re-running the transformer's forward pass over it. This is a provider/inference-engine-level optimization, not an application-level cache.

Anthropic, OpenAI, and Google all now offer this, with meaningfully different mechanics (exact thresholds and pricing are provider-specific and evolve — see `references.md` for current figures rather than treating any number here as permanent):

- **Anthropic** requires you to explicitly mark cache breakpoints in the prompt; cache writes cost a premium over base input price, cache reads cost a steep discount, so the pattern pays off once a cached prefix is reused more than once or twice within its TTL.
- **OpenAI** applies caching automatically once a prompt prefix crosses a length threshold, routed by a hash of the prefix, no code changes required for the basic case — though newer model families have introduced explicit cache breakpoints with their own write-premium economics.
- **Google Gemini** applies *implicit* context caching by default above a per-model minimum token threshold for its newer model family.

**Why it matters even though it looks like "just a provider feature."** As an LLMOps engineer, your job is to *design prompts so this optimization actually engages*: put static content (system prompt, shared instructions, shared reference documents) at the *front* of the prompt and put per-request variable content (the user's specific question) at the *end*. If you interleave static and dynamic content, or put the variable part first, you defeat prefix-based caching entirely regardless of which provider you use — this is a prompt-engineering discipline, not just a config toggle.

#### Example — Structuring a Prompt for Cache-Friendliness

```python
def build_cache_friendly_prompt(system_instructions: str, shared_reference_doc: str,
                                 user_question: str) -> list[dict]:
    """Static, reusable content goes first (cacheable prefix);
    per-request variable content goes last (never cached, always fresh)."""
    return [
        {"role": "system", "content": system_instructions},   # static -> cached
        {"role": "system", "content": shared_reference_doc},  # static -> cached
        {"role": "user", "content": user_question},            # dynamic -> not cached
    ]
```

If `system_instructions` + `shared_reference_doc` together exceed the provider's minimum cacheable-prefix threshold (order of ~1,000-2,000+ tokens depending on provider — see `references.md`), this structure lets every subsequent request in the same conversation, or across users sharing the same system prompt, reuse the cached prefix at a steep discount — the module's source material cites up to ~90% input-cost savings on the cached portion for high-reuse prefixes.

---

### 3.6 Streaming — Optimizing Perceived Latency

#### Theory

**What it is.** Instead of waiting for the entire response to be generated before sending anything to the client, the server pushes tokens to the client as they're generated. The client renders them incrementally (the now-familiar "typing" effect in chat UIs).

**What problem it solves.** Total latency (time to the *last* token) is often dominated by output length and is hard to shrink without hurting quality. But **perceived** latency — how fast the system *feels* — is dominated by TTFT. Streaming doesn't change total latency at all; it changes when the user starts *seeing progress*, which the module's source material characterizes as feeling roughly an order of magnitude faster even though the underlying compute is identical. This is a genuinely free win: streaming does not cost more compute than non-streaming (the model still generates the same tokens); it only changes *when* you flush them to the client.

**When to use it, when not to.** Use it for every interactive, human-facing surface: chat, copilots, assistants, anything with a human waiting synchronously. Do NOT bother streaming for backend-to-backend calls where nothing renders incrementally to a human (e.g., a service that needs the complete JSON response before it can proceed) — there, streaming adds protocol complexity (chunk reassembly, error handling mid-stream) with no perceptual benefit, since the "user" is another program that needs the full payload anyway.

#### Architecture

```
              NON-STREAMING                              STREAMING (SSE)

  Client        Server                        Client         Server
    |             |                              |              |
    |--request--->|                              |--request---->|
    |             |-- generating full response... |              |-- token 1 (280ms) -->|
    |             |   (3.2s of silence            |<--- render "token 1" -----------------|
    |             |    from client's POV)          |              |-- token 2 ----------->|
    |             |                                |<--- render "token 2" -----------------|
    |<--response--|  (all at once, at t=3.2s)      |              |    ... streaming ...   |
    |                                               |              |-- token N ----------->|
                                                     |<--- render "token N", stream closes --|
  Perceived wait: 3.2s of nothing                  Perceived wait: 280ms to first content,
                                                    then continuous visible progress
```

#### Code — FastAPI Server-Sent Events Streaming Endpoint

FastAPI now ships an official, first-class SSE tutorial pattern using `EventSourceResponse`/`ServerSentEvent`, which handles keep-alives, cache-prevention headers, and proxy-buffering-prevention for you — superseding older hand-rolled `StreamingResponse`-based SSE that required setting those headers manually.

```python
"""
streaming_server.py — FastAPI SSE endpoint that streams LLM tokens to the client
as they are generated, minimizing perceived latency (TTFT-driven UX).
"""
import asyncio
import json
import time
from fastapi import FastAPI
from fastapi.responses import EventSourceResponse  # official FastAPI SSE support

app = FastAPI()


async def token_generator(prompt: str):
    """Simulates an LLM streaming client. In production this wraps the
    provider SDK's streaming call (e.g., client.messages.stream(...) or
    an httpx streaming request against a self-hosted vLLM OpenAI-compatible
    endpoint) and yields tokens as they arrive from the model."""
    start = time.perf_counter()
    tokens = generate_tokens_from_model(prompt)  # your model call, yields str chunks

    first_token_logged = False
    async for token in tokens:
        if not first_token_logged:
            ttft_ms = (time.perf_counter() - start) * 1000
            first_token_logged = True
            # emit TTFT as a monitoring side-channel, not to the client
            log_ttft_metric(ttft_ms)

        yield {"event": "token", "data": json.dumps({"text": token})}

    yield {"event": "done", "data": json.dumps({"status": "complete"})}


@app.get("/chat/stream")
async def chat_stream(prompt: str):
    return EventSourceResponse(token_generator(prompt))


# --- Client-side consumption (browser JS, for reference) ---
#
# const source = new EventSource(`/chat/stream?prompt=${encodeURIComponent(q)}`);
# source.addEventListener("token", (e) => {
#     const { text } = JSON.parse(e.data);
#     appendToChatBubble(text);   // incremental render -- the perceived-speed win
# });
# source.addEventListener("done", () => source.close());
```

```python
# A minimal, dependency-free async generator stand-in for `generate_tokens_from_model`,
# useful for local testing without a real model:
async def generate_tokens_from_model(prompt: str):
    fake_tokens = ["Sure, ", "let ", "me ", "help ", "with ", "that."]
    for tok in fake_tokens:
        await asyncio.sleep(0.05)  # simulate inter-token latency (ITL)
        yield tok
```

Two production notes that matter beyond the happy path:

1. **TTFT and ITL should still be measured server-side**, even though the point of streaming is to hide latency from the user — you need the real numbers for your dashboards and SLOs; "the user didn't notice" is not the same as "it was actually fast," and regressions in backend generation speed will eventually erode even a well-streamed UX.
2. **Reverse proxies can silently defeat streaming** by buffering the response before forwarding it (a classic nginx/Kubernetes-ingress gotcha). If you deploy this behind an ingress controller (from Module 05), you must disable response buffering for the streaming route (e.g., `proxy_buffering off` on nginx, or the equivalent annotation on your ingress class) or you will ship code that streams correctly in local dev and arrives all-at-once in production.

---

### 3.7 API vs. Local Deployment

#### Theory

**What it is.** The decision between calling a hosted, third-party model API (OpenAI, Anthropic, Google, etc.) versus self-hosting an open-weight model on your own (or your cloud provider's) GPU infrastructure using a serving engine like vLLM or TensorRT-LLM.

**What problem it solves / why it's hard.** This is not a "which is better" question — it is a multi-variable optimization over four independent axes:

1. **Request volume** — API pricing is per-token with no fixed cost; self-hosting has a large fixed cost (GPU capacity, whether idle or busy) plus a small marginal cost per request. At low volume, fixed self-hosting cost dominates and API wins; at high volume, the per-token API cost eventually exceeds the amortized fixed cost and self-hosting wins. The module's source material frames this break-even around the 20-million-requests/month range (with API being the clearer choice below ~10M/month and self-hosting the clearer choice above ~50M/month, with a genuinely mixed zone in between) — treat this as an illustrative order-of-magnitude anchor to reason from, not a universal constant; your actual break-even depends heavily on your specific token lengths, GPU pricing, utilization efficiency, and negotiated API rates.
2. **Compliance / data residency** — in regulated industries (healthcare, finance, government), sending certain data categories to a third-party API may be legally prohibited regardless of cost economics. Here, local deployment is not a cost optimization; it's a hard constraint that overrides the break-even math entirely.
3. **Customization needs** — fine-tuning control, model architecture changes, or the need to run a specific open-weight model not available via any API push toward self-hosting.
4. **Operational maturity** — self-hosting requires a team capable of running GPU infrastructure: capacity planning, autoscaling, model-serving upgrades, on-call for inference-server incidents. A company with strong economic and compliance reasons to self-host, but no GPU/SRE capability, may still rationally choose API in the short term while building that capability — "we should self-host" and "we can self-host reliably today" are different claims, and conflating them is a common strategic mistake.

**When to use API, when to self-host, when NOT to do either prematurely.**

- Use API when: prototyping, need latest frontier models, traffic is low-to-moderate, team lacks GPU ops experience, time-to-market matters more than marginal cost.
- Use local when: traffic is very high, compliance mandates data residency, you need deep customization (fine-tuning, custom architectures), and you have (or are building) real GPU/SRE operational capability.
- Don't self-host prematurely just because "it'll be cheaper eventually" — the operational burden (upgrading serving engines, handling GPU failures, capacity planning for traffic spikes) is real engineering cost that a spreadsheet comparing only $/token vs. $/GPU-hour will miss entirely.

#### Architecture — Decision Framework

```
                    API vs. LOCAL DEPLOYMENT DECISION FRAMEWORK

        Is there a hard compliance/data-residency requirement?
                        |
                 YES ---+--- NO
                  |            |
                  v            v
          LOCAL DEPLOYMENT   Is monthly volume > ~20-50M requests
          (required,          AND do you have GPU/ops capability?
           regardless                    |
           of cost)                YES --+-- NO
                                    |          |
                                    v          v
                              LOCAL         Use API now;
                              DEPLOYMENT     build GPU/ops
                              (cost-driven)  capability if
                                             volume trend
                                             continues, then
                                             re-evaluate
                                             |
                                             v
                                     Is traffic mixed
                                     complexity (some hard,
                                     mostly simple)?
                                             |
                                      YES ---+--- NO
                                       |            |
                                       v            v
                                 HYBRID:        Stay on API
                                 route simple
                                 -> local,
                                 complex/edge
                                 -> API fallback
```

#### Comparison Table — Deployment Options

| Dimension | Hosted API (OpenAI/Anthropic/Google) | vLLM (self-hosted) | TensorRT-LLM (self-hosted) | Ollama (self-hosted) |
|---|---|---|---|---|
| Best for | Prototyping, frontier model access, low-to-moderate volume, teams without GPU ops | Production-scale self-hosted serving; the de facto default choice for self-hosting in 2026 | Maximum throughput/latency on NVIDIA hardware when you can afford compile-time cost and vendor lock-in | Local development, single-user/small-team use, quick experimentation — not designed for multi-tenant production throughput |
| Setup complexity | Lowest — API key and go | Moderate — needs GPU provisioning, K8s/Docker packaging (build on Module 04-05 skills) | High — requires compiling model-specific engines ahead of time, NVIDIA-only | Very low — single command, but that simplicity is exactly why it doesn't scale to production multi-tenant traffic |
| Throughput/latency profile | Provider-managed; generally good but you don't control the batching/scheduling internals | Strong out-of-the-box via continuous batching + PagedAttention; broad hardware support | Highest raw throughput on NVIDIA GPUs (compiled engines), reported ~15-30% higher than vLLM in benchmarks, at the cost of iteration speed and portability | Fine for one request at a time; not built for concurrent multi-user serving |
| Cost model | Per-token, no fixed infra cost; expensive at very high volume | Fixed GPU cost + ops engineering time; cheaper per-request at high, sustained volume | Same fixed-cost model as vLLM, optimized further for raw efficiency per GPU-hour | Fixed cost of whatever machine it runs on; not a serious production cost lever either way |
| Compliance / data residency | Data leaves your infrastructure (subject to provider's data-handling terms) | Full control — data never leaves your infrastructure | Full control | Full control (but again, not production-shaped) |
| Customization | Limited to what the provider exposes (fine-tuning APIs, system prompts) | Full control over model choice, quantization, custom kernels, fine-tuned weights | Full control, plus deep compiler-level optimization | Full control, limited by its simpler serving model |
| Operational burden | Near-zero — provider manages scaling/availability | Real — you own capacity planning, upgrades, GPU failures, autoscaling | Higher still — engine recompilation on model/config changes | Low, but that's because it isn't solving production-scale problems |
| Note (2026 context) | Increasingly offers built-in prompt caching, batch discounts | Now the production-default self-hosting choice; Hugging Face's own TGI has moved to maintenance mode in favor of vLLM/SGLang | Still the throughput ceiling for NVIDIA-committed shops willing to pay the lock-in and compile-time cost | Excellent onboarding/dev tool; do not reach for it as your production multi-tenant serving layer |

#### Example — Hybrid Deployment Configuration

```yaml
# deployment.yaml
mode: api                      # primary serving path
provider: openai
fallback: local-vllm           # backup path if primary is unavailable/unsuitable

requirements:
  p95_latency_target_ms: 1200
  data_residency_required: false
  gpu_team_available: false    # <- honest capability assessment, not aspiration
```

This is deliberately the kind of file that should live in version control next to your service (following the same versioning discipline from Module 02): the *reasoning* behind a deployment choice — traffic assumptions, compliance posture, latency SLO, team capability — becomes an explicit, reviewable artifact rather than tribal knowledge that evaporates when the person who made the decision leaves the team.

#### Example — Hybrid, Tiered-Routing Production Architecture

```python
"""
hybrid_router.py — Task-complexity-based routing across a hosted API and
a self-hosted local model, with an API fallback for edge cases the local
model can't handle. Mirrors the e-commerce hybrid case study: ~5% of
traffic to a premium API, ~80% to a cheap local model, ~15% falling back
to a cheaper hosted API tier when local confidence is low.
"""
from enum import Enum


class Route(Enum):
    PREMIUM_API = "premium_api"      # e.g., GPT-4-class -- complex, high-value tasks
    LOCAL_MODEL = "local_vllm"        # e.g., Llama-3-class self-hosted -- high-volume, simple
    FALLBACK_API = "fallback_api"     # e.g., GPT-3.5-class -- edge cases local can't handle


def classify_complexity(request: dict) -> str:
    """Placeholder for a real classifier: could be a small model, a set of
    heuristics, or a confidence score from the local model's own output."""
    if request.get("task_type") == "product_recommendation":
        return "complex"
    if request.get("local_model_confidence", 1.0) < 0.6:
        return "edge_case"
    return "simple"


def route_request(request: dict) -> Route:
    complexity = classify_complexity(request)
    if complexity == "complex":
        return Route.PREMIUM_API
    if complexity == "edge_case":
        return Route.FALLBACK_API
    return Route.LOCAL_MODEL


def handle(request: dict, clients: dict):
    route = route_request(request)
    try:
        return clients[route].generate(request)
    except Exception:
        # resilience: if local serving is down/degraded, fail over to hosted API
        if route == Route.LOCAL_MODEL:
            return clients[Route.FALLBACK_API].generate(request)
        raise
```

The case study this mirrors reports roughly 60% cost savings versus full-API usage, plus improved uptime (reported around 99.2%) precisely because the fallback path adds resilience: a local-model outage degrades to a hosted-API response rather than a hard failure.

---

## 4. Real-World Case Studies (Reasoned Inference)

The following are *plausible architectures*, reasoned from publicly known engineering-blog patterns and industry norms for companies operating at this scale — not confirmed internal implementation details. Treat them as worked examples of how to apply this module's frameworks, not as verified facts about any specific company's stack.

**A conversational AI product at OpenAI/Anthropic scale.** A consumer chat product serving hundreds of millions of requests would almost certainly rely heavily on prompt/context caching for system prompts and tool-definitions that are identical across huge numbers of requests — this is exactly the economic case prompt caching was built for, and it would plausibly be layered with continuous-batching inference infrastructure (conceptually similar to what vLLM implements, though a frontier-lab-scale provider would likely run heavily customized internal serving stacks rather than vanilla open-source vLLM) and aggressive streaming for perceived latency on every interactive surface. Given the scale involved, TTFT would likely be treated as a top-line product metric monitored as rigorously as uptime.

**Google Gemini.** Google's own documented default of implicit context caching for Gemini 2.5+ models reflects exactly the "make the optimization ambient rather than opt-in" philosophy this module advocates — at Google's traffic volume, a huge share of requests plausibly share long, static context (search grounding instructions, safety system prompts, tool schemas), making automatic prefix caching a very high-leverage default rather than a per-team opt-in feature.

**Netflix / Spotify-style recommendation-adjacent LLM features.** A company like this, layering LLM-based features (e.g., natural-language search, playlist/content description generation) on top of an existing massive-scale recommendation infrastructure, would plausibly apply the hybrid-routing pattern from section 3.7 particularly aggressively: the vast majority of requests (browsing, simple queries) would likely be served by small, cheaply self-hosted or fine-tuned models integrated tightly with their existing feature infrastructure, reserving frontier hosted-API calls for a small slice of genuinely open-ended natural-language tasks — this mirrors the e-commerce hybrid case study's 80/15/5 traffic split almost exactly, and matches the general industry pattern of "premium model for the long tail of hard requests, cheap model for the high-volume common case."

**Uber / Amazon-style operational-scale companies.** With enormous internal request volumes for tasks like support-ticket triage, logistics text classification, and internal tooling, these organizations sit clearly on the self-hosting side of the break-even framework in section 3.7 for their highest-volume workloads, while plausibly continuing to use hosted frontier APIs for lower-volume, higher-complexity, or rapidly-evolving use cases (e.g., a new internal copilot prototype) where engineering the self-hosted path isn't yet justified by volume. This is the "start with API, migrate to local as volume and organizational GPU/ops maturity grow" trajectory from section 3.7 playing out at enterprise scale.

**Databricks / NVIDIA.** As vendors *of* the infrastructure this module teaches (Databricks publishes LLM inference performance-engineering guidance; NVIDIA builds TensorRT-LLM and Triton), these companies' own internal LLM-powered products (e.g., a coding/data assistant embedded in a platform) would plausibly serve as reference implementations of the very optimization stack their public engineering content describes — continuous batching, KV-cache-aware memory management, and (for NVIDIA specifically) a strong pull toward compiled TensorRT-LLM engines given their obvious hardware alignment advantage.

**A healthcare or financial-services company (compliance-driven case).** Regardless of request volume or cost economics, an organization processing protected health information or regulated financial data would plausibly be forced toward local/self-hosted deployment (or a specific compliant private-cloud API offering with contractual data-handling guarantees) purely by regulatory requirement — this is the "compliance first" principle from section 3.7 overriding the cost-break-even math entirely, and it's a scenario every senior LLMOps engineer should be able to recognize and architect for without being told twice.

---

## 5. Common Mistakes

1. **Optimizing average latency instead of P95/P99.** A system with a great mean latency and a terrible tail still produces angry users and paging alerts — the tail is where production incidents live.
2. **Treating the flagship/largest model as the default for every request.** This is the single most expensive default in LLM product engineering; it should be an escalation path, not the starting point.
3. **Letting context grow silently.** Chat history that's never truncated or summarized, RAG retrieval that returns more chunks "just to be safe," system prompts that accumulate one-off instructions over months — these compound into large, invisible cost and latency regressions.
4. **Building a semantic cache without metadata scoping.** Forgetting tenant/model-version/locale tags in the cache key risks cross-tenant data leakage and stale answers served after a model upgrade.
5. **Setting the semantic-cache similarity threshold without evaluation.** A threshold picked by eyeballing a few examples, rather than validated against a labeled set of true-positive/false-positive matches, silently degrades answer quality while looking like a pure performance win on a dashboard.
6. **Streaming that gets buffered by a reverse proxy.** Code that streams correctly on localhost and arrives as one giant chunk in production because an nginx/ingress layer buffers responses by default — always test streaming behavior through the actual production network path, not just directly against the app server.
7. **Treating cost control as a reporting exercise instead of a request-time decision.** Computing cost after the fact in a monthly dashboard means you find out about a runaway cost pattern weeks after it started; budget guardrails need to be inline, request-time logic (as in the `guarded_request` example).
8. **Choosing local deployment based on "it should be cheaper eventually" without an honest operational-capability assessment.** Self-hosting an inference stack you don't have the GPU/SRE maturity to run reliably trades a predictable API bill for unpredictable incident load — often a worse trade even when the raw unit economics favor self-hosting on paper.
9. **Ignoring prompt-caching prefix ordering.** Interleaving static and dynamic prompt content, or putting the variable user question before the static system instructions, silently defeats provider-side prompt caching regardless of which provider you use.
10. **Capping output length uniformly across all task types.** A `max_tokens` cap tuned for FAQ answers will truncate and corrupt legitimate long-form outputs like generated code or long-form documents — cap per task type, not globally.

---

## 6. Best Practices and Production Tips

**When to use / when NOT to use each technique — quick reference:**

| Technique | Use when | Avoid / be cautious when |
|---|---|---|
| Semantic caching | High query-repetition surfaces (FAQ, support, docs Q&A) | Responses are user/request-specific state, or answers change frequently without invalidation |
| Prompt caching | Long, static, frequently-reused prompt prefixes (system prompts, shared docs) | Prompts are highly variable with little reusable prefix structure |
| Streaming | Any interactive, human-facing surface | Backend-to-backend calls needing the complete payload before proceeding |
| Batching (explicit) | Offline/background jobs tolerant of queueing delay | Interactive user-facing requests (let the serving engine's continuous batching handle this invisibly instead) |
| Model routing | Traffic has a clear complexity gradient (mostly simple, some hard) | Task quality requirements are uniformly high across all traffic (routing adds little value, and adds a classifier failure mode) |
| Local/self-hosted deployment | High sustained volume, compliance mandate, deep customization need, real GPU/ops capability | Low/uncertain volume, no compliance driver, no GPU/ops team yet |

**Scaling:** design your cost model (section 3.1) to project cost *linearly* against your growth forecast, not just current volume — the break-even analysis in section 3.7 changes as volume grows, so revisit the API-vs-local decision on a schedule (e.g., quarterly) rather than treating it as decided once and forever.

**Monitoring:** TTFT, P95/P99 total latency, cache hit rate, and cost-per-request should be first-class dashboard panels and alertable SLOs, not ad hoc queries someone runs when a Slack complaint comes in. (Later modules in this course build out the full observability stack — Prometheus/Grafana dashboards and distributed tracing — that these metrics feed into.)

**Security:** semantic cache entries containing user data are a data-handling surface like any other — apply the same access controls, encryption-at-rest, and retention/TTL discipline you'd apply to any datastore holding user content, and make sure cache keys are scoped so no tenant can read another tenant's cached responses.

**Performance tradeoffs:** every technique in this module trades *something* — cache staleness risk for speed/cost, routing-classifier complexity for cost savings, self-hosting operational burden for lower marginal cost. Make these tradeoffs explicit in a design doc (like the `deployment.yaml`/`serving-config.yaml` patterns shown above) rather than implicit in code that nobody remembers the reasoning behind six months later.

**Alternatives worth knowing about:** SGLang is an increasingly common alternative to vLLM in the same "high-performance open-source self-hosted serving" category as of 2026 (see `github.md`/`references.md` for pointers); Triton Inference Server is commonly used as a general-purpose front for compiled TensorRT-LLM engines when you need multi-framework model serving beyond just LLMs.

---

## 7. Interview Questions

**Q1: Why do production teams track P95/P99 latency instead of average latency?**
*Model answer:* Average latency hides the tail of the distribution, and LLM serving latency is often multimodal — most requests are fast, but requests that land behind a full batch, hit a cold instance, or trigger KV-cache pressure can be several times slower. P95/P99 surfaces the experience of the worst-affected users, which is what drives complaints and SLA violations, and it's a more actionable number for capacity planning than a mean that could stay flat while the tail worsens.

**Q2: A team says "just use GPT-4 for everything, it's the best model." What's wrong with this reasoning from a production standpoint?**
*Model answer:* Model size is one of three cost levers, and "better performance" from a larger model is not free — it should only be paid for when the accuracy gain is actually needed for that specific task. The right pattern is task-based routing: send the bulk of simple/common requests to a smaller, cheaper model, and escalate only the subset that genuinely needs more capability, validated by an eval, not a hunch.

**Q3: Explain the difference between semantic caching and prompt/context caching. When would you use each?**
*Model answer:* Semantic caching caches complete *responses*, keyed by embedding similarity between the current query and previously-seen queries — good for high-repetition query surfaces like FAQ bots, where paraphrases of the same question should hit the same cached answer. Prompt/context caching is a provider- or engine-level mechanism that caches the internal KV-cache state of a repeated *exact* prompt prefix (e.g., a long system prompt), so repeated requests sharing that prefix skip reprocessing it — useful whenever you have a large, static, frequently-reused prompt segment, regardless of how varied the per-request user input is. They compose well together: cache the response when possible, and cache the shared prefix when it isn't.

**Q4: What's the risk of setting a semantic cache's similarity threshold too low, and how would you validate the right threshold?**
*Model answer:* Too low a threshold causes near-miss queries to be served a cached answer that's actually wrong for the specific question asked — a correctness bug that looks like a performance win on a latency/cost dashboard. The right approach is to validate the threshold against a labeled evaluation set of true-positive and false-positive semantic matches (not eyeballing a few examples), tuning for an acceptable false-positive rate given the domain's tolerance for a wrong-but-plausible-sounding answer.

**Q5: Why does streaming improve user experience without increasing total system cost?**
*Model answer:* Streaming changes only *when* generated tokens are flushed to the client, not how many tokens are generated or how much compute is used — the model still produces the same output. Because perceived responsiveness is dominated by TTFT (when does the user see something) rather than total latency (when is everything done), streaming can make a response feel dramatically faster without any change in the underlying generation cost.

**Q6: How would you decide between calling a hosted LLM API and self-hosting an open-weight model with vLLM?**
*Model answer:* Evaluate four axes: request volume (API tends to win below roughly 10M requests/month, self-hosting above roughly 50M/month, with a break-even zone around 20M/month as a rough anchor); compliance/data-residency requirements (can override the cost math entirely); customization needs (fine-tuning, specific open-weight models); and operational maturity (do you actually have the GPU/SRE capability to run inference infrastructure reliably). A common trajectory is starting on API for speed-to-market, then migrating specific high-volume workloads to self-hosted infrastructure as both volume and organizational capability grow — often ending in a hybrid architecture rather than an all-or-nothing choice.

**Q7: Describe a hybrid deployment architecture and explain why it often beats a pure API or pure self-hosted approach.**
*Model answer:* A hybrid architecture routes requests by task complexity: a small share of high-value, complex requests go to a premium hosted API; the bulk of high-volume, simple requests are served by a cheaper self-hosted model; and a fallback path routes edge cases the local model can't confidently handle to a mid-tier hosted API, which also provides resilience if local serving degrades. This captures most of the cost savings of self-hosting on the high-volume common case while preserving quality on hard cases and adding failover — a worked example achieves roughly 60% cost savings versus full API usage with high uptime, versus either extreme of "always call the expensive API" or "force every request through a model that can't handle the hard tail."

**Q8: Why should cost-budget enforcement happen inside the request path rather than in a post-hoc monthly report?**
*Model answer:* A monthly report tells you a cost problem existed weeks after it started accumulating; by the time anyone reads the report, the spend has already happened. Inline, request-time budget checks (estimate cost, compare to a threshold, shrink context or reroute if over-budget before the expensive call is made) turn cost control into an active system-design decision rather than a passive accounting exercise, catching runaway cost patterns before they compound across millions of requests.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

A production LLM system is judged on three axes simultaneously — quality, speed, and cost — and this module gave you the vocabulary (TTFT, P95 latency, token/context cost), the cost-decomposition framework (model size, context length, output length), the optimization toolkit (batching, semantic caching, prompt caching, streaming), and the deployment decision framework (API vs. local vs. hybrid) to reason about all three at once instead of optimizing one at the expense of the others.

### Key Takeaways

- TTFT drives *perceived* speed; P95 (not average) total latency drives *tail* user experience; token and context cost drive the bill.
- The three cost levers — model size, context length, output length — are where every optimization ultimately acts; combining interventions across all three compounds savings (the ~36-cents-to-11-cents pattern), rather than each acting in isolation.
- Semantic caching, prompt caching, streaming, and batching each solve a *different* part of the latency/cost problem; production systems use all four together, applied to the traffic pattern each best fits.
- The API-vs-local deployment decision is a function of volume, compliance, customization, and operational maturity — not model preference — and the strongest production architectures are usually hybrid, routing by task complexity.
- Cost control and latency control belong inside the system's decision-making logic (request-time guardrails, explicit serving configs), not as after-the-fact reports.

### Production Checklist

- [ ] TTFT, P95/P99 total latency, cost-per-request, and cache-hit-rate are instrumented and dashboarded (not just spot-checked).
- [ ] A request-level cost model exists and is used to project monthly spend against realistic traffic forecasts.
- [ ] Model routing exists for any traffic with a meaningful complexity gradient; the largest/most expensive model is not the default for all requests.
- [ ] Context length is actively managed (summarization, chunking, retrieval tuning) rather than allowed to grow unbounded over time.
- [ ] Output length is capped per task type, with caps validated against legitimate long-form use cases so nothing gets truncated incorrectly.
- [ ] Semantic cache (if used) has metadata-scoped keys (tenant/model-version/locale), a validated similarity threshold, and an explicit invalidation/TTL strategy.
- [ ] Prompt structure places static content first and dynamic content last, to make prompt/context caching actually engage.
- [ ] Streaming is enabled on all interactive, human-facing endpoints, and verified end-to-end through the actual production network path (proxy buffering disabled).
- [ ] Batching strategy is explicit: continuous batching for interactive traffic (engine-level), explicit batch grouping for offline/background jobs.
- [ ] The API-vs-local-vs-hybrid deployment decision is documented (e.g., a `deployment.yaml`-style artifact) with the volume, compliance, customization, and operational-maturity reasoning behind it, and is revisited on a schedule as volume grows.
- [ ] Cost budget guardrails run inline in the request path, not only in a post-hoc monthly report.

---

## 9. Further Reading

Detailed citations, links to official documentation (vLLM metrics design, Anthropic/OpenAI/Google prompt-caching docs, Redis semantic-cache reference architecture, FastAPI SSE tutorial, the PagedAttention paper, and more), curated GitHub repositories, and supplementary video/book recommendations for this module are collected in the companion files in this same folder:

- `references.md` — official docs, papers, and blog posts
- `github.md` — curated repositories (vLLM, TensorRT-LLM, GPTCache, RedisVL, Triton, and more)
- `videos.md` — supplementary video resources
- `books.md` — supplementary book recommendations

Refer to those files for source links rather than expecting them repeated here.
