# Autonomous Agent Design — Scenario-Based Q&A

**Situation:** A production support agent occasionally gets stuck calling the same `search_knowledge_base(query="...")` tool with the exact same query 15+ times before hitting your timeout. What would you do and why?

Model answer: Treat this as an infinite-loop failure mode, not a "smarter prompt" problem. First add a cheap structural guardrail — hash (tool name, arguments) for recent calls and refuse to re-execute an identical call, short-circuiting with the prior result instead. Then investigate why the model thought repeating was progress: usually the tool's response is ambiguous (looks different enough each time to seem like new information, or looks like an error the model thinks retrying will fix). Fix the tool to return an explicit, unambiguous terminal status, and only after that consider whether the step budget itself needs tightening. A step cap alone is a backstop, not a fix — ship the duplicate-call guard first.

---

**Situation:** Your company is scoping a new claims-processing feature and a junior engineer proposes a 5-agent LangGraph system (intake, extraction, policy lookup, fraud check, decision) on day one. What would you do and why?

Model answer: Push back and ask what's driving the 5-way split before writing any graph code. If the five steps are just sequential and each one is a single well-defined tool call, one well-tooled ReAct agent with five tools is simpler to build, cheaper to run, and far easier to debug than five coordinated agents. Multi-agent orchestration earns its complexity when there's genuine specialization with different context/tool needs (e.g., fraud-check needing access to sensitive external data the intake agent shouldn't see) or a need for parallelism. Recommend starting with the single-agent version, instrumenting it, and only splitting out a specialist agent where the single agent's accuracy or context size demonstrably suffers.

---

**Situation:** An agent handling refund requests calls `issue_refund(order_id="ORD-9981", amount=450.00)` but ORD-9981 was an order the user asked about three turns ago that the agent never actually looked up in this session — it invented the ID from a similar-looking one mentioned earlier. What would you do and why?

Model answer: This is tool-call argument hallucination, specifically a "stale/fabricated argument" case, and it's dangerous because refunds are irreversible. The fix is a validation gate, not a prompt tweak: require that any argument feeding an irreversible tool call be traceable to a value the agent actually retrieved via a tool in the current episode (not just present anywhere in the conversation), and reject/re-prompt if it isn't. Additionally, gate `issue_refund` behind a human-approval or a second confirmatory lookup step given the blast radius of getting it wrong — the fix belongs at the guardrail layer, since no amount of prompt engineering fully eliminates hallucination risk.

---

**Situation:** You built a Reflexion-style self-critique retry loop for a code-generation agent, and after three retries it's still failing the same unit test, just with slightly different wrong code each time. What would you do and why?

Model answer: This usually means the self-critique isn't diagnosing the real root cause — it's generating plausible-sounding but generic critiques ("the logic might be off") instead of a specific, falsifiable one tied to the actual test failure output. Improve the critique step by feeding it the actual assertion error/diff, not just "it failed," so the reflection is grounded in concrete evidence. Also cap retries (Reflexion is not guaranteed to converge) and, if three grounded-critique attempts still fail, escalate to a human or a different strategy (e.g., decompose the function into smaller testable pieces) rather than retrying indefinitely with the same approach.

---

**Situation:** A stakeholder asks why your research assistant agent sometimes takes 45 seconds and sometimes 4 seconds for what looks like a similar question. What would you do and why?

Model answer: Explain that this is expected in an agentic (vs. workflow) system — the model dynamically decides how many tool calls/reasoning steps it needs, so latency is a function of task difficulty, not a fixed pipeline cost. To make this defensible rather than just "it's variable," instrument step counts and per-step latency so you can show the distribution, set a reasonable step-budget ceiling for the worst case, and consider a plan-and-execute redesign if the business needs more predictable latency — plan-and-execute makes the step count visible up front, at the cost of some flexibility.

---

**Situation:** Your team wants agent memory so a customer-facing assistant "remembers" past interactions, and someone suggests just increasing the context window and putting the full conversation history from every past session into every prompt. What would you do and why?

Model answer: Push back on conflating working memory with long-term memory. Full history in every prompt doesn't scale (cost, latency, and the model's tendency to get distracted by irrelevant old context), and it isn't actually "memory" in the useful sense — it's undifferentiated history. Recommend a proper long-term memory layer: extract and store discrete facts/preferences (not raw transcripts) in a vector store, retrieve only what's relevant to the current query via similarity search, and keep the current session's working memory separate and short-lived. This is architecturally the same shape as RAG, applied to the agent's own experience instead of a document corpus.

---

**Situation:** During a design interview, you're asked to design an agent for "automatically triaging and routing inbound IT support tickets across 200 possible tool integrations (one per internal system)." What would you do and why?

Model answer: Flag tool-selection accuracy as the central risk at this scale — exposing all 200 tool schemas to one reasoning call would blow up both cost and wrong-tool-call rate. Propose a two-stage design: a fast classification/routing step (keyword or embedding-based) that narrows to the relevant department's tool subset (typically 10-20 tools), then a ReAct-style agent reasoning only over that narrowed set. Discuss the fallback path when the router itself is uncertain (route to a human, or to a small default toolset) so an ambiguous ticket doesn't get silently mis-routed.

---

**Situation:** A pilot of a plan-and-execute travel-booking agent works fine in testing but in production it books a hotel for the wrong dates because the flight search (step 1) returned a different available date range than assumed when the upfront plan was written. What would you do and why?

Model answer: This is the classic plan-and-execute staleness failure — a plan built entirely upfront can't react to information that only becomes available mid-execution. Fix it by allowing re-planning after each step's observation (check whether step 1's actual result still satisfies step 2's assumptions before executing it), rather than blindly running the fixed plan to completion. This is a deliberate design tradeoff to surface in an interview: plan-and-execute gains auditability and fewer "what next" LLM calls, but only stays correct if you build in a re-planning checkpoint for exactly this kind of dependency between steps.

---

**Situation:** Leadership asks whether your single-agent customer service bot "needs to become multi-agent" now that it's being asked to also handle billing disputes in addition to general support questions.

Model answer: Don't default to yes. Ask what specifically breaks today: if the single agent's tool set and instructions are getting unwieldy (billing tools polluting context for general questions, or the model confusing which policy applies), that's a real signal for a supervisor plus a billing specialist and a general-support specialist. If the two categories share most tools and just need a slightly different system prompt, a single agent with a lightweight intent-branch (or even two prompt variants behind a simple router) solves it without orchestration overhead. Frame the recommendation around measured degradation (accuracy, wrong-tool-call rate, context bloat), not the assumption that more agent types is automatically better architecture.
