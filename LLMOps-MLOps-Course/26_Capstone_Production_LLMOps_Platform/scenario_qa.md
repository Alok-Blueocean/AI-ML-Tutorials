# Capstone: Production LLMOps Platform — Scenario-Based Q&A

**Situation:** Your customer-support RAG assistant has been live for a month. Support tickets about "wrong answers" are rising, but your dashboards show latency, error rate, and cost are all normal. Where do you look, and why?

Model answer: Recognize that infrastructure health (latency, errors, cost) and answer quality are separate layers of the platform, and a green infrastructure dashboard says nothing about the evaluation or data layers. Pull the observability traces for a sample of the complained-about interactions and check the retrieval step specifically — most "wrong answer" complaints in a RAG system trace back to retrieval (stale or missing documents in the knowledge base, a recent corpus edit that wasn't re-indexed) rather than the model itself. Cross-reference against the evaluation layer: if the golden eval set hasn't been updated to reflect the kinds of questions now coming in, your automated quality gate may be blind to exactly this failure mode, and expanding it should be part of the fix, not just patching the immediate retrieval bug.

---

**Situation:** You're three weeks from a planned launch, and a leadership review asks you to justify why the platform needs a separate evaluation layer and an observability layer — "isn't that the same thing, just monitoring?" What would you do and why?

Model answer: Distinguish them by the question each answers: evaluation answers "did this change make the system better or worse?" — a before/after comparison run against a fixed golden dataset, typically pre-deployment or in CI. Observability answers "is the system healthy right now, for real production traffic?" — live latency, cost, error rates, and traced requests, with no fixed dataset because real traffic is the input. Conflating them creates blind spots in both directions: relying only on evaluation misses live production drift and infrastructure failures that a static test set never encounters; relying only on observability misses whether a proposed change actually improves quality before it ships. Recommend keeping both, and explicitly wiring evaluation results into the CI/CD quality gate while observability feeds live dashboards and alerting.

---

**Situation:** Two weeks after launch, your platform's monthly LLM API cost is triple the pre-launch estimate, and finance wants to know whether the platform is architected inefficiently or whether this is expected growth. What would you do and why?

Model answer: Use the observability layer's cost-per-request tracking (if it's wired up — if not, that's the first gap to close) to separate volume growth from unit-cost inefficiency: check whether cost-per-request matches the pre-launch estimate and total requests simply exceeded projections, versus cost-per-request itself climbing, which would point to prompt bloat, unbounded conversation history, or retrieval returning more context than needed. If it's volume growth beyond projection, that's a demand-forecasting miss, not an architecture problem, and the fix is revisiting the FinOps budget and possibly adding cost controls (caching, model routing) proactively rather than retroactively. If it's unit-cost creep, trace it to a specific layer (retrieval returning too many chunks, an uncapped `max_tokens`) and fix it directly.

---

**Situation:** During a demo to a potential enterprise customer, someone asks "what happens if the vector database goes down — does the whole assistant stop working?" and you realize you've never actually tested that failure mode. What would you do and why?

Model answer: Treat this as a real gap in the orchestration layer's resilience design, not just an awkward question to deflect in the moment — a production platform needs an explicit fallback path for every critical dependency, and "the assistant just errors out" is rarely acceptable for a customer-facing feature. Design and test a graceful degradation path: if retrieval is unavailable, the system could fall back to a general-knowledge response with a clear disclaimer that it couldn't access the knowledge base, or escalate to a human, rather than failing silently or throwing a raw error. After the demo, add a chaos-style test (deliberately taking the vector DB offline in staging) to the evaluation or testing process so this failure mode is verified, not assumed.

---

**Situation:** Your platform's safety layer includes a guardrail that blocks obvious jailbreak attempts, but a security researcher reports they were able to extract part of your system prompt using a multi-turn technique that no single message would have triggered the guardrail on. What would you do and why?

Model answer: Acknowledge this as a real finding rather than dismissing it because "the guardrail worked as designed" per-message — multi-turn jailbreaks are a known limitation of guardrails that evaluate each message in isolation, and the fix needs to consider conversation-level context, not just single-turn pattern matching. Add output-side protection specifically for system-prompt leakage (checking whether the response contains substantial verbatim overlap with the system prompt) as a second layer independent of input screening, and log this technique into your safety regression test set so future guardrail or prompt changes are checked against it. Thank the researcher's report as exactly the kind of adversarial testing your evaluation layer should be doing proactively, and consider whether a periodic red-teaming exercise should be a standing part of the platform's process.

---

**Situation:** Your platform is currently a single monolithic service handling retrieval, generation, and tool orchestration in one process, and it's become difficult to deploy a change to one part (say, the retrieval logic) without redeploying and risking the whole system. What would you do and why?

Model answer: Recognize this as the orchestration and serving layers becoming entangled, and propose separating concerns along the same lines the platform's own architecture describes — retrieval, model/prompt handling, and orchestration logic can be decoupled into services (or at least clearly separated modules) with their own deploy cycles, so a retrieval fix doesn't require redeploying generation logic and risking an unrelated regression. Weigh this against added operational complexity (more services to monitor, network calls where there were function calls) — for a small team, a modular monolith with clean internal boundaries may be the right stopping point rather than full microservices, but the goal either way is that a change to one layer doesn't force redeploying and re-risking the others.

---

**Situation:** A regulator or enterprise customer asks for a complete audit trail proving what data trained the current production model, what prompt version generated a specific flagged response, and who approved the deployment — and pulling this together currently takes your team two days of manual archaeology across Slack, email, and code history. What would you do and why?

Model answer: This is the direct cost of not having version control and audit logging wired end-to-end as the capstone architecture describes — data versioning (Module 22), prompt versioning, model registry entries, and deployment approvals should all be linked so this exact question is a lookup, not an investigation. Prioritize closing the weakest link first: identify which of the four (data → model → prompt → deployment approval) currently has no automated trace and is being reconstructed from memory or chat logs, since that's almost certainly where the two days are going. Frame the fix to leadership as risk reduction, not bureaucracy — the two-day reconstruction cost will recur on every future audit or incident until the chain is automated.

---

**Situation:** Your evaluation gate blocks deployment if faithfulness drops below 0.85, and a new model version scores 0.84 — just barely below threshold — but the team believes it's meaningfully better on other dimensions (helpfulness, conciseness) that aren't in the gate. What would you do and why?

Model answer: Don't override the gate manually to force a deploy — a threshold that gets overridden whenever it's inconvenient stops being a real gate. Instead, treat the near-miss as evidence the evaluation metric set might be incomplete: if helpfulness and conciseness are genuinely important and improving, add them as tracked metrics (even if not blocking) so the tradeoff is visible and can inform a deliberate decision about whether the threshold or the metric weighting needs revisiting. If the team still wants to ship this specific version, that should go through an explicit, documented exception process with a named approver — not a quiet bypass — precisely because "we usually don't deploy below 0.85 but made an exception because X" is exactly the kind of decision that needs a paper trail for the next postmortem or audit.

---

**Situation:** Six months post-launch, a postmortem reveals that a slow but steady accuracy decline went unnoticed for weeks because nobody was regularly reviewing the observability dashboards — the alerts were only configured for hard failures (5xx errors, timeouts), not for gradual quality drift. What would you do and why?

Model answer: Add drift detection as a first-class monitoring signal, not just infrastructure health checks — this is precisely the distinction the platform's architecture draws between "is it healthy right now" (uptime, latency, errors) and quality drift, which requires comparing current production outputs (or a sampled, labeled subset of them) against a baseline over time, similar to the approach in the drift-detection module. Configure alerting on gradual metric decline (a moving average crossing a threshold), not just hard failures, since hard failures are the easy case and gradual drift is exactly the failure mode that silently erodes trust the longest before anyone notices. Also close the process gap: assign explicit ownership for regularly reviewing quality dashboards, since a dashboard nobody looks at provides no more protection than no dashboard at all.

---

**Situation:** Your platform's feedback loop is designed to feed flagged bad responses back into future evaluation sets, but you discover the "flag as incorrect" button in the UI has been silently broken for two months, and no flagged examples have been collected. What would you do and why?

Model answer: Treat this as a broken feedback loop undermining the platform's core "ops" premise — the architecture explicitly names the feedback loop as what separates a one-off demo from a real production practice, and two months of silently lost signal means two months of real-world failure cases that should have improved the eval set never did. Fix the button, and add a basic health check that periodically verifies the feedback pathway actually works end-to-end (a synthetic test flag that confirms it's recorded), so a break in this channel is caught in hours, not months. Separately, assess whether other feedback signals (support escalations, user complaints logged elsewhere) can partially backfill the two-month gap, since some of that lost signal may be recoverable from adjacent systems.

---

**Situation:** A junior engineer joining the capstone project asks why the architecture insists on version-controlling prompts the same way as code, when "a prompt is just a string, not really software." What would you do and why?

Model answer: Push back on the framing directly — a prompt change can alter model behavior as significantly as a code change (a single wording tweak can shift accuracy, tone, or safety behavior meaningfully), so treating it with less rigor than code is a mismatch between its actual impact and how carefully it's managed. Version-controlling prompts (file-based, in Git, referenced by version like `support_v3.txt` in the capstone example) gives you the same benefits code versioning gives: you can see exactly what changed, roll back a regression instantly, and tie a specific prompt version to the evaluation results and production incidents associated with it. Make the case concrete: "which exact prompt was live when this bad response happened" should be a one-line lookup, the same as "which code commit was live," not a guess based on who remembers editing it last.

---

**Situation:** Your platform handles both a low-stakes internal FAQ bot and a higher-stakes customer-facing refund-approval agent, but currently both run through identical guardrails, evaluation thresholds, and human-review policies. A stakeholder asks if this uniform treatment makes sense. What would you do and why?

Model answer: No — argue for risk-tiered policies rather than one-size-fits-all, since uniform treatment either over-constrains the low-stakes bot (slowing it down and adding unnecessary friction for FAQ answers) or under-constrains the high-stakes agent (not enough human oversight for consequential, hard-to-reverse actions like refunds). Define explicit risk tiers across the platform — the FAQ bot can run with lighter guardrails and no mandatory human review; the refund agent should have a hard-coded action limit, mandatory human confirmation above a threshold, and a stricter evaluation gate before any prompt or model change ships. Document the tiering criteria so future features get classified consistently rather than each new feature's oversight level being decided ad hoc.

---

**Situation:** Your team is asked in an interview-style capstone review to defend why you chose Chroma as the vector store, Redis for caching, and FastAPI for serving, and the interviewer pushes: "couldn't you have just used one all-in-one platform instead of stitching these together?" What would you do and why?

Model answer: Answer with the actual tradeoffs considered rather than defending the specific tools as objectively best — an all-in-one managed platform reduces integration work and operational surface area at the cost of flexibility and potential vendor lock-in, while a composed stack of best-fit tools (a lightweight vector store, a fast cache, a well-understood serving framework) gives more control and swappability at the cost of more integration and operational work to own. Explain the specific reasoning for this project's scale and constraints (e.g., "Chroma was sufficient for our corpus size and avoided managed-service cost; Redis was already the team's caching default") — the interviewer is testing whether you can articulate tradeoffs and reasoning, not whether you picked the "correct" tools, which is exactly the point the module's key takeaways make about documenting tradeoffs, not just architecture.

---

**Situation:** After a successful capstone demo, your instructor/interviewer asks: "If this had to support 100x the current traffic tomorrow, what's the first thing that breaks?" What would you do and why?

Model answer: Give a specific, reasoned answer rather than a vague "we'd scale up" — walk through each layer and identify the actual bottleneck: a single-instance FastAPI server with no autoscaling would be the first to fall over under 100x load, followed by the vector store if it wasn't built for concurrent high-QPS queries, followed by API rate limits on the underlying LLM provider itself, which no amount of your own infrastructure scaling can bypass. Propose the fix in priority order: autoscaling and load balancing for the serving layer first (cheapest, fastest to add), then evaluate whether the vector store needs to move to a managed, horizontally-scalable option, then negotiate higher rate limits or add request queuing/backpressure for the LLM provider dependency, since that's the hardest constraint to remove entirely.

---

**Situation:** A capstone reviewer asks you to walk through what happens end-to-end when a user asks a question that the RAG system cannot answer confidently from the knowledge base — tracing the request through every layer of your architecture. What would you do and why?

Model answer: Narrate the full path deliberately, since this question is testing whether the layers actually connect, not just exist independently: the query enters through the serving layer (API, rate-limited), the orchestration layer triggers retrieval against the data layer's vector store, low similarity scores or the fallback logic (configured as `fallback: "escalate_to_human"` in the capstone example) detect low confidence, the model/prompt layer either generates a hedged "I don't have enough information" response or the orchestration layer routes to human escalation instead of forcing a possibly-hallucinated answer, the safety layer confirms no guardrail violations either way, and the observability layer logs the full trace including the low-confidence signal for later review. Emphasize that the low-confidence fallback path is not an edge case to handle later — it should be designed and tested with the same rigor as the happy path, since it's exactly the moment hallucination risk is highest.

---

**Situation:** Your capstone platform is otherwise complete, but a reviewer notes that every layer you've built (data, model, orchestration, serving, evaluation, observability, safety) has no defined owner if something breaks at 2 AM — there's no on-call process at all. What would you do and why?

Model answer: Accept this as a legitimate gap even for a capstone-scale project, since "who's responsible when this breaks" is part of operating a system responsibly, not an afterthought reserved for larger teams — the observability layer's dashboards and alerts are only useful if someone is actually going to see and act on them. Propose a minimal but real on-call structure even for a small team or solo capstone: define what conditions page someone (using the alerting thresholds already built), where alerts route (even if it's just a personal phone notification for a solo project), and a basic escalation/rollback runbook for the most likely failure modes (bad deployment, cost spike, safety guardrail failure). Frame this as proof of end-to-end operational thinking, which is exactly what separates the "ops" in LLMOps from having simply built a working demo.
