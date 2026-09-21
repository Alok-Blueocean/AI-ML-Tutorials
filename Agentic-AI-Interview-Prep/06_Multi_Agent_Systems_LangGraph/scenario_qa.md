# Multi-Agent Systems: LangGraph — Scenario-Based Q&A

**Situation:** Your production LangGraph agent intermittently throws `InvalidUpdateError: At key 'notes', got multiple values for the same key at the same super-step`. It happens maybe once in every few hundred runs. What would you do and why?

Model answer: This is a concurrent-write conflict, not a flaky bug — some superstep has two branches (parallel fan-out from a conditional edge, or two nodes both scheduled in the same step) both returning an update to the `notes` key, and LangGraph has no reducer telling it how to combine them, so it raises rather than silently picking one. First, find the fan-out: look for any point where more than one node can be active in the same step and both touch `notes`. Then decide the correct merge semantics — usually "append" (`Annotated[list, operator.add]`) if `notes` is meant to accumulate contributions from multiple agents, or restructure the graph so only one branch owns that key if overwriting was actually intended. Don't reach for a lock or a random tie-breaker; the fix belongs at the state-schema level, not the execution layer.

---

**Situation:** You're choosing a checkpointer backend for a LangGraph agent going to production: `MemorySaver`, `SqliteSaver`, or a Postgres-backed checkpointer. The team is deciding between a single-instance deployment and a horizontally-scaled multi-instance deployment. What would you do and why?

Model answer: `MemorySaver` is a non-starter for production regardless of scale — it's in-process RAM, so a restart or redeploy silently loses every in-flight thread's state, which defeats the purpose of checkpointing. For a genuinely single-instance deployment, `SqliteSaver` is a reasonable, low-operational-overhead choice — it's file-backed and durable across restarts on that one instance. The moment you're running multiple instances behind a load balancer (or need to survive the instance itself being replaced, e.g., in a container orchestrator), you need the Postgres-backed checkpointer so any instance can pick up any `thread_id`'s state — Sqlite's single-file model doesn't handle concurrent multi-instance access safely. Frame the decision explicitly around instance count and restart tolerance, not just "which one sounds more production-grade."

---

**Situation:** You're designing a human-in-the-loop approval gate for a LangGraph agent before it executes a $50,000 wire transfer. What would you do and why?

Model answer: Put the `interrupt()` call structurally immediately before the transfer tool executes, not earlier in the flow where a human could approve a plan that later changes — the interrupt should see the exact final arguments (amount, recipient) that will actually be executed. The payload sent to the reviewer needs to be a complete, self-contained decision packet (amount, recipient, source account, and anything explaining why the agent decided to act), not just "approve this step?" Back the graph with a persistent checkpointer, not `MemorySaver` — the whole point of a human approval gate is that it can sit paused for minutes or hours, and a process restart during that window must not lose the pending request. Finally, treat rejection as a first-class path with its own node/logic (log, notify the requester, do not silently retry), not just an early return.

---

**Situation:** You're asked whether a new multi-agent business process should use a supervisor/orchestrator-worker pattern or a swarm/peer-to-peer hand-off pattern. The process is "route and resolve an inbound customer complaint across billing, technical support, and account security teams, where any team can determine mid-investigation that another team should actually own it." What would you do and why?

Model answer: Lean swarm here, not supervisor — the defining detail is "any team can determine mid-investigation that another team should own it," which means there's no single coordinator who always has enough information upfront to route correctly; control needs to pass directly between specialists as new information surfaces. A supervisor pattern would force every hand-off decision back through a central router that has to be kept in sync with what each specialist just learned, adding a hop and a potential bottleneck for no real benefit. If instead the process were "classify once, dispatch to exactly one team, done," a supervisor would be simpler and more debuggable, since there's a natural single coordinator and no need for peer-to-peer complexity. The deciding factor to name explicitly in an interview: does a single coordinator have enough information to route correctly every time, or does routing information only emerge mid-task from the specialists themselves?

---

**Situation:** A teammate proposes building a shared fraud-investigation subgraph so it can be reused across both the claims-processing graph and a separate loan-underwriting graph. What would you do and why?

Model answer: This is a good use of subgraphs structurally, but validate the state-schema compatibility first — the two parent graphs likely have different state shapes, so the subgraph needs either a shared minimal schema (only the fields fraud investigation actually needs) with explicit transformation at the call site, or to be invoked as a "call inside a node" rather than "added directly as a node" so state mapping is explicit rather than assumed via shared keys. Insist the subgraph be tested standalone with its own test suite before it's wired into either parent, since a bug introduced for one consumer would otherwise silently affect the other. This is the right instinct (avoid duplicating fraud logic) executed carefully rather than by just copy-pasting nodes into a common file.

---

**Situation:** A node in your production graph calls an external payment API and the process crashes mid-call, before the node returns. On restart, the graph resumes and the node reruns from its start — and calls the payment API a second time. What would you do and why?

Model answer: This is exactly the durability caveat worth naming precisely: checkpointing saves state between completed steps, not mid-step, so a node that dies mid-execution reruns entirely on resume rather than continuing from where it left off. The fix isn't in LangGraph's persistence layer — it's making the node's external call idempotent, typically via an idempotency key (e.g., a claim ID plus attempt counter, or a key generated once and stored in state before the call) that the payment API's SDK or your own wrapper uses to detect and safely no-op a duplicate request. Any node that calls a non-idempotent external side effect needs this treatment; it's a design requirement that follows directly from how LangGraph's checkpointing actually works, not an edge case to patch over later.

---

**Situation:** Product wants a customer-facing chat UI to show live progress ("checking your policy now...") while a supervisor-pattern claims agent runs, and separately the engineering team wants full step-by-step traces for debugging production issues. Someone suggests building one custom logging solution to cover both. What would you do and why?

Model answer: Keep these as two separate mechanisms even though both start from the same underlying events. Use LangGraph's built-in step-level streaming, surfaced through whatever transport the UI already uses, for the customer-facing progress messages — it's designed for exactly this and needs no extra infrastructure. For engineering-facing debugging, wire the graph to LangSmith instead of hand-rolling logging — it gives full execution traces, and more importantly a foundation for dataset-based regression evaluation later, which a custom logging solution won't provide without significant extra build. Building one bespoke system to serve both jobs means under-serving both: customer-facing streaming needs to be fast and minimal, debugging traces need to be complete and queryable, and those are different design constraints.

---

**Situation:** In a system design interview, you're asked to design a multi-tenant SaaS agent platform where each tenant's conversation history and in-flight agent runs must stay strictly isolated. What would you do and why?

Model answer: Use `thread_id` as the isolation boundary, but don't treat it as a security boundary by itself — scope every `thread_id` to a tenant explicitly (e.g., `{tenant_id}:{conversation_id}`, checked against the caller's tenant claim before invoking the graph), since LangGraph itself doesn't enforce access control on threads. Flag the documented Postgres constraint that `thread_id` values must stay under 255 characters, which matters once you're composing tenant ID plus conversation ID plus any versioning into the key. For genuinely strict isolation requirements (regulated data, contractual tenant separation), consider whether a shared checkpointer database is even acceptable, versus per-tenant database schemas or instances — that's a real production decision this design surfaces, not just a LangGraph configuration detail.

---

**Situation:** Six months into running a single well-tooled ReAct agent (no LangGraph) for internal IT ticket triage, a teammate proposes migrating it to a LangGraph supervisor with five specialist agents "because that's the more scalable architecture." What would you do and why?

Model answer: Ask what's actually breaking today before agreeing. If the single agent's tool set has grown unwieldy, its context is bloated with irrelevant tools per request, or its accuracy is measurably degrading as ticket categories grow, that's real evidence for specialization and a supervisor is a reasonable next step. If it's running fine and the proposal is motivated by "LangGraph is what real production agent systems use," push back — migrating a working single-agent system to a five-agent graph adds real cost (more latency from inter-agent hops, more surfaces for the concurrent-write and routing bugs covered elsewhere in this topic, more to debug) that needs to be justified by a measured constraint, not architectural fashion. This is the same "start simple, add agents only when a stated constraint requires it" principle from topic 05, applied to an existing system rather than a new one.
