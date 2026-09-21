# Tool / Function Calling and MCP — References

All links below were fetched and content-verified this session unless explicitly marked otherwise. MCP is a fast-moving spec (see notes below) — re-verify version-specific claims before an interview if meaningful time has passed since this was written (September 2026).

## Papers

- No single academic paper defines MCP — it's an engineering protocol/spec, not a research paper. The closest "paper-equivalent" background is the ReAct paper (Yao et al., 2022, https://arxiv.org/abs/2210.03629), which underlies the reasoning-then-tool-call loop that all function calling sits inside; see the `05_Autonomous_Agent_Design` topic for the full treatment (not re-verified again here — already verified in that topic's session).

## Official Docs

- Model Context Protocol — official site and docs hub. https://modelcontextprotocol.io
- MCP Specification, current version 2026-07-28 (confirmed via the versioning page as the "current" — not draft, not deprecated — revision as of this session). https://modelcontextprotocol.io/specification/2026-07-28
- MCP Specification versioning policy page (explains Draft/Current/Final revision states and version negotiation). https://modelcontextprotocol.io/specification/versioning
- MCP architecture overview (hosts/clients/servers, data layer vs. transport layer, primitives). https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture
- MCP introduction / "what is MCP" page. https://modelcontextprotocol.io/introduction
- Anthropic, "Introducing the Model Context Protocol" — original announcement, November 25, 2024. https://www.anthropic.com/news/model-context-protocol
- Anthropic, "Writing effective tools for AI agents" — engineering guidance on tool design (consolidation, context-efficient responses, description quality), published September 2025. https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic API docs — defining tools (JSON Schema, `input_examples`, `strict`, tool_choice options: auto/any/tool/none). https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
- Anthropic API docs — handling tool calls (tool_use/tool_result formatting rules, `is_error` field, invalid-tool-call retry behavior). https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls
- Anthropic API docs — parallel tool use (execution-order semantics are left to the caller, formatting rules for multiple tool_result blocks). https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use
- Anthropic API docs — remote MCP servers via the Messages API MCP connector. https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers (redirected from docs.anthropic.com; final URL verified)
- OpenAI — function/tool calling guide (schema definition, `parallel_tool_calls`, `tool_choice` auto/required/specific/`allowed_tools`, `strict` mode requirements). https://developers.openai.com/api/docs/guides/function-calling (redirected from platform.openai.com/docs/guides/function-calling; final URL verified)
- OpenAI — MCP support in the Responses API / Agents SDK / ChatGPT connectors. https://developers.openai.com/api/docs/mcp/
- Pydantic — JSON Schema generation (`model_json_schema()`, `TypeAdapter.json_schema()`). https://pydantic.dev/docs/validation/latest/concepts/json_schema/ (redirected from docs.pydantic.dev; final URL verified)

## GitHub Repos

- `modelcontextprotocol/python-sdk` — official Python SDK, currently on v2 (major rework for the 2026-07-28 spec; server class is `MCPServer`, replacing the v1 `FastMCP` name — v1 still maintained on a separate branch for existing users). https://github.com/modelcontextprotocol/python-sdk
- `modelcontextprotocol/typescript-sdk` — official TypeScript SDK, also on v2, split into separate `@modelcontextprotocol/server` and `@modelcontextprotocol/client` npm packages (v1 shipped a single combined SDK package). https://github.com/modelcontextprotocol/typescript-sdk
- `modelcontextprotocol/servers` — official reference server implementations (filesystem, git, fetch, memory, sequential-thinking, time, "everything" demo server) plus links to community servers. https://github.com/modelcontextprotocol/servers
- `modelcontextprotocol/inspector` — the MCP Inspector, the standard local dev tool for testing a server's tools/resources/prompts before wiring it into a real client. https://github.com/modelcontextprotocol/inspector

## Articles / Interview Prep

- DataCamp, "MCP Interview Questions: Beginner to Advanced (2026)" — broad coverage of architecture (host/client/server), tools vs. resources, OAuth 2.1/security, server-development pitfalls (e.g., stdout corruption), and system-design scenarios. Note: at the time of this fetch the article still referenced "2025-11-25" as the current stable spec rather than 2026-07-28 — a concrete example of how quickly secondary MCP content goes stale; prefer the official spec page for version-specific facts. https://www.datacamp.com/blog/mcp-interview-questions
- Descope, "MCP vs. Function Calling: How They Differ and Which to Use" — comparison across architecture, security/credential isolation, portability across providers, and production scalability. https://www.descope.com/blog/post/mcp-vs-function-calling
- Model Context Protocol Blog, "One Year of MCP: November 2025 Spec Release" — adoption milestones (registry growth, contributor counts, major-vendor adoption) at the one-year mark. https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/
- AWS Well-Architected Framework, Agentic AI Lens — "Develop fallback behavior and error handling for tool invocations" (retry/circuit-breaker/fallback-chain guidance referenced in the error-handling section of this topic). https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentops04-bp03.html (not URL-verified this session — surfaced via search only; content described matches the topic but the page itself was not fetched and checked directly)

## Notes on What to Prioritize

Interview signal for this topic clusters around three things: (1) being able to precisely state that the model emits structured intent and never executes anything itself — this underlies almost every security/permissions follow-up question; (2) correctly explaining that MCP standardizes the tool/resource *interface* while function calling remains the underlying mechanism the model uses to invoke it (interviewers specifically probe for candidates conflating the two or claiming MCP "replaces" function calling); (3) being able to reason concretely about when MCP is worth the overhead versus a bespoke integration, rather than reciting "MCP is the USB-C of AI" without a decision rule. Because MCP's spec and SDKs have changed materially within the last year (stateless protocol core, deprecated sampling/logging primitives, major SDK version bumps), it is reasonable and credible in an interview to say "the details may have moved since I last checked" rather than asserting stale specifics with false confidence.
