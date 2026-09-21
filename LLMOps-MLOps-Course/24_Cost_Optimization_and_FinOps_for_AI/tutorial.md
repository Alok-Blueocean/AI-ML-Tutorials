# Cost Optimization and FinOps for AI

## What This Topic Is

FinOps is short for "Financial Operations" — a practice that brings engineering, finance, and business teams together to understand and control cloud spending. **FinOps for AI** applies the same discipline to the two dominant cost drivers in modern AI systems: **tokens** (for LLM API calls) and **GPUs** (for training, fine-tuning, and self-hosted inference).

In traditional software, compute cost is fairly predictable — you provision a server and pay a steady rate. In AI systems, cost is *usage-driven and highly variable*: every prompt, every generated token, every model call adds up, and a single careless change (a verbose system prompt, an unbounded retry loop, an oversized model for a simple task) can multiply your bill overnight. FinOps for AI is about making that spending visible, predictable, and intentional.

## Why It Matters

- **Costs scale with usage, not headcount.** A successful AI feature that gets adopted by more users doesn't just need more support — it directly costs more in tokens or GPU-hours. Without monitoring, success can silently become unsustainable.
- **Small inefficiencies compound.** Sending 500 extra tokens of context per request seems trivial, but multiplied across millions of requests per month, it becomes a real line item.
- **AI budgets are still new territory.** Many organizations don't yet have mature guardrails for LLM spend the way they do for traditional cloud infrastructure, so costs can spiral before anyone notices.
- **Trade-offs are real.** Cheaper models or shorter contexts can reduce quality; the goal of FinOps isn't just "spend less" — it's spending in a way that matches business value.

## Main Concepts in Plain Terms

### 1. Token Economics
LLM API providers typically charge per **token** (a token is roughly a chunk of a word), usually with separate rates for **input tokens** (what you send) and **output tokens** (what the model generates) — output is often priced higher than input. Your total cost per request is approximately:

```
cost ≈ (input_tokens × input_price) + (output_tokens × output_price)
```

Key levers that affect this:
- **Prompt length** — system prompts, few-shot examples, and retrieved context (in RAG) all consume input tokens.
- **Conversation history** — in multi-turn chats, the entire history is often resent with every new message, so cost grows as conversations get longer.
- **Output verbosity** — asking a model to "explain in detail" costs more than asking for a concise answer.

### 2. GPU Economics
If you're training, fine-tuning, or self-hosting models, cost is driven by **GPU-hours**: how many GPUs you rent/own, for how long, and how efficiently you use them. Key ideas:
- **Utilization matters more than raw GPU count.** An idle or underused GPU is wasted money — batching requests and right-sizing instances improves utilization.
- **Training vs. inference costs differ.** Training is a large, one-time (or periodic) cost; inference is an ongoing, usage-driven cost that usually dominates total spend over a model's lifetime.
- **Model size vs. hardware needs.** Larger models need more (and pricier) GPU memory and compute, so model choice directly drives infrastructure cost.

### 3. Common Cost-Optimization Levers
These are the practical tools teams reach for, roughly in order of typical impact:

| Lever | What It Does |
|---|---|
| **Right-size the model** | Use the smallest/cheapest model that meets quality requirements for a given task (e.g., a small model for classification, a large one only for complex reasoning). |
| **Prompt trimming** | Remove unnecessary context, shorten system prompts, and summarize long history instead of resending it in full. |
| **Caching** | Reuse results for repeated or near-identical requests (exact-match caching or semantic caching) instead of calling the model again. |
| **Batching** | Group multiple requests together to improve GPU utilization for self-hosted models. |
| **Response limits** | Cap `max_tokens` on output to avoid unexpectedly long (and costly) generations. |
| **Routing / model cascades** | Send easy queries to a cheap model first, and only escalate to an expensive model when needed. |
| **Quantization** | Run models in lower precision (e.g., 8-bit instead of 16-bit) to cut GPU memory and compute needs, usually with a small quality trade-off. |
| **Autoscaling & spot instances** | Scale infrastructure down during low traffic and use discounted/spot compute where interruption is tolerable. |

### 4. FinOps Practices (the "Ops" Part)
Optimization only works if it's continuous. Common practices include:
- **Tagging and attribution** — track spend per team, product, or feature so costs are visible and accountable.
- **Budgets and alerts** — set thresholds that trigger notifications (or automatic throttling) before costs run away.
- **Regular cost reviews** — periodically revisit which models/prompts are used where, since better/cheaper options appear frequently.
- **Cost-per-outcome metrics** — instead of just "total spend," track cost per resolved ticket, per document processed, per user session, etc., so cost is tied to business value.

## A Simple Example

A minimal way to enforce basic cost controls when calling an LLM API — capping output length and picking a cheaper model for simple tasks:

```python
def call_llm(prompt, task_complexity="simple"):
    # Model routing: use a cheaper model unless the task needs more power
    model = "small-model" if task_complexity == "simple" else "large-model"

    response = llm_client.chat(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        max_tokens=300,       # cap output length to avoid runaway costs
        temperature=0.2,
    )
    return response
```

Even this small pattern — routing by task complexity and capping `max_tokens` — is a real-world FinOps lever: it directly limits the two biggest cost multipliers (model price and output length) without any complex tooling.

## Key Takeaways / Best Practices

- **Treat token and GPU usage as a first-class metric**, tracked alongside latency and accuracy — not an afterthought discovered at invoice time.
- **Right-size before you optimize** — often the biggest win is simply using a smaller/cheaper model where it's sufficient.
- **Trim what you send** — shorter prompts and summarized history reduce cost on every single call.
- **Cache aggressively** for repeated or predictable queries; it's often the highest-leverage optimization available.
- **Cap output length** with `max_tokens` (or equivalent) to prevent runaway generations.
- **Attribute costs** to teams/features so the people making usage decisions can see the financial impact.
- **Set budgets and alerts** so cost problems are caught early, not after a surprising bill.
- **Balance cost against quality** — the goal is efficient spending that matches business value, not simply spending the least possible.
