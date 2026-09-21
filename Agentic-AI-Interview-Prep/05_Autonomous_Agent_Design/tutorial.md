# Autonomous Agent Design — Condensed Study Notes

## What Makes Something an "Agent" (vs. a Workflow)

- Anthropic's widely-cited framing: a **workflow** orchestrates LLMs and tools through predefined code paths (you hardcode the steps); an **agent** lets the LLM dynamically direct its own process and tool usage, deciding at runtime what to do next and when it's done. Most production systems sit somewhere on this spectrum, not at either extreme.
- Rule of thumb: use the simplest structure (a single prompt, then a fixed workflow) that meets the requirement, and only reach for agentic autonomy when the number of steps genuinely can't be predicted in advance.
- Real-world example: an invoice-approval system that always does "extract fields -> validate against PO -> route for approval" is a workflow (fixed path); a research assistant that decides for itself how many searches to run and when it has "enough" information is an agent (open-ended path length).

## The ReAct Loop (Reason + Act)

- ReAct interleaves reasoning traces ("Thought: I need the customer's order status") with actions (tool calls) and observations (tool results), repeating until the model decides it has enough information to answer. This is the default mental model behind almost every tool-calling agent today.

```
loop:
    thought = llm.reason(scratchpad)          # "what should I do next and why"
    if thought.is_final_answer:
        return thought.answer
    action = thought.tool_call                # e.g., search(query="...")
    observation = execute_tool(action)
    scratchpad += (thought, action, observation)
```

- Real-world example: a customer-support agent handling "where's my order and can I get a refund" reasons -> calls `get_order_status` -> observes it shipped 2 days ago -> reasons again -> calls `check_refund_policy` -> observes the item is past the return window -> produces a final answer citing both facts.
- Paper: Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022).

## Reflexion: Self-Critique Loops

- Reflexion extends ReAct with a verbal self-critique step: after a failed or low-quality attempt, the model generates a natural-language reflection on what went wrong ("I searched for the wrong product ID"), which gets prepended to context on the next attempt. No gradient updates — the "learning" is entirely in-context, across attempts within the same episode or task.
- This is distinct from a single ReAct loop's thought/action/observation — Reflexion adds an outer retry loop with an explicit critique artifact that persists across attempts.
- Real-world example: a code-generation agent runs unit tests, they fail, and instead of blindly retrying the same code it first writes "the tests failed because I assumed the input was sorted — it isn't," then regenerates code with that correction in context.
- Paper: Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning" (2023).

## Plan-and-Execute

- Instead of deciding one step at a time (ReAct), a planner LLM first produces a multi-step plan up front; an executor then works through the plan step by step, optionally re-planning if a step fails or new information changes the picture. Fewer LLM calls for the "what's next" decision, and the plan is inspectable/auditable before any tool runs — valuable when actions are costly or risky.
- Real-world example: a trip-booking agent plans "1) check flight availability, 2) check hotel availability for those dates, 3) compare total cost against budget, 4) book if under budget, else ask user" before executing anything, so a human reviewer can sanity-check the plan before step 4 touches a payment method.
- Tradeoff: a fixed upfront plan can go stale if step 2's result should change step 1's choice — most production plan-and-execute systems allow re-planning after each step's observation, which blurs the line back toward ReAct.

## Tree-of-Thought

- Instead of one linear reasoning chain, the model explores multiple candidate "thoughts" at each step, forming a tree, and uses a search strategy (BFS/DFS) plus a self-evaluation heuristic to prune weak branches and pursue promising ones. Useful for problems where the first plausible-sounding step isn't necessarily the right one (puzzles, planning under constraints, code with several valid approaches).
- Real-world example: a scheduling agent trying to fit 8 meetings into 3 conflicting rooms explores several candidate assignments in parallel, scores each partial assignment for feasibility, and abandons branches that create an unresolvable conflict early rather than discovering it only at the end.
- Cost tradeoff: multiple candidate generations and evaluations per step multiply token/latency cost versus a single ReAct pass — reserve it for problems where getting stuck in a bad local path is expensive to recover from.
- Paper: Yao et al., "Tree of Thoughts: Deliberate Problem Solving with Large Language Models" (2023).

## Agent Memory: Short-Term vs. Long-Term

- **Short-term / working memory**: the current conversation/task scratchpad — the running list of thoughts, actions, and observations within one episode. Lives in the context window; disappears when the episode ends unless explicitly persisted.
- **Long-term memory**: durable knowledge that survives across episodes/sessions, typically stored in a vector database and retrieved (embedding similarity search) into context when relevant, similar in mechanism to RAG but storing the agent's own past experiences/facts rather than a static document corpus.
- Real-world example: a personal-assistant agent uses working memory to track "which flight the user just picked" mid-conversation, and long-term vector memory to recall "this user always prefers aisle seats," retrieved and injected into context the next time flight booking comes up, weeks later.
- Design point: long-term memory needs a write policy (what's worth remembering, deduplication, decay/eviction) as much as a read policy — unbounded memory growth degrades retrieval quality just like an unbounded RAG index does.

## Tool Selection and Routing

- As the number of available tools grows, dumping every tool schema into every prompt hurts both latency/cost (bigger context) and accuracy (the model is more likely to pick the wrong tool or hallucinate parameters when choosing among 40 similar-looking tools than among 5). Common mitigations: group tools into named toolsets and only load the relevant set per task type; use a lightweight router step (classification, or embedding similarity over tool descriptions) to narrow candidates before the main reasoning call; keep tool names and descriptions unambiguous and non-overlapping.
- Real-world example: an internal IT-helpdesk agent has 60 tools across HR, IT, and finance systems; instead of exposing all 60 to every request, a fast intent-classification step first routes to the right department's tool subset (12-15 tools), which measurably cuts wrong-tool-call rate.

## Handling Agent Failure Modes

- **Infinite loops**: the agent repeats the same tool call (or a tight cycle of calls) without making progress, usually because the exit condition never becomes true or the model keeps "forgetting" it already tried something. Mitigations: hard step/iteration caps as a backstop, deduplicate identical (tool name + arguments) calls within an episode and short-circuit repeats, require tools to return an explicit terminal status (done/failed) rather than an ambiguous result, and log repeated-span traces so this is detectable, not just preventable.
- **Tool-call hallucination**: the model invents a tool that doesn't exist, or calls a real tool with malformed/fabricated arguments (e.g., an order ID that was never retrieved). Mitigations: validate tool-call schema and argument types before dispatch and reject/re-prompt on failure rather than executing anything; constrain generation with structured output / function-calling APIs instead of free-text parsing; never let a hallucinated argument reach an irreversible action (payment, delete, send) without a validation or approval gate.
- **Getting stuck** (neither looping nor progressing, just unable to find a valid next action): usually a sign the tool set is insufficient for the task, or the task decomposition was wrong. Mitigations: a timeout/step-budget that surfaces the partial state to a human rather than failing silently, and treating repeated "no good next action" as its own loggable failure category distinct from an infinite loop.
- Real-world example: a claims-processing agent kept calling `lookup_policy(policy_id=None)` in a loop because an earlier step silently failed to extract the policy ID; a hash-based duplicate-call detector caught it after 3 identical calls and routed the case to a human instead of burning the full step budget.
- This is the design-time half of agent reliability; the runtime observability tooling to *detect* these failures in production (tracing, span-level logging, failure clustering) is covered in depth in the sibling LLMOps course's tracing/debugging module — treat that as the operational counterpart to the failure modes above, not a duplicate topic.

## Single-Agent vs. Multi-Agent: Decision Criteria

- Start with one well-tooled agent. Multi-agent orchestration adds real cost: more LLM calls (hence latency and spend), harder debugging (which agent said what, in what order), and coordination failure modes that don't exist in a single agent (agents talking past each other, duplicated work, one agent's error propagating into another's context).
- Reach for multiple agents when: the task cleanly decomposes into specialist roles with genuinely different tool sets/context needs (e.g., a legal-review agent shouldn't see raw payment data a billing agent needs); you need parallelism across independent subtasks (concurrent research on unrelated sub-questions); or a single agent's context window/tool count would otherwise get overloaded and its instruction-following degrades.
- A single agent is usually the better choice when the task, however complex, is fundamentally one coherent reasoning thread with one tool "vocabulary" — adding orchestration overhead there tends to make the system slower and harder to debug without improving quality.
- Real-world example: a claims-processing pipeline that only ever does document extraction, then a policy lookup, then a decision, is well served by one agent with three tools; the same company's fraud-investigation workflow — which needs a document-extraction specialist, a policy-lookup specialist, and an external-data-enrichment specialist running in parallel and reconciling conflicting findings — is a better fit for a supervised multi-agent graph (see the next topic).

## Quick Gotchas Worth Naming in an Interview

- "Agentic" is a spectrum, not a binary — most production systems mix fixed workflow steps with a bounded agentic sub-loop, rather than being pure ReAct end to end.
- A step-count cap is a necessary backstop against infinite loops, but it is not a fix for the underlying cause — always pair it with duplicate-call detection and clear terminal tool states.
- More tools is not free: tool-selection accuracy degrades as the tool list grows, so routing/grouping matters as much as which tools you build.
- Multi-agent systems do not make an unreliable single agent reliable — if one agent hallucinates tool arguments, adding more agents around it usually just gives the hallucination more places to propagate.
