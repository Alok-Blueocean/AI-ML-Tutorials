# Autonomous Agent Design — Projects

## Small: Framework-Free ReAct Agent with Guardrails

Build a ReAct-style agent from scratch (no LangGraph/CrewAI/etc.) with 3-4 real tools (a web-search API, a calculator, a unit converter, or similar) wired to an LLM's native function-calling API. Add a hard step cap, a duplicate-tool-call detector, and schema validation on every tool call before execution. This proves you understand the loop that every higher-level framework is abstracting away, and that you can name and defend against the two most common agent failure modes (loops, hallucinated calls) without relying on a framework to do it for you.

## Small-Medium: Reflexion Agent on a Verifiable Task

Build an agent that attempts a task with an automatically checkable outcome (a coding kata with unit tests, or a math word-problem set with known answers), and on failure generates a self-critique that gets prepended to context for a bounded number of retries (e.g., max 3). Track success rate per attempt number across 30-50 tasks and report whether/how much the self-critique step improves eventual success rate versus plain retries with no critique. This proves you can implement and empirically evaluate a specific agentic pattern rather than just describing it.

## Medium: Agent with Working + Long-Term Memory

Build an assistant-style agent that handles multiple sessions for the "same user," using a working-memory scratchpad per session and a vector-store-backed long-term memory that persists facts/preferences across sessions (e.g., a travel-booking assistant that remembers seat/airline preferences from a prior session). Include a simple write policy (what gets written to long-term memory and when) and demonstrate at least one case where recalled long-term memory changes behavior in a later session. This proves you understand memory as two genuinely different subsystems with different read/write/eviction concerns, not just "give the agent a bigger context window."

## Large: Single-Agent vs. Multi-Agent Bake-Off on the Same Task

Pick one moderately complex business task (e.g., "triage and draft a response to an inbound support email using account history, a knowledge base, and a refund-policy lookup") and implement it two ways: once as a single well-tooled ReAct agent, once as an orchestrated multi-agent system (a supervisor plus 2-3 specialist agents, using LangGraph — see the next topic). Instrument both with the same latency/cost/step-count logging and run both against the same 30-50 test cases, scoring accuracy and failure modes for each. Write up a decision memo recommending one architecture for production, backed by your measured numbers rather than intuition. This is the project that best demonstrates senior-level judgment: knowing when the "more sophisticated" architecture is not actually the better one.
