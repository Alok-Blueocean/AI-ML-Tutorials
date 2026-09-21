# Tool / Function Calling and MCP — Projects

## Small: Resilient Multi-Tool Chat Agent

Build a CLI chat agent with 3-4 tools (weather, a fake stock-price lookup, a calculator, a "current time in timezone X" tool) using the Anthropic or OpenAI SDK directly, no agent framework. Implement the full request -> tool-execution -> result-injection loop by hand, run independent lookups in parallel, and add exponential-backoff retry with jitter for one tool that you deliberately make flaky (randomly times out). This proves you understand the loop mechanics and error handling at the lowest level, before any framework hides it from you.

## Medium: A Home-Grown MCP Server Plus Two Different Clients

Build an MCP server (Python or TypeScript SDK) exposing 2-3 tools and one resource over a real data source you have access to — a local SQLite database, a folder of Markdown notes, or a small internal API. Connect it to two different MCP clients: one being an actual MCP host (Claude Desktop or an IDE with MCP support) and the other a client you write yourself using the SDK's `Client` class over Streamable HTTP. This proves you can build to the spec (not just call someone else's server) and demonstrates the "build once, connect from many clients" value proposition concretely rather than by assertion.

## Medium: Tool-Call Reliability Harness

Build a wrapper layer that sits between an agent and its tools, implementing: per-tool timeout and retry policy (configurable per tool, not global), a circuit breaker that stops calling a tool after N consecutive failures and returns a clear degraded-mode result instead, structured logging of every call (tool name, arguments, latency, success/failure, retry count), and an idempotency-key mechanism for at least one non-idempotent (write) tool. Run a chaos-injection test that randomly fails 20% of calls and verify the harness recovers without duplicate side effects. This proves you can operationalize the "error handling and retries" material rather than just explain it.

## Large: Multi-Server MCP Agent With Access Control and Audit Logging

Build an agent that connects to at least three MCP servers (mix local stdio and remote Streamable HTTP — e.g., a filesystem server, a database server, and a public reference server like `fetch` or `git`), presenting a unified tool list to the LLM. Add an authorization layer that gates risky tools (writes, deletes, external sends) behind explicit human-in-the-loop confirmation, treats all tool results and tool descriptions as untrusted input with a basic prompt-injection screen, and logs every tool call with enough detail to answer "what did this agent do and why" after the fact. Include a short writeup comparing this design against the same functionality built as bespoke per-tool integrations, with a concrete estimate of the integration code you avoided by standardizing on MCP.
