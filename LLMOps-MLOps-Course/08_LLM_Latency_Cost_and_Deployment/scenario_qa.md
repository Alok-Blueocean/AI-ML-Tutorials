# LLM Latency, Cost, and Deployment — Scenario-Based Q&A

**Situation:** A team proudly reports that their assistant's average response latency is 400ms, well within target, but the on-call channel keeps getting paged for "the app feels slow" complaints. What would you do and why?

Model answer: Point out that optimizing and reporting on average latency hides exactly where production pain lives — a system with a great mean and a terrible tail still produces angry users and pages, because the users who hit the tail are the ones who complain. Start tracking and setting SLOs on P95 and P99 latency instead of (or in addition to) the mean, and investigate what's driving the tail specifically — often a small subset of long-context requests, cold cache misses, or contention during traffic bursts — rather than chasing further improvement in a mean that's already acceptable.

---

**Situation:** A finance stakeholder flags that the LLM API bill has tripled month-over-month even though request volume only grew 20%, and asks the team to explain it before approving next quarter's budget. What would you do and why?

Model answer: Investigate the two likeliest silent-growth culprits the module calls out: whether the flagship/largest model has become the default for every request rather than an escalation path, and whether context has grown silently — unpruned chat history, RAG retrieval returning more chunks "just to be safe," or a system prompt that's accumulated one-off instructions over months. Pull a cost breakdown by model tier and by average input-token count per request over time to confirm which factor (or both) explains the 3x-on-20%-volume gap, then fix the specific driver — route by task complexity instead of defaulting to the flagship model, and add explicit token budgeting/truncation to stop unbounded context growth — rather than applying a blanket "reduce token limits" fix that could break legitimate requests.

---

**Situation:** A team builds a semantic cache to cut LLM costs and, a week after launch, a customer from Tenant A receives an answer that was clearly generated for Tenant B's data. What would you do and why?

Model answer: Diagnose this immediately as a cache-key scoping failure — the module's explicit warning is that a semantic cache built without metadata scoping (tenant, model version, locale) in the cache key risks exactly this kind of cross-tenant leakage. Take the cache offline or flush it immediately given the data-exposure severity, then rebuild the cache key to include tenant ID (and ideally model version and locale) as mandatory scoping fields, not just the semantic similarity of the query text, and add an automated test that specifically verifies no cache hit can cross a tenant boundary before re-enabling the cache in production.

---

**Situation:** A semantic cache's hit rate looks great on the dashboard and the team celebrates a big latency win, but a few weeks later support tickets increase for "the bot gave a weird, slightly off answer." What would you do and why?

Model answer: Suspect the similarity threshold was picked by eyeballing a few examples rather than validated against a labeled set of true-positive and false-positive matches — a threshold that's a little too loose serves semantically-similar-but-not-actually-equivalent cached answers, which looks like a pure performance win on a hit-rate dashboard while silently degrading answer quality. Build a labeled set of query pairs (should-match vs. should-not-match) and tune the threshold against precision on that set rather than hit-rate alone, and consider logging cache hits with their similarity score for a sampling review process so quality regressions from over-eager caching are caught before they reach a support-ticket volume.

---

**Situation:** A streaming chat feature works perfectly when developers test it locally, but in production, users report that responses appear to "hang" for several seconds and then dump the entire answer at once, defeating the point of streaming. What would you do and why?

Model answer: Recognize this as the classic reverse-proxy buffering trap — code that streams correctly against the app server directly can still get buffered by an nginx/ingress layer in production that buffers responses by default, so testing only on localhost misses exactly the failure that matters. Test streaming behavior through the actual production network path (through the real ingress/proxy chain), not just directly against the app server, and explicitly configure the proxy layer to disable response buffering for the streaming endpoint (e.g., `proxy_buffering off` in nginx, or the equivalent for whatever ingress controller is in use) as part of the deployment checklist for any new streaming feature.

---

**Situation:** A team monitors LLM spend via a monthly cost dashboard, and one month discovers a single misbehaving feature burned through 40% of the entire monthly budget in the first three days, with no one noticing until the invoice arrived. What would you do and why?

Model answer: Identify the core issue as treating cost control as a reporting exercise rather than a request-time decision — a monthly dashboard tells you about a runaway cost pattern weeks after it started, by which point the damage is done. Implement inline, request-time budget guardrails (a `guarded_request`-style check that can reject, downgrade to a cheaper model, or rate-limit before an expensive call is made, not just log it afterward) for any feature with meaningful per-request cost variance, and add real-time or near-real-time cost alerting (hourly or daily thresholds, not monthly) so an anomalous spend pattern triggers an alert within hours, not at month-end.

---

**Situation:** After months of steady API costs, a team decides to self-host an open-weight model to "save money," and six months later concludes it actually cost more than the API ever did once salaries and incident time are counted. What would you do and why?

Model answer: Frame this as exactly the trap the module warns about: choosing local deployment based on "it should be cheaper eventually" without an honest assessment of the team's GPU/SRE operational maturity trades a predictable API bill for unpredictable incident load, and that trade is often worse even when the raw unit economics favor self-hosting on paper. Before recommending self-hosting again, do an honest capability and total-cost-of-ownership assessment — including on-call burden, GPU procurement/utilization risk, and engineering time spent on serving infrastructure — not just a per-token cost comparison, and consider that a partial approach (self-host only the highest-volume, most cost-sensitive workload; keep the API for everything else) may capture most of the savings with a fraction of the operational risk.

---

**Situation:** A team wants to add a "budget guardrail" to their product and debates whether to enforce it by checking cumulative spend at the end of each day and disabling the feature if it's over budget, or by checking before each individual request. What would you do and why?

Model answer: Recommend request-time (before-call) enforcement over end-of-day batch checking, consistent with the module's guidance that cost control needs to be inline, request-time logic rather than a reporting exercise — an end-of-day check means the budget can already be blown by the time the day's check runs, and the feature gets disabled for all remaining users rather than gracefully degrading the specific requests that would have pushed spend over budget. Implement a `guarded_request`-style check that evaluates cumulative spend against budget before each call and, when near the limit, downgrades to a cheaper model or applies stricter rate limiting rather than an all-or-nothing daily kill switch, preserving service for most users while still protecting the budget.

---

**Situation:** A team migrates from a chat feature backed by the flagship model to a cheaper, smaller model for cost reasons, and immediately gets complaints that responses feel "less smart," even on questions the smaller model should easily handle. What would you do and why?

Model answer: Before assuming the smaller model is simply less capable at the task, check for context-length and prompt-caching regressions introduced by the migration — a different model generation can have a different context window, different tokenizer, and different prompt-caching prefix behavior, and if the request pipeline wasn't re-tuned for the new model, truncation or budget miscalculation could be silently degrading responses independent of the model's actual raw capability. Isolate the variable by running the exact same prompt through both models on a fixed eval set and comparing quality directly; if the smaller model is genuinely worse on a meaningful subset of query types, consider a tiered routing approach (escalate only the harder queries to the flagship model) rather than an all-or-nothing model swap.

---

**Situation:** A latency-sensitive support-chat feature caps every response at the same `max_tokens` value used for a completely different long-form report-generation feature, and users of the chat feature start seeing occasional abrupt cutoffs mid-sentence. What would you do and why?

Model answer: Identify a uniform output-length cap across very different task types as the root cause — a cap sized for long-form reports will occasionally be too small relative to the chat feature's own worst-case realistic response length if the two features share one global setting inconsistently, or conversely a cap tuned for chat may unnecessarily constrain the report feature. Set `max_tokens` per task type based on each feature's own realistic output-length distribution, and add monitoring for a truncation-rate metric per feature (responses that hit the cap) so a too-tight cap surfaces as a measurable signal rather than sporadic user complaints.

---

**Situation:** An engineer proposes prompt-caching every request to cut costs, but after implementation, the team sees no meaningful cache-hit rate improvement despite the prompts looking "mostly the same" across requests. What would you do and why?

Model answer: Check prompt-caching prefix ordering first — provider-side prompt/context caching works off a matching prefix, so if static content (system instructions, few-shot examples) and dynamic content (the user's variable input) are interleaved, or if variable content comes before the static block, the cache silently never matches regardless of how similar the prompts "look" to a human skimming them. Restructure prompts so all static content forms one unbroken, stable prefix with variable content appended at the end, and verify the fix using the provider's cache-hit telemetry rather than eyeballing prompt similarity, since a prompt that looks mostly the same to a human can still have a broken prefix from the caching mechanism's perspective.

---

**Situation:** A team handling a sudden 10x traffic spike from a marketing campaign notices GPU utilization and per-request latency both climb sharply, and someone suggests just adding more replicas as the fix. What would you do and why?

Model answer: Before reaching for horizontal scaling alone, check whether batching is being used effectively for the workload — grouping compatible concurrent requests into batched inference calls significantly improves throughput per GPU for many serving stacks, and a serving setup that processes requests one-at-a-time will scale far less efficiently under a spike than one with continuous/dynamic batching enabled. Confirm the inference server (vLLM, TGI, Triton, etc.) has batching correctly configured and tuned for the traffic pattern first, since fixing that can absorb a meaningful fraction of a spike more cost-effectively than proportionally scaling replica count, then add autoscaling on top for whatever headroom batching alone doesn't cover.

---

**Situation:** A CFO asks the ML platform lead to justify why the team is paying for both an LLM API subscription and a self-hosted GPU cluster, arguing the company should "just pick one" for simplicity. What would you do and why?

Model answer: Push back on "pick one" as a false simplification — API vs. local deployment is a per-workload decision, not an org-wide religion, and it's entirely normal and often correct for a mature platform to run both simultaneously for different workload shapes (e.g., the API for spiky/experimental/low-volume features where operational simplicity wins, self-hosted infrastructure for the highest-volume, most cost-sensitive, steady-traffic workload where unit economics and control win). Present the decision per-workload against the module's criteria (traffic pattern, latency sensitivity, team's operational maturity, cost at the specific volume involved) rather than a single company-wide infrastructure choice, and quantify what each option is actually buying for its specific workload.

---

**Situation:** A team's RAG assistant's cost-per-query has crept up steadily over six months with no single obvious cause, and a cost audit is requested before the next budget cycle. What would you do and why?

Model answer: Systematically check the module's known silent-growth vectors rather than guessing: whether chat history truncation/summarization is still working as originally designed or has quietly stopped firing, whether RAG retrieval's top-k or reranking cutoff has crept upward "to be safe," whether the system prompt has accumulated one-off instructions added by different engineers over six months without anyone auditing its total size, and whether the model-tier routing logic still routes most traffic to a cheaper model or has drifted toward defaulting to the flagship model. Present the audit as a token-cost breakdown over time by these specific categories so the actual driver (likely more than one) is visible and fixable, rather than an undifferentiated "costs went up" finding.

---

**Situation:** A team debating self-hosting an open-weight model for a new product argues the raw per-token cost is clearly cheaper than the API, and wants to proceed based on that number alone. What would you do and why?

Model answer: Push back that the per-token comparison alone is an incomplete basis for the decision — the module's central warning on this tradeoff is that self-hosting trades a predictable bill for unpredictable incident load, and the real comparison needs to include the team's GPU/SRE operational capability, the incident and on-call cost of running inference infrastructure, and the engineering time diverted from product work to serving infrastructure. Ask for an honest assessment of the team's current operational maturity for running GPU inference reliably before greenlighting the migration on unit economics alone, and consider a staged approach — self-host the single highest-volume workload first as a bounded pilot, rather than migrating everything at once, to validate the true all-in cost before committing further.

---

**Situation:** A team wants a single dashboard number that tells them whether their LLM deployment is "healthy," and proposes tracking only overall requests-per-second. What would you do and why?

Model answer: Recommend against a single-metric health view and instead track the four numbers the module frames as jointly defining "good": latency (specifically tail latency, P95/P99, not just mean or throughput), cost per request/session, and whatever quality/correctness signal is relevant to the product — RPS alone tells you nothing about whether users are having a good or bad experience, or whether cost is under control. Build a small dashboard surfacing P95/P99 latency, cost-per-request trend, and a quality proxy metric side by side, since a system can look "healthy" on throughput alone while quietly failing on any of the other three dimensions.

---

**Situation:** A newly launched feature streams tokens to the frontend, but users report the perceived experience feels no faster than the old non-streaming version, even though time-to-first-token is measurably low in backend logs. What would you do and why?

Model answer: Investigate whether the frontend is actually rendering tokens incrementally as they arrive, or buffering the full stream client-side before displaying anything — a low time-to-first-token on the backend is wasted if the UI doesn't render progressively, since the user experiences the same "wait then see it all" pattern regardless of how fast the backend technically started streaming. Confirm end-to-end streaming behavior by observing the actual browser rendering timeline (not just backend logs), and check the same reverse-proxy-buffering failure mode that can silently defeat streaming at the infrastructure layer even when the application code streams correctly — the perceived-latency win from streaming requires every layer (server, proxy, frontend rendering) to actually stream, not just one.
