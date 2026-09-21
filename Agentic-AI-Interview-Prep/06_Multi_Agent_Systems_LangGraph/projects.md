# Multi-Agent Systems: LangGraph — Projects

## Small: Two-Agent Support Ticket Router

Build a supervisor node that classifies an inbound support ticket and routes it to one of two worker nodes — a billing specialist (with tools for refund lookup and invoice retrieval) or a technical specialist (with tools for account status and known-issue lookup) — then returns to the supervisor to decide whether the ticket is resolved or needs the other specialist. Add a `MemorySaver` checkpointer keyed by ticket ID so a ticket's thread can be revisited across separate customer messages. This proves you can build the core supervisor/orchestrator-worker shape end to end, including the return-to-supervisor loop, without over-engineering a two-worker problem.

## Small-Medium: Retry Loop with a Confidence Gate

Build a document-extraction agent with an `extract -> validate` cycle: the validate node scores extraction confidence, a conditional edge routes back to `extract` (with feedback on what looked wrong) below a threshold, and forward to `finalize` above it, with a hard retry cap that routes to a human-escalation node instead of looping forever. This proves you can implement the cyclic-graph capability that distinguishes LangGraph from a plain DAG workflow, including the guardrail (retry cap) a production version needs.

## Medium: Claims-Processing System with a Human-Approval Gate

Build a supervisor graph for insurance claims with three worker subgraphs — document extraction, policy lookup, and fraud investigation — each independently testable before being wired into the parent. Add an `interrupt()` immediately before any `pay_claim` tool call where the amount exceeds a configurable threshold, requiring a human reviewer's `Command(resume=...)` before the payment executes, and back the whole graph with a Postgres checkpointer so a claim's in-flight state survives a process restart. Instrument the graph with LangSmith tracing. This is the project that most directly exercises the tutorial's central example end to end: cycles, subgraphs, persistence, and a real human-in-the-loop gate on an irreversible action.

## Large: Multi-Tenant Onboarding Orchestrator with Streaming and Regression Evaluation

Build a customer-onboarding system serving multiple tenants concurrently, using a swarm (peer-to-peer hand-off) pattern among specialist agents — account provisioning, compliance/KYC checks, and integration setup — since no single agent naturally owns the full process and any agent may need to hand off back to a prior one when a downstream check fails. Stream step-level progress to a status UI, back the graph with a production-grade (Postgres) checkpointer with per-tenant `thread_id` isolation, and build a LangSmith-backed regression suite of 30-50 past onboarding runs to catch behavior regressions before each deploy. Write a short design memo justifying the swarm choice over a supervisor shape, backed by the specific property of the process (no natural coordinator, frequent need to jump back to an earlier specialist) that drove it. This project demonstrates production-grade judgment across every major LangGraph capability in this topic, not just the ability to wire nodes together.
