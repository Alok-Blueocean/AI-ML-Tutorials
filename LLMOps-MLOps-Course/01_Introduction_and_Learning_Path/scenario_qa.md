# Module 01 — Scenario-Based Q&A

**Situation:** A new engineering director joins and, on day one, asks your team to "just apply our existing DevOps CI/CD pipeline to the new fraud-scoring model, since it's basically just another service." What would you do and why?

Model answer: Push back constructively with the specific gap, not just "that's wrong." Explain that DevOps CI/CD assumes behavior is a pure function of code, so passing tests means the service is correct; a fraud model's behavior is also a function of training data and the model artifact, neither of which a unit-test suite touches. Propose adding, on top of the existing pipeline: data schema/drift validation before training, a model-evaluation gate comparing the candidate against the current production model on held-out data, and a registry to track which model version is live. Frame it as extending, not replacing, the existing pipeline — the org's DevOps investment isn't wasted, it's the foundation the MLOps gates sit on top of.

---

**Situation:** A product manager says "we're not doing anything fancy, we just call the OpenAI API with a system prompt — surely we don't need all this MLOps machinery." What would you do and why?

Model answer: Correct the premise directly: even zero-fine-tuning LLM usage has a full LLMOps surface. The system prompt is a versioned artifact that changes behavior when edited — if nobody's tracking prompt versions, a regression from an edited prompt is indistinguishable from a model provider issue. There's real per-request cost and latency to monitor. Output is free-form text, so correctness needs rubric or LLM-as-judge evaluation, not exact-match assertions. And hallucination is a genuinely new failure mode with no unit-test equivalent. Recommend the minimum viable LLMOps surface for this specific case: prompt version control with diffs, a small held-out eval set (even 20-50 examples), and basic cost/latency logging — not the full 26-module platform on day one.

---

**Situation:** Six months after launch, a production incident review reveals nobody can say which system prompt, which retrieved documents, or which model snapshot generated a specific bad customer-facing answer from three weeks ago. What would you do and why?

Model answer: Name this precisely as the "governance gap" — deployment/serving tooling matured faster than lineage/governance tooling industry-wide, and this incident is a textbook instance of it. The fix isn't a one-off investigation; it's closing the structural gap: tag every inference span with the prompt version, the model/version identifier, and (if RAG is involved) the vector index snapshot ID at request time, and ship that tagged trace to a store queryable by request ID. Treat this as a standing requirement going forward (Modules 03, 09, 14, 16, 18 territory), not a one-time forensic exercise, because the same blind spot will recur on the next bad output otherwise.

---

**Situation:** A junior engineer on your team wants to start the course (or a real project) by diving straight into building a LangGraph multi-agent system because "that's the interesting part," skipping the CI/CD and versioning fundamentals. What would you do and why?

Model answer: Don't just say "follow the syllabus" — explain the actual risk: an elaborate agentic system built without release-engineering discipline (versioning, CI/CD gates, a registry) has no way to tell whether a change to the agent's prompts or tool set made things better or worse, and no clean way to roll back a bad deployment. It will look impressive in a demo and be unmaintainable in production within weeks. Suggest a compressed but non-skipped path: get comfortable with model/prompt versioning and a basic promotion gate on a small piece of the system first, then build the agent on top of that foundation — the fundamentals take days, not months, and save weeks of firefighting later.

---

**Situation:** Leadership asks you to justify, in one slide, why the LLM-powered support chatbot project needs its own "LLMOps" line item in the budget when the classical fraud-detection model project didn't need anything like it.

Model answer: Frame it as the same underlying discipline (MLOps) applied to a wider release unit, not a brand-new department. The fraud model's release unit was code + model artifact; the chatbot's release unit is code + prompt template + retrieval index + (potentially) fine-tuned weights — each an independent axis of change that can silently regress the system with zero code changes. The budget line isn't "LLMOps is fancier," it's "there are more moving, independently-versioned parts, so the test/monitoring surface has to be proportionally wider." Back it with the concrete example: a non-engineer editing a system prompt in a shared doc has caused real production regressions industry-wide, and that risk has no classical-MLOps equivalent to catch it.

---

**Situation:** A team lead insists that because their LLM feature has no fine-tuning and no RAG — just a single prompt-in/response-out call to a hosted API — it doesn't need an evaluation dataset or any eval process at all.

Model answer: Disagree, using the "governance gap" and hallucination arguments directly: a single-call LLM feature still produces open-ended text that can be fluently, confidently wrong, and there is no code-level way to assert correctness the way you would on a fixed-schema classifier. Recommend the cheapest defensible starting point — a 20 to 50 example held-out eval set covering the realistic input distribution plus a few known edge cases, scored either by rubric or LLM-as-judge — rather than no evaluation at all. Point out this is far cheaper than a single embarrassing production hallucination discovered by a customer instead of by the team.

---

**Situation:** Your organization has ten different product teams, each independently building LLM features, and every team is reinventing its own ad hoc prompt-versioning convention (some in a Python string, some in a spreadsheet, some in a Notion doc). What would you do and why?

Model answer: Treat this the way this module frames a large platform's "release engineering foundation" — as shared infrastructure, not a per-team decision. Propose a shared prompt/model registry (the concrete tooling comes in Module 03/13) that every team adopts, rather than each team re-deriving the same versioning discipline at different quality levels. The argument for centralizing: at ten teams' scale, the entanglement/CACE-style risk of an undocumented prompt edit causing a regression is now an organizational risk, not just a per-team one, and a shared registry means an incident in one team's system can be debugged with the same tooling and vocabulary as any other.

---

**Situation:** During a hiring interview, a senior candidate claims "LLMOps is a completely separate discipline from MLOps, with its own tooling stack, and MLOps skills mostly don't transfer." How would you evaluate this answer?

Model answer: Treat it as a meaningful red flag rather than a valid opinion — the well-supported industry framing (and this module's central claim) is that LLMOps is an extension layer on MLOps, not a rival discipline: it reuses the versioning/CI-CD/monitoring foundation and adds prompt versioning, retrieval/vector-index management, token economics, and LLM-as-judge evaluation on top. A candidate who frames it as a total replacement likely hasn't operated a real production LLM system long enough to hit the classical failure modes (drift, silent regressions, rollback complexity) that LLMOps still has to solve using MLOps-derived tools. Probe further by asking them to name three things LLMOps reuses from MLOps — a strong candidate answers immediately (versioning, CI/CD gates, monitoring discipline).

---

**Situation:** A stakeholder asks why the course (and, by extension, your team's real roadmap) puts observability (Module 14) before building RAG and agent systems (Modules 15-18), when most teams they've seen build the flashy feature first and add monitoring "once something breaks."

Model answer: Explain this is a deliberate correction of an extremely common and expensive real-world ordering mistake: teams that build RAG/agent systems before observability exists cannot debug the first serious production incident without ad hoc log-diving, and by the time observability gets bolted on retroactively, the team has often accumulated months of un-investigable "the bot said something weird" reports with no trace data to diagnose them. Building minimal observability (structured logging with trace IDs) before the flagship architecture costs relatively little and avoids months of blind production operation — cite this module's explicit "under-investing in observability until after an incident" common mistake as the industry pattern being deliberately avoided.

---

**Situation:** Your company operates both a stateless internal admin tool (no ML, no LLM) and a customer-facing LLM copilot. A new engineer proposes applying the same model registry, drift dashboard, and LLM-as-judge evaluation harness to both, "for consistency."

Model answer: Use the module's decision-tree logic explicitly: walk each system through "does it make a prediction/generate content/decide using a model?" The admin tool answers no — plain DevOps is sufficient, and bolting on MLOps machinery there is pure overhead with zero corresponding safety benefit. The copilot answers yes and further "does it call an LLM" — yes, so it needs at minimum versioned prompts, cost/latency tracking, and output evaluation. Consistency for its own sake is the wrong goal here; the right goal is matching engineering investment to actual risk, and applying identical heavyweight tooling to a system with no learned/generative component only trains the org to see MLOps machinery as bureaucratic overhead rather than as a real safety net where it's actually needed.

---

**Situation:** A team that has been running a classical ML pricing model in production for three years is now adding an LLM-generated natural-language explanation feature on top of it ("Because of X, Y, Z, your quote is $..."). They ask whether this counts as a new LLMOps initiative or just a UI feature on an existing MLOps system.

Model answer: It's a graft of an LLMOps ring onto an existing MLOps ring, and both need to be treated as live simultaneously, not merged into one undifferentiated system. The pricing model still needs its existing MLOps discipline (registry, drift monitoring, evaluation gates) untouched. The new explanation feature adds a genuinely new LLMOps surface: a versioned prompt template, a new failure mode (the explanation could misstate or contradict the actual pricing factors — a coherence/faithfulness problem distinct from the pricing model's own accuracy), and its own evaluation gate (an LLM-judge-scored coherence check against the pricing factors actually used, similar to the "Because you watched" explanation example in this module). Recommend gating deployment of the explanation feature on this new evaluation, independent of the pricing model's existing gates.

---

**Situation:** Partway through the course, a study partner asks whether it's safe to skip straight to Module 16 (Vector Databases) because they already know Docker and Kubernetes cold, want a reference now, and plan to "circle back" to earlier modules eventually.

Model answer: Distinguish reference lookup from first-pass learning. Jumping to Module 16 purely to look up a vector-database comparison is fine and exactly what the module's dependency-view section anticipates. But treat "circle back eventually" skeptically for a first pass through Modules 07-14 specifically — the RAG module assumes working vocabulary and running code from the prompting, evaluation, and observability modules, and debugging a RAG system without evaluation discipline already in place is exactly the "flying blind" failure mode called out as a common mistake. Recommend confirming they can already state, from memory, what a promotion gate and an LLM-as-judge eval set are before treating M15-18 as safe to attempt standalone.

---

**Situation:** After the capstone project (Module 26) ships, an engineer proposes writing a retrospective doc claiming "we've now fully implemented LLMOps for this system." How would you evaluate that claim, and what would you check before signing off on it?

Model answer: Treat "fully implemented LLMOps" as a claim to verify against the Module 01 architecture diagram's bands, not accept at face value. Walk the capstone against each band: is there a registry layer for model, prompt, and vector-index versions (not just the model)? Are promotion gates code-driven rather than manual? Is there an observability layer tagging every request with the deployment triple? Is there a feedback/drift loop that can trigger re-evaluation? If any band is missing — commonly the governance/lineage tagging, since it's the newest and most-skipped piece — the honest retrospective names the gap explicitly rather than claiming completeness, consistent with the module's framing that most real teams underinvest here first.

---

**Situation:** A cost-conscious founder at an early-stage startup says the 26-module course's ~130-170 hour, 16-18 week estimate is "way too much" for a two-person team that needs to ship an LLM feature in three weeks, and asks what to actually prioritize.

Model answer: Don't defend the full course as mandatory — triage against actual blast radius, exactly as the module's own "when to apply full MLOps rigor" guidance suggests. For a real production feature going live in three weeks, prioritize: minimal prompt versioning (even a git-tracked file, not a full registry), a small held-out eval set before each prompt change ships, basic cost/latency logging, and structured request logging with trace IDs — a few days of work, not weeks. Explicitly defer the heavier infrastructure (Kubernetes at scale, full experiment tracking, drift dashboards) until there's real production traffic and a real scaling problem to justify it, while designing the prompt-handling code so those additions don't require a rewrite later.

---

**Situation:** During a post-incident review, someone asks "was this incident actually preventable by anything in this course, or was it just bad luck?" — the incident being a subtly-broken production RAG answer traced back to an untracked vector-index rebuild that happened the same week as an unrelated prompt edit.

Model answer: Identify this as directly preventable, and name the specific gap: this is the deployment-triple/lineage problem from Section 3.1 and the architecture deep-dive's sequence diagram — if every span had been tagged with the prompt version and the vector-index snapshot ID at request time, the postmortem would have been a five-minute lineage query ("which index snapshot and prompt version were live for this trace_id") instead of a multi-day investigation. Recommend the fix not as "add more monitoring" generically, but specifically: wire index-snapshot IDs and prompt versions into the same structured log/trace records used for cost and latency, closing exactly the "governance gap" this module names as a known, common, and fixable industry shortfall.

---

**Situation:** A candidate in a technical interview is asked to draw the "three rings" diagram (DevOps inside MLOps inside LLMOps) from memory and instead draws them as three separate, non-overlapping circles. How would you use this to probe further, and what would a strong follow-up answer look like?

Model answer: Use it as a diagnostic, not an automatic fail — ask them to explain what happens to DevOps practices (CI builds, code review, container security scanning) once a team adopts MLOps, then LLMOps. A weak answer treats them as replaced. A strong answer explains that each ring is nested and reuses everything inside it: an LLMOps team still runs every DevOps practice and every MLOps practice, adding prompt/retrieval/token-cost/LLM-judge concerns on top rather than instead. If the candidate self-corrects to the nested model once prompted, that's a strong signal; a candidate who defends the three-separate-circles model even after the prompt is a real gap worth flagging, since it suggests they'd advocate discarding working CI/CD discipline when a team adopts LLM features.

---

**Situation:** Your team just discovered that a hosted foundation-model provider silently updated the default model behind an API endpoint you call without a version pin, and your evaluation metrics quietly drifted over two weeks before anyone noticed.

Model answer: Name this precisely as an LLMOps-specific "what can cause a regression with zero code change" case from this module's comparison table — specifically an upstream foundation-model version bump, which has no classical-MLOps or DevOps equivalent. Fix it at two levels: immediately, pin to an explicit model version/snapshot string wherever the API allows it, and treat "provider changed the default model" as a first-class entry in your rollback/incident runbook. Structurally, add automated eval-set reruns on a schedule (not just on your own deploys) specifically to catch drift introduced by upstream changes outside your own release cycle — this is the kind of silent, non-deploy-triggered regression that makes LLMOps monitoring qualitatively different from watching your own CI/CD pipeline.

---

**Situation:** A new hire asks you, informally, "what's actually the single biggest thing that separates a team that's good at LLMOps from one that's bad at it, in your experience?"

Model answer: Point to evaluation discipline as the highest-leverage differentiator, per this module's common-mistakes framing: teams that under-invest in Modules 09-12-equivalent evaluation methodology to chase "cooler" work like agentic systems end up unable to tell whether any given change made their system better or worse, which means every subsequent decision is a guess dressed up as an improvement. Teams that get evaluation right early can iterate confidently and catch regressions before users do; teams that skip it accumulate technical and reputational debt that compounds, because every downstream module in this course — observability, RAG, agents, drift detection — assumes a working evaluation signal to check itself against.
