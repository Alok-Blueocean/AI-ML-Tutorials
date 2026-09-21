# Autonomous Agent Design — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Build a bare ReAct loop.** Without any framework, write a Python while-loop that gives an LLM 2-3 tools (e.g., a calculator and a fake "search" function returning canned text), parses its tool-call output, executes it, and feeds the observation back in, until the model returns a final answer. This is the loop every agent framework is automating for you.

2. **Add a step cap and duplicate-call guard.** Extend exercise 1 so the loop hashes (tool name, arguments) for every call, refuses to execute an identical call twice in a row, and hard-stops after N steps, returning whatever partial answer it has. Trigger both conditions deliberately with a prompt designed to confuse the model.

3. **Conceptual: workflow vs. agent.** Take a business process you know well (expense approval, onboarding, a support ticket lifecycle) and identify which parts are better as a fixed workflow and which parts genuinely need agentic autonomy. Justify each split in one sentence.

4. **Implement a Reflexion-style retry.** Take your exercise-1 agent on a task it initially gets wrong (e.g., a multi-step arithmetic word problem), have it generate a one-paragraph self-critique of the failed attempt, prepend that critique to context, and retry. Compare success rate over 10 trials with vs. without the critique step.

5. **Conceptual: Reflexion vs. plain retry.** Explain why simply re-running the same prompt with a higher temperature is not the same as Reflexion, and what specific information the self-critique adds that a naive retry doesn't.

6. **Design a plan-and-execute agent on paper.** For a "book a business trip within a $2,000 budget" task, write out the upfront plan a plan-and-execute agent would produce, then identify one scenario where a step's result should trigger re-planning rather than blind execution of the rest of the plan.

7. **Tree-of-Thought on a toy puzzle.** Pick a small constraint-satisfaction puzzle (e.g., a 4x4 Sudoku or a simple scheduling problem) and manually trace 2 levels of a thought tree: the candidate first moves, a heuristic score for each, and which branch you'd prune. Then discuss why this problem benefits from tree search over a single linear ReAct chain.

8. **Build short-term + long-term memory.** Extend your exercise-1/2 agent with (a) a working-memory scratchpad that resets every episode, and (b) a long-term memory backed by a small local vector store (FAISS/Chroma) that persists one fact per episode and is retrieved into context on future episodes. Show a case where the long-term memory changes the agent's behavior on a later run.

9. **Tool routing under scale.** Simulate having 30 tools available (stub functions are fine) and measure wrong-tool-call rate when all 30 schemas are exposed to the model at once vs. when a simple keyword/embedding router first narrows to the 5 most relevant tools. Report the difference.

10. **Conceptual: tool-call hallucination taxonomy.** For a given agent transcript (real or constructed) that calls a tool with a fabricated argument, classify the failure as one of: hallucinated tool name, hallucinated/invalid argument value, or stale argument (using a value from an earlier, now-invalid state). Propose a distinct guardrail for each category.

11. **Design an infinite-loop detector.** Specify, in pseudocode, a monitor that sits between an agent's reasoning step and tool execution, tracks the last K (tool, arguments) pairs, and raises a distinct signal for "identical repeat" vs. "cyclical repeat" (A, B, A, B, ...). Discuss what the system should do differently in each case.

12. **Single-agent vs. multi-agent design memo.** For a described business process of your choosing (e.g., insurance claims, loan underwriting, IT ticket triage), write a half-page design memo arguing for either a single well-tooled agent or a multi-agent graph, explicitly naming the tradeoffs (latency, cost, debuggability, specialization) that drove your choice.

13. **Failure-injection test.** Take any agent you've built in this exercise set and deliberately break one tool (make it return malformed JSON, or silently return `None`). Observe and document how the agent behaves — does it hallucinate a plausible-looking result, loop, or fail gracefully? Then add the minimal guardrail that fixes the worst behavior you observed.

14. **Conceptual: when NOT to add autonomy.** Describe a real or plausible scenario where a team added agentic autonomy (letting the LLM decide the next step) to a process that should have stayed a fixed workflow, and explain concretely what went wrong or could go wrong (cost, latency, unpredictability, harder debugging).

15. **End-to-end critique.** Given a described production agent that occasionally gets stuck in tool-call loops and occasionally hallucinates a customer ID, propose a prioritized list of 4 concrete interventions spanning design (memory/planning changes), guardrails (validation/caps), and observability (what you'd want logged to catch the next occurrence faster).
