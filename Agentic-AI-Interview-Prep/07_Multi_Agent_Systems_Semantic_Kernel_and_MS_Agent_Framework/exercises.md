# Multi-Agent Systems: Semantic Kernel and Microsoft Agent Framework — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Build a minimal Microsoft Agent Framework agent.** Using `agent-framework`, build a single agent from a chat client (e.g., `OpenAIChatClient().as_agent(...)`) with one plain-function tool, and run it non-streaming with `agent.run(...)`. Confirm the function's docstring and type hints are enough for the model to call it correctly without any decorator.

2. **Build the equivalent agent in Semantic Kernel, then bridge it.** Build the same single-tool agent in SK (`@kernel_function`, a `Plugin`, a `Kernel`, a `ChatCompletionAgent`). Then, instead of rewriting the tool, use `.as_agent_framework_tool()` to bridge the existing `KernelFunction` into a new MAF agent, and confirm both the SK agent and the bridged MAF agent produce equivalent tool calls.

3. **Conceptual: why did the `Kernel` requirement disappear?** Explain what role the `Kernel` object played in every Semantic Kernel agent (DI container, service registry, plugin registry) and what capability, if any, is lost by MAF removing that mandatory dependency in favor of building agents directly from a chat client.

4. **Build a Sequential orchestration.** Build a two-agent Sequential workflow — a draft-writer agent followed by an editor agent — where the editor's input is the writer's full output. Confirm the orchestration enforces the fixed order rather than letting the editor run first.

5. **Build a Handoff orchestration.** Build a general-support agent and a billing-specialist agent, with the general-support agent handing off control to the billing specialist when it detects a billing-related intent. Test a case where the hand-off correctly triggers and a case where it correctly doesn't.

6. **Build a Group Chat orchestration.** Build a three-agent Group Chat orchestration where the agents critique and revise a shared proposal, capped at a fixed number of turns. Log the full conversation and confirm the turn cap is actually enforced (it doesn't run forever).

7. **Add a human-in-the-loop approval gate.** Extend your Sequential orchestration from exercise 4 (or build a new one) with an approval-required tool gating a high-value/destructive action, mirroring the LangGraph `interrupt()` exercise from topic 06. Demonstrate the workflow pausing for review and only proceeding after approval.

8. **Conceptual: Handoff vs. swarm.** Compare MAF's Handoff orchestration to LangGraph's swarm/peer-to-peer pattern from topic 06. Identify what's the same conceptually (no central coordinator, agents pass control directly) and name one concrete implementation-level difference (e.g., named built-in orchestration vs. hand-composed graph edges).

9. **Conceptual: Group Chat / Magentic vs. supervisor.** Compare MAF's Group Chat and Magentic orchestrations to LangGraph's supervisor/orchestrator-worker pattern from topic 06. Explain which MAF pattern is the closer analog to a fixed-order supervisor and which is closer to an open-ended coordinator, and why.

10. **Migration exercise.** Take a small existing (or constructed) Semantic Kernel `ChatCompletionAgent` with two plugins and manually rewrite it as an MAF agent. Produce a line-by-line list of what changed (imports, tool registration, agent creation, session vs. thread, invoke vs. run) and why each change was necessary, not just cosmetic.

11. **Conceptual: what the planners used to do.** Explain concretely what Semantic Kernel's Stepwise and Handlebars planners did (prompting the model to choose functions before native function-calling existed) and what specific mechanism replaced them. Explain why the docs describe this as function calling being "more powerful and easier to use," not just a like-for-like swap.

12. **Build a Magentic orchestration.** Build a manager agent coordinating two specialist agents on an open-ended research task (no fixed step order known upfront). Contrast its behavior with your fixed-order Sequential orchestration from exercise 4 — specifically, show a case where the manager's dynamic coordination handles a scenario the fixed Sequential order couldn't.

13. **Design memo: should we migrate this quarter?** A stakeholder asks whether a stable, unmodified-in-a-year production Semantic Kernel app should be migrated to MAF this quarter. Write a half-page risk/benefit memo. Your recommendation should hinge on whether there's an active pain point (a needed capability SK lacks, a support/security concern) rather than "MAF is newer."

14. **Streaming exercise.** Implement `agent.run(..., stream=True)` on an MAF agent, collect the `AgentResponseUpdate` stream, and reconstitute the full response using `AgentResponse.from_updates(...)` (or the equivalent generator-based helper). Confirm the reconstituted text matches what a non-streaming `run()` call on the same input would produce.

15. **Interview drill: catch yourself defaulting to legacy answers.** Given the prompt "design a multi-agent Microsoft-stack system for automated expense report review," answer out loud, then check your own answer against two things: did you name Microsoft Agent Framework as the primary framework, and did you mention Semantic Kernel/AutoGen only as legacy context (if at all) rather than as your recommended stack? Redo the answer if either check fails.
