# Multi-Agent Systems: LangGraph — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Build the two-node ReAct cycle.** Implement the `agent -> tools -> agent` `StateGraph` sketched in the tutorial against a real LLM with 2-3 tools. Confirm the conditional edge correctly routes to `END` when the model returns a final answer and back to `tools` otherwise.

2. **Trigger and fix an `InvalidUpdateError`.** Build a graph where two parallel branches (via `add_conditional_edges` fanning out, or two nodes both returning to the same superstep) both write to the same state key without a reducer. Reproduce the `InvalidUpdateError`, then fix it by adding an `Annotated[list, operator.add]` (or equivalent) reducer to that key, and confirm the concurrent writes now merge instead of erroring.

3. **Conceptual: explain the reducer.** In your own words, explain what a reducer function does and why LangGraph raises `InvalidUpdateError` rather than silently picking one write or overwriting, when two branches update the same key in a single step with no reducer defined.

4. **Add a `MemorySaver` checkpointer.** Extend your exercise-1 agent with `MemorySaver`, invoke it with a `thread_id`, then invoke it again later with the same `thread_id` and show the conversation state was preserved. Invoke with a new `thread_id` and show it starts fresh.

5. **Add a Postgres checkpointer and survive a simulated crash.** Swap `MemorySaver` for a Postgres-backed checkpointer, start a multi-step run, kill the process partway through (e.g., `os._exit()` inside a node after the first step completes), restart the process, and resume execution with the same `thread_id`. Confirm it resumes from the last completed step rather than the beginning.

6. **Build a supervisor with a human-in-the-loop interrupt before a destructive tool call.** Build a 3-node graph: a supervisor, a "safe" worker, and a "destructive" worker (e.g., `delete_record`). Call `interrupt()` immediately before the destructive tool executes, whatever the payload, and resume with `Command(resume=...)` carrying an approve/reject decision. Confirm a "reject" short-circuits without executing the tool.

7. **Conceptual: checkpointing vs. long-term memory.** Explain the difference between checkpointer-backed (`thread_id`-scoped) memory and LangGraph's separate long-term `Store`. For a travel-booking assistant that both (a) needs to resume a booking flow after a crash and (b) needs to remember a user's seat preference across unrelated future sessions, name which mechanism serves which requirement and why one wouldn't substitute for the other.

8. **Build a subgraph hierarchy.** Build a top-level supervisor graph with two worker nodes, each of which is itself a compiled subgraph containing its own internal extract/validate retry loop. Test each subgraph independently before wiring it into the parent, and confirm state flows correctly in and back out.

9. **Design task: supervisor vs. swarm.** Take a business process you know (loan underwriting, IT ticket triage, order fulfillment) and decide whether a supervisor/orchestrator-worker shape or a swarm/peer-to-peer hand-off shape fits better. Justify the choice in 3-4 sentences, naming the specific property of the process (single natural coordinator vs. no natural coordinator) that drove the decision.

10. **Add step-level streaming.** Instrument your supervisor graph from exercise 8 to stream step-level state updates so a UI could print "extractor started," "extractor finished, routing to policy_lookup," etc., as they happen, rather than only returning the final result.

11. **Conceptual: why checkpointing isn't full durable execution.** Explain precisely what is and isn't preserved if a node crashes partway through its own execution (not between steps). Describe what "reruns from its own start" means in this context and why that matters for writing idempotent node logic (e.g., a node that calls a non-idempotent payment API).

12. **Build a validate/retry cycle with a retry cap.** Build a graph with `extract -> validate` where a conditional edge routes back to `extract` if a confidence score is below threshold, and forward to `summarize` otherwise. Add a retry counter in state and force the graph to exit to a fallback/human-escalation node after N failed attempts instead of looping forever.

13. **Instrument with LangSmith and build a mini regression set.** Wire your exercise-6 or exercise-8 agent to LangSmith tracing. Collect 8-10 past runs (real or synthetic) into a small evaluation dataset and write one regression check that would fail if a future change broke the supervisor's routing logic.

14. **Debugging exercise.** You're handed this scenario: a production graph occasionally throws `InvalidUpdateError: At key 'notes', got multiple values for the same key at the same super-step`. Without seeing the code, list the two most likely causes and the specific state-schema/reducer fix for each.

15. **End-to-end design memo.** For a claims-processing system, sketch the full graph on paper: nodes, edges (including at least one cycle), checkpointer choice with justification, one human-in-the-loop gate with its trigger condition, and at least one subgraph. Justify each of the four choices in one or two sentences, tying each back to a concrete requirement of the claims process rather than "because LangGraph supports it."
