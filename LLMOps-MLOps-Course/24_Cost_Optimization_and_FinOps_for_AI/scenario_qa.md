# Cost Optimization and FinOps for AI — Scenario-Based Q&A

**Situation:** Your team's LLM API bill tripled month-over-month with no corresponding increase in user traffic, and finance wants an explanation by end of day. What would you do and why?

Model answer: Start with per-request cost, not total spend, since total spend conflates volume and unit cost — pull a breakdown of average input/output tokens per request over time to see if unit cost rose even though traffic didn't. Common causes to check first: a recent prompt change that added context (a new few-shot example set, a bloated system prompt, uncapped conversation history resent every turn), a model upgrade to a pricier tier, or a bug causing retries/duplicate calls. Once identified, quantify the fix in dollar terms ("trimming the resent history saves an estimated $X/month at current volume") so the explanation to finance is concrete and the fix is verifiable after deployment, not just asserted.

---

**Situation:** A stakeholder insists the team should always use the most capable (and most expensive) model for every feature "to guarantee the best quality," regardless of task complexity. What would you do and why?

Model answer: Reframe the argument around measured quality-per-dollar rather than accepting "best model = best outcome" as self-evident — for simple tasks like intent classification or short extraction, a small model often matches a large model's accuracy at a fraction of the cost, and the "best quality" framing ignores that budget saved on easy tasks can fund more evaluation, more capacity for hard tasks, or simply better margins. Propose a model-routing or cascade approach: benchmark a smaller/cheaper model against the golden eval set for each task category, and only route to the expensive model where the accuracy gap is real and matters for that specific use case. Bring data from that benchmark to the conversation rather than debating the principle in the abstract.

---

**Situation:** Your multi-turn chat application resends the entire conversation history with every new user message, and a long-running support conversation now costs 40x more per message than the first turn. What would you do and why?

Model answer: Identify this as the classic "conversation history growth" cost trap and address it directly: instead of resending full raw history indefinitely, summarize older turns into a compact running summary once the conversation exceeds a length threshold, keeping only the summary plus the most recent few turns verbatim in the prompt. Validate that this doesn't silently degrade answer quality — run the summarization approach against a set of long conversations in your eval suite and check whether the model still correctly references earlier context. Communicate the tradeoff explicitly: this trades a small amount of context fidelity for a large, predictable reduction in per-message cost on long conversations, which is usually the right trade for typical support use cases.

---

**Situation:** Your team wants to add semantic caching (reusing responses for near-identical queries) to cut costs, but a colleague worries this will serve stale or slightly wrong answers to users who phrase things differently but expect a fresh answer. What would you do and why?

Model answer: Validate the concern is real but scope it precisely rather than rejecting caching outright — semantic caching is safe and high-leverage for queries where the answer is genuinely stable (FAQ-style, reference lookups), and risky for anything time-sensitive or personalized (account balances, "what's my order status"). Implement caching selectively, scoped to query categories known to be cache-safe, with a similarity threshold tuned conservatively (favoring cache misses over incorrect hits), and add a TTL so cached answers expire rather than persisting indefinitely. Measure the cache hit rate and the cost savings it produces against a small sample of manually reviewed cache hits to confirm answer quality isn't degrading before rolling it out broadly.

---

**Situation:** A self-hosted model serving GPUs are running at 15% average utilization, but the team is hesitant to reduce GPU count because "we need headroom for traffic spikes." What would you do and why?

Model answer: Separate "headroom for spikes" from "always-on idle capacity," since 15% average utilization with occasional spikes is exactly the pattern autoscaling solves, not a reason to keep fixed overprovisioned capacity running constantly. Implement autoscaling that scales GPU capacity down during low-traffic periods and up during spikes, potentially combined with batching requests to improve utilization per active GPU, and consider spot/discounted instances for the elastic portion of capacity if brief interruptions are tolerable for your use case. Quantify the current waste (idle GPU-hours × hourly cost) to make the case concretely, and set a target utilization band (e.g., 50-70% average) as the metric to track after the change, rather than leaving "headroom" as a vague, unmeasured justification for overprovisioning.

---

**Situation:** After switching to a cheaper model for a classification task to cut costs, a downstream team reports accuracy dropped enough to cause visible user complaints, and now leadership wants to know if the cost savings were worth it. What would you do and why?

Model answer: Present the actual tradeoff in comparable units rather than defending the decision on cost alone — quantify the dollar savings from the cheaper model against the business cost of the accuracy drop (support tickets, user churn, manual correction time), since "cheaper" and "worth it" are different questions. If the accuracy regression wasn't validated against the golden evaluation set before the switch, that's the real process gap: any model change, even one framed as a "safe" cost optimization, needs to clear the same evaluation gate as a quality-driven change before shipping. Decide whether to roll back, tune the routing so only genuinely easy cases go to the cheaper model, or accept the tradeoff explicitly with stakeholder sign-off — but make that a deliberate, measured decision, not a byproduct of a cost cut.

---

**Situation:** Finance asks for a report on "cost per AI feature" but your team only has total monthly spend across all LLM usage, with no way to attribute it to individual products or teams. What would you do and why?

Model answer: Explain that this is a tagging/attribution gap, not a reporting problem — without per-request tagging of which team, product, or feature initiated a given LLM call, spend can never be broken down after the fact, only estimated. Implement request-level tagging (e.g., a `feature` or `team` field passed with every API call, or separate API keys/projects per feature if the provider supports it) going forward, and be upfront with finance that historical spend can only be roughly estimated retroactively, not precisely reconstructed. Frame the fix as a small upfront engineering cost that pays for itself the first time someone needs exactly this kind of report again.

---

**Situation:** An engineer proposes quantizing a self-hosted model from 16-bit to 8-bit precision to cut GPU memory needs in half, but another engineer worries this will silently hurt output quality. What would you do and why?

Model answer: Don't treat this as a binary "safe" or "unsafe" choice — quantization typically causes a small, measurable quality degradation, and whether that's acceptable depends on the task and how much margin the model has above your quality bar. Run the quantized model against the same golden evaluation set used for other model changes, and compare accuracy/faithfulness metrics directly against the full-precision baseline rather than relying on informal spot-checks. If the degradation is within acceptable bounds for the use case, ship it and capture the GPU cost savings; if it's not, the data makes that clear rather than the decision resting on either engineer's intuition.

---

**Situation:** A team member wants to set `max_tokens` very high "just in case the model needs to generate a long response," and you notice several endpoints have no output length cap at all. What would you do and why?

Model answer: Point out the concrete risk: with no `max_tokens` cap, a single malformed prompt, a model quirk, or an edge-case input can produce a runaway generation that costs far more than intended and adds unpredictable latency, and this is one of the cheapest, easiest-to-apply cost controls available. Set a deliberate cap based on the actual expected response length for each endpoint's task (a classification endpoint needs a tiny cap; a long-form generation endpoint needs a larger one, but still bounded), rather than leaving it unset "just in case." Audit all production endpoints for this gap in one pass, since it's a five-minute fix per endpoint with real downside protection and essentially no cost to legitimate use cases.

---

**Situation:** Your company is deciding whether to fine-tune a smaller open-source model for a narrow internal task versus continuing to use a large hosted API, and leadership asks for a cost justification. What would you do and why?

Model answer: Frame this explicitly as training cost (one-time or periodic) versus inference cost (ongoing, usage-driven) rather than comparing sticker prices — fine-tuning has real upfront GPU cost and engineering time, but if the task has high, sustained call volume, the ongoing inference savings of a smaller self-hosted model can pay back the training investment within a defined period. Build the model with projected monthly volume: estimate current hosted-API cost at that volume versus self-hosted GPU cost (including utilization assumptions) plus amortized fine-tuning cost, and identify the breakeven point. If volume is low or unpredictable, the hosted API's pay-per-use model is usually more cost-effective despite the higher per-call price — make the recommendation volume-dependent rather than a blanket answer.

---

**Situation:** After implementing a cost dashboard, you notice one specific internal tool accounts for 60% of total LLM spend, used by a small team for what seems like a minor workflow. What would you do and why?

Model answer: Investigate before assuming waste — check whether the 60% reflects genuine high business value (e.g., processing a large volume of documents that saves significant manual labor) or an inefficiency (a verbose prompt, an unnecessary retry loop, calling a large model for a task a small one could handle). Bring the cost-per-outcome metric into the conversation with that team specifically — cost per document processed, or per task completed — rather than just total spend, since a high absolute number can still be efficient at scale. If it turns out to be inefficient, prioritize optimizing it first precisely because it's the largest line item — the same percentage improvement there dwarfs equivalent effort spent optimizing smaller consumers.

---

**Situation:** A product manager wants to launch a new AI feature but has no idea what it will cost to run at scale, and asks you to "just estimate it" before committing to a budget. What would you do and why?

Model answer: Build the estimate from the same token-economics formula used to monitor existing features: estimate average input tokens (prompt + context/RAG retrieval + conversation history) and average output tokens per request, multiply by the provider's per-token pricing, then multiply by projected request volume at expected launch scale. Explicitly flag the biggest uncertainty — usage volume is often the hardest variable to predict pre-launch — and present the estimate as a range (conservative and optimistic volume scenarios) rather than a single number, so the PM can set a budget with awareness of the uncertainty rather than false precision. Recommend instrumenting cost tracking from day one of launch so the estimate can be validated and corrected quickly against real usage.

---

**Situation:** Your team implements model routing (cheap model first, escalate to expensive model when needed) but a few weeks later notices the "cheap model" tier is handling requests it clearly shouldn't, producing poor answers that then don't get escalated. What would you do and why?

Model answer: Diagnose the escalation logic specifically — routing only saves money and preserves quality if the trigger for escalating to the expensive model is reliable; if it's based on a weak heuristic (e.g., prompt length) rather than an actual confidence or quality signal, hard cases will silently stay on the cheap model. Improve the escalation trigger, for example by having the cheap model emit a confidence score or by using a lightweight classifier to route by task difficulty before the first call, and validate the routing decision itself against the golden evaluation set, checking specifically for cases that should have escalated but didn't. Treat the router as a component that needs its own evaluation, not just a cost-saving mechanism assumed to be quality-neutral once deployed.

---

**Situation:** Six months into running a chatbot feature, someone asks whether the ongoing LLM cost is still justified by the business value it produces, and nobody has looked at this since launch. What would you do and why?

Model answer: This is exactly the gap regular cost reviews exist to close — pull the cost-per-outcome metric (e.g., cost per resolved conversation, or per successfully answered query) and put it next to a business-value proxy (deflected support tickets, user satisfaction scores, time saved) to answer the value-for-money question directly rather than debating total spend in isolation. If the metric shows declining marginal value (e.g., usage has plateaued or shifted to lower-value queries while cost hasn't), that's actionable — revisit model choice, prompt efficiency, or caching opportunities that may not have been worth the effort at launch but are now. Recommend scheduling this review as a recurring practice (quarterly, tied to the FinOps cadence) rather than something that only happens when someone happens to ask.

---

**Situation:** Your team is under pressure to cut AI costs by 30% this quarter, and someone proposes simply reducing `max_tokens` across all endpoints by 30% as the fastest way to hit the target. What would you do and why?

Model answer: Resist the purely mechanical approach — an across-the-board output cap cut will hit cost targets on paper but risks truncating legitimate long-form responses on some endpoints while barely touching short-response endpoints where `max_tokens` was never the binding cost driver. Instead, break down current spend by lever (model tier, prompt/context size, output length, call volume, caching opportunity) and target the levers actually driving cost on each specific endpoint, which likely means different fixes per endpoint — right-sizing the model here, trimming context there, capping output length only where it was genuinely oversized. This takes more analysis than a blanket cut but avoids trading a cost win for a quality regression that generates its own costs in complaints and rework.

---

**Situation:** A new engineer asks why the team tracks GPU utilization as a cost metric when "we're not even close to running out of capacity." What would you do and why?

Model answer: Explain that utilization is a cost-efficiency metric, not a capacity-warning metric — low utilization means you're paying for GPU-hours that aren't doing useful work, regardless of whether you're anywhere near a capacity ceiling, similar to paying for an office nobody sits in because there happens to be room. Point to the earlier lesson that "utilization matters more than raw GPU count" — the fix for a cost problem caused by low utilization isn't necessarily fewer GPUs, it's better packing of work onto the GPUs you have (batching, autoscaling to match demand) so the same capacity does more useful work per dollar. Tie it back to the team's actual FinOps goal: the objective isn't minimizing GPU count, it's maximizing useful work per dollar spent.
