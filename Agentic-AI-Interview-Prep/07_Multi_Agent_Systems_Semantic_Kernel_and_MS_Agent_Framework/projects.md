# Multi-Agent Systems: Semantic Kernel and Microsoft Agent Framework — Projects

## Small: Semantic-Kernel-to-Agent-Framework Bridge Demo

Take a small existing (or purpose-built) Semantic Kernel plugin with 2-3 `@kernel_function`-decorated methods and a `ChatCompletionAgent` using it. Build a second, MAF-based agent that reuses the exact same `KernelFunction`s via `.as_agent_framework_tool()`, without rewriting the underlying plugin code, and confirm both agents produce equivalent tool calls on the same prompts. This proves you understand the concrete, documented migration path Microsoft provides — bridging existing code incrementally — rather than treating "migrate to MAF" as an abstract rewrite mandate.

## Small-Medium: Support-Bot Handoff Between Two Specialists

Build a two-agent MAF Handoff orchestration: a general-support agent that detects billing-related intent and hands off to a billing-specialist agent with its own tools (refund lookup, invoice retrieval). Add a case where the billing specialist determines the issue is actually technical and hands back. This proves you can build the same peer-to-peer coordination shape as LangGraph's swarm pattern (topic 06), using MAF's named Handoff orchestration instead of hand-composed graph edges.

## Medium: Claims-Processing System with an Approval Gate, Rebuilt on Microsoft Agent Framework

Rebuild the claims-processing example used throughout topic 06 (intake, extraction, policy lookup, fraud investigation, decision) as an MAF Sequential orchestration with a Handoff to a fraud-specialist agent when extraction flags red flags, and an approval-required tool gating any `pay_claim` call above a configurable threshold. Write a short comparison memo against a LangGraph implementation of the same system: what was simpler to express with MAF's named orchestrations versus what LangGraph's lower-level graph primitives made easier to customize. This project directly demonstrates the "same problem, two frameworks" comparison a senior interview is likely to probe.

## Large: Incremental Migration of a Production-Shaped Semantic Kernel System

Build a moderately complex Semantic Kernel system first (a multi-plugin `ChatCompletionAgent` handling, e.g., IT ticket triage with account-lookup, knowledge-base-search, and escalation plugins), then execute — and document — an incremental migration to Microsoft Agent Framework: bridge existing plugins with `.as_agent_framework_tool()` first, replace the single agent with an MAF Sequential or Magentic orchestration second, and only rewrite individual plugin functions as plain MAF tools last, once each is confirmed working through the bridge. Track what broke at each stage and how you diagnosed it. This is the project that most closely simulates the real, most likely 2026 task on this topic: not building a new MAF system from a blank slate, but migrating an existing Semantic Kernel investment without a risky big-bang rewrite.
