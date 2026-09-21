# Tool / Function Calling and MCP — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual/design questions.

1. **Define a tool schema by hand.** Write a JSON Schema tool definition (name, description, input_schema) for a `search_flights(origin, destination, date, max_price)` tool. Then generate the same schema from a Pydantic model with `model_json_schema()` and diff the two — note anywhere they disagree and why (e.g., optional-field representation, `additionalProperties`).

2. **Run the request -> execute -> inject loop manually.** Using the Anthropic or OpenAI SDK directly (no agent framework), implement the full loop for a single tool (`get_current_time(timezone)`): send the prompt, detect the tool call, execute it, inject the result, get the final answer. Print every message in the conversation array so you can see the exact shape of tool_use/tool_result (or tool_calls/role:tool) messages.

3. **Conceptual: structured intent vs execution.** Explain in your own words why "the model never executes code" is a security-relevant design choice, not just an implementation detail. Give one concrete example of what could go wrong if a tool call were auto-executed without any application-side validation step.

4. **Compare parallel vs sequential tool-call latency.** Implement 3 independent lookup tools (can be fake/mocked with `sleep(1)` each). Call all three sequentially and time it; then call them concurrently (`asyncio.gather` or `Promise.all`) and time it. Report the speedup and explain why it's not quite 3x.

5. **Design a sequential-dependency case.** Build a two-tool workflow where the second tool's arguments depend on the first tool's result (e.g., `search_product(name)` then `check_inventory(product_id)`). Explain why naively parallelizing these two calls would break, and what tool_result you'd need to fabricate if you tried to force it anyway.

6. **Implement retry-with-backoff for a flaky tool call.** Wrap a tool function that fails randomly ~40% of the time with a transient error and always fails for one specific "structural" input, with exponential backoff + jitter for the transient case and immediate failure (no retry) for the structural case. Log each attempt's delay.

7. **Conceptual: forced tool choice tradeoffs.** Explain the difference between `tool_choice: auto`, `any`/`required`, forcing one specific tool by name, and `none`. Give a production scenario where forcing a specific tool is the right call, and one where forcing `any` (model must pick something, but you don't care which) is the right call.

8. **Handle a schema-mismatch failure.** Simulate a model emitting a tool call missing a required argument. Write the tool_result / error-feedback message you'd send back, and explain what you'd change in the tool description if this kept happening across many real requests rather than being a one-off.

9. **Build a minimal MCP server.** Using the official Python or TypeScript MCP SDK, build a server exposing exactly one tool (e.g., `get_word_count(text: str) -> int`) and one resource. Run it over stdio, connect to it with the MCP Inspector (or a minimal client), list its tools, and call the tool. Note every place the SDK generated something for you that you would otherwise have hand-written (schema, protocol handshake, error formatting).

10. **Connect an MCP server to a real client.** Take the server from exercise 9 and connect it to an actual MCP host you have available (Claude Desktop, an IDE with MCP support, or a small custom client using the SDK's `Client` class) over both stdio and Streamable HTTP. Compare what changes in configuration/auth between the two transports.

11. **Conceptual: MCP vs bespoke integration.** You're asked to expose your team's internal metrics database to (a) one internal Slack bot only, versus (b) three different internal tools plus an external partner's agent. Argue which scenario justifies building an MCP server versus a bespoke in-process tool, and identify the point at which you'd revisit that decision if usage grew.

12. **Design a tool-permission boundary.** For an agent with tools `read_file`, `write_file`, `send_email`, and `run_shell_command`, design an approval/permission scheme (which calls run automatically, which require human confirmation, which are disabled entirely by default) and justify each tier.

13. **Audit an MCP server for prompt-injection risk.** Given a hypothetical third-party MCP server whose tool descriptions and tool results you don't control, list the specific places an attacker could inject instructions aimed at the model (tool description text, tool result content, resource content) and one concrete mitigation for each.

14. **Design the failure-handling policy for a multi-tool agent.** You have 5 tools with very different reliability/cost profiles (a fast internal cache lookup, a flaky third-party API, an expensive LLM-backed sub-call, a write-once external API call, a local file read). Design a per-tool timeout/retry/circuit-breaker policy table and justify why it isn't the same for all five.

15. **End-to-end critique.** A production agent occasionally sends duplicate Stripe refund requests because it retries a tool call that actually succeeded server-side but timed out on the response. Diagnose the root cause, propose a fix (think idempotency keys, and reconsider what "retry" should mean for a non-idempotent, side-effecting tool), and explain how this differs from retrying a read-only lookup.
