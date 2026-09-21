# Multi-Agent Systems: LangGraph — Condensed Study Notes

Docs referenced throughout: `docs.langchain.com/oss/python/langgraph/` (the docs moved here from the old `langchain-ai.github.io/langgraph/` URL, which now redirects).

## What LangGraph Is and Why It Exists

- LangGraph is a low-level orchestration framework and runtime for building stateful, long-running agents as an explicit graph rather than a hidden while-loop. It does not prescribe a fixed agent architecture — you compose your own graph shape (nodes and edges) instead of configuring a black-box "agent" object.
- It sits below LangChain's higher-level abstractions: LangChain gives you chains/prebuilt agents for common cases; LangGraph gives you the primitives to build a bespoke agent when the prebuilt shape doesn't fit.
- Real-world example: a claims-processing pipeline that needs "extract documents, then loop on policy lookup until confidence is high, then require human sign-off above a dollar threshold" doesn't fit a single linear chain — it's exactly the branching/looping/pausing shape LangGraph is built for.

## Core Primitives: State, Nodes, Edges

- **State**: a shared schema (typically a `TypedDict` or Pydantic model) that flows through the graph. Every node reads from and writes updates to this state.
- **Nodes**: plain functions (or LLM/tool calls) that take the current state and return a partial update to it.
- **Edges**: connections between nodes — either direct (always go from A to B) or conditional (a function inspects the state and decides which node to go to next).

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END

class AgentState(TypedDict):
    messages: list
    next_step: str

def call_model(state: AgentState) -> dict:
    # ... call the LLM, decide whether to call a tool or finish
    return {"messages": state["messages"] + [response], "next_step": "tools" if wants_tool else "end"}

def call_tool(state: AgentState) -> dict:
    result = execute_tool(state["messages"][-1])
    return {"messages": state["messages"] + [result]}

graph = StateGraph(AgentState)
graph.add_node("agent", call_model)
graph.add_node("tools", call_tool)
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", lambda s: s["next_step"], {"tools": "tools", "end": END})
graph.add_edge("tools", "agent")   # cycle back for another reasoning step

app = graph.compile()
```

*(API surface simplified for clarity — verify exact signatures against current docs before quoting from memory in an interview; LangGraph's Python API has evolved across versions.)*

- Real-world example: a document-review agent's graph has an `extract` node, a `validate` node, and a conditional edge that routes back to `extract` if validation fails confidence threshold, or forward to `summarize` if it passes — this cycle is the "reasoning loop" made explicit and inspectable instead of hidden inside a single LLM call's internal chain-of-thought.

## Cyclic Graphs vs. Simple DAGs

- A plain DAG (directed acyclic graph) — the shape most workflow tools assume — can't represent "keep retrying until X" or "loop between two agents until a supervisor is satisfied," because a DAG by definition has no cycles. LangGraph explicitly supports cycles, which is the structural reason it's suited to agentic loops (ReAct-style reasoning, retry-until-valid, multi-agent back-and-forth) rather than just fixed pipelines.
- Real-world example: a `agent -> tools -> agent` cycle (shown above) is a two-node cycle — this is a ReAct loop expressed as a graph rather than as a Python while-loop, but with the same result on every iteration checkpointed and inspectable.

## Persistence and Checkpointing

- LangGraph automatically saves state after each step via a **checkpointer** — `MemorySaver` for local development, `SqliteSaver` for single-instance production use, and a Postgres-backed checkpointer for multi-instance production scale. This gives durable execution: if the process crashes mid-run, execution can resume from the last completed step rather than starting over.
- Each persisted execution is tied to a `thread_id` — reusing a thread ID resumes that conversation/run's saved state; a new thread ID starts fresh. This is the mechanism behind both long-running agent resilience and multi-turn conversational memory.
- Distinction worth naming precisely: checkpointing saves state *between completed steps* — if a node itself fails mid-execution, the in-progress work inside that node is lost and the node reruns on retry, it does not resume mid-node.
- Real-world example: a long-running research agent that takes 10 minutes and 40 tool calls can be safely restarted after an infrastructure blip without redoing the first 35 calls, because each step's state was checkpointed as it completed.

## Human-in-the-Loop Interrupts

- The `interrupt()` function, called inside a node, pauses graph execution at that exact point, persists state via the checkpointer, and returns a JSON-serializable payload to the caller describing what it needs. Execution resumes later by invoking the graph again with `Command(resume=<value>)`, which becomes the return value of the original `interrupt()` call inside the node.
- This is compiled into the graph as a structural feature, not bolted on with a separate polling mechanism — a persistent (database-backed, not in-memory) checkpointer is required for this to survive process restarts.
- Real-world example: a claims-processing graph interrupts before a `pay_claim(amount=...)` tool call whenever the amount exceeds $5,000, surfacing the proposed payout to a human reviewer; the graph resumes exactly at the payment node with the reviewer's approve/reject decision once `Command(resume=...)` is sent back.

```python
from langgraph.types import interrupt, Command

def pay_claim_node(state):
    if state["amount"] > 5000:
        decision = interrupt({"question": "Approve payout?", "amount": state["amount"]})
        if decision != "approve":
            return {"status": "rejected"}
    process_payment(state["amount"])
    return {"status": "paid"}

# Resuming after a human reviews the interrupt payload:
# app.invoke(Command(resume="approve"), config={"configurable": {"thread_id": "claim-123"}})
```

## Subgraphs for Multi-Agent Hierarchies

- A compiled graph can be used as a node inside a larger parent graph (a subgraph). State flows into the subgraph, is processed by its own internal nodes/edges, and the final subgraph state flows back to the parent. This is how you compose hierarchies: a top-level supervisor graph where each "worker" is itself a full graph (potentially with its own internal ReAct loop).
- Real-world example: a top-level insurance-claims graph has a supervisor node plus two subgraph nodes — a document-extraction subgraph (its own extract/validate/retry cycle) and a fraud-investigation subgraph (its own multi-step lookup loop) — each independently testable before being wired into the parent.

## Supervisor / Orchestrator-Worker Pattern

- The most common LangGraph multi-agent shape: a supervisor node (itself an LLM call, or simple routing logic) inspects the current state/task and decides which worker agent to invoke next; workers are added as nodes (often subgraphs), and control returns to the supervisor after each worker finishes, until the supervisor decides the task is complete.
- An alternative is the **swarm** pattern, where agents hand off control directly to each other (peer-to-peer) rather than always returning to a central supervisor — useful when there's no natural single coordinator and agents need to decide among themselves who should act next.
- Real-world example: a claims-processing supervisor routes to a document-extraction agent first, then — based on what was extracted — either directly to a policy-lookup agent or, if red flags are present, to a fraud-investigation agent, and finally synthesizes both agents' outputs into a decision.

```python
# Sketch of a supervisor pattern (structure verified via LangGraph's official
# multi-agent supervisor tutorial; exact helper APIs may have moved — check current docs)
def supervisor(state):
    next_agent = llm_router.decide(state)          # "extractor" | "policy_lookup" | "end"
    return {"next": next_agent}

graph.add_node("supervisor", supervisor)
graph.add_node("extractor", extractor_subgraph)
graph.add_node("policy_lookup", policy_subgraph)
graph.add_conditional_edges("supervisor", lambda s: s["next"], {
    "extractor": "extractor", "policy_lookup": "policy_lookup", "end": END
})
graph.add_edge("extractor", "supervisor")
graph.add_edge("policy_lookup", "supervisor")
```

## Streaming

- LangGraph supports streaming at multiple granularities: token-level streaming from an individual LLM call, and step-level streaming of state updates as each node completes — so a UI can show "extracting documents... now checking policy..." progress rather than waiting for the whole graph to finish.
- Real-world example: a customer-facing claims-status chat surfaces each supervisor decision ("Checking your policy now") as it happens via step-level streaming, instead of a long silent wait followed by one final answer.

## LangGraph vs. LangSmith (Evaluation Is a Separate Product)

- LangGraph itself does not own agent evaluation. LangChain's evaluation/observability product is **LangSmith** — tracing, dataset-based evaluation, and regression testing for LangGraph (and other) agents live there, not inside LangGraph's core API. If a team wants LangGraph plus built-in eval tooling from the same vendor, they are pairing it with LangSmith as a separate product, not expecting LangGraph to do both.
- Real-world example: a team ships a LangGraph supervisor agent to production and uses LangSmith to capture full execution traces, then builds a regression test suite of past customer conversations in LangSmith to catch quality regressions before each new deploy.

## Why Reach for LangGraph Instead of a Plain ReAct Loop

- A hand-rolled ReAct while-loop puts your code in charge of the loop and typically parses free-text output to find tool calls — fragile if the model's output format drifts. LangGraph instead uses the LLM's native structured tool-calling API and makes the loop's structure (nodes/edges) explicit and inspectable rather than implicit in a while-loop's control flow.
- The concrete capabilities you gain by moving from a hand-rolled loop to LangGraph: durable checkpointed execution (resume after a crash), built-in human-in-the-loop interrupts, native support for cycles and branching beyond a single linear loop, subgraph composition for multi-agent hierarchies, and step-level streaming — all without hand-building that infrastructure yourself.
- The cost: another framework's abstractions and versioning to track, and genuine learning curve for state-schema/reducer semantics (see gotchas below). For a one-off two-tool agent with no persistence or human-approval requirement, a plain loop may still be the pragmatic choice — the same "start simple" principle from topic 05 applies to framework choice, not just architecture choice.

## Quick Gotchas Worth Naming in an Interview

- Without a **reducer** function on a state key, two parallel branches writing to the same key in a single step raise an `InvalidUpdateError` rather than silently merging — reducers (e.g., "append to this list" instead of "overwrite") are how you tell LangGraph how to combine concurrent writes.
- Checkpointer-backed memory (`thread_id`-scoped) gives within-thread/conversation memory; LangGraph's separate long-term memory `Store` gives cross-thread memory. Conflating the two is a common mistake — checkpointing is not the same mechanism as the long-term agent memory discussed in topic 05.
- "Checkpointing" is not "durable execution" in the strongest sense: it saves state between completed steps, not mid-step — a node that dies partway through its own work reruns from its own start on retry, not from some intermediate point inside itself.
- LangGraph does not include built-in evaluation — pair it with LangSmith (or an independent eval framework) rather than assuming eval is native.
