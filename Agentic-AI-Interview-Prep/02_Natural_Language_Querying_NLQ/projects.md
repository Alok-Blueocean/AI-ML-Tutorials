# NLQ / Text-to-SQL — Projects

## Small: Text-to-SQL Agent with Self-Correction over a SQLite Schema

Build a natural-language query tool over a small SQLite database (5-8 tables — an e-commerce or CRM-style schema works well). Include schema-aware prompting, 5-10 curated few-shot examples, and an execution-feedback loop that retries on SQL errors up to a bounded number of attempts. Evaluate on 20 hand-written test questions using execution accuracy, not string match. This proves you understand the core architecture and the difference between exact-match and execution-accuracy evaluation.

## Medium: Secure Multi-Tenant NLQ Tool

Build an NLQ tool over a multi-tenant schema (a `tenant_id` or `org_id` column threading through most tables) with enforced row-level security and a read-only database role, so that no natural-language question — however phrased, including adversarial attempts to reference another tenant's data — can return data outside the requesting user's scope. Add an audit log of every generated SQL statement and its executing user. This proves you treat LLM-generated queries as untrusted input requiring database-level security controls, not prompt-level politeness.

## Medium: Ambiguity-Aware Analytics Copilot

Build an NLQ tool that detects ambiguous questions against a real schema (e.g., a public open dataset like a retail or transit dataset) and either asks a clarifying follow-up or explicitly states its interpretation assumption in every answer, with the generated SQL always shown to the user for verification. Evaluate on a test set that deliberately includes ambiguous, underspecified, and unanswerable-given-the-schema questions, measuring how often the system correctly flags ambiguity instead of confidently guessing. This proves you understand that a syntactically correct query answering the wrong question is a worse failure than a visible refusal.

## Large: NLQ Layer over Multiple Data Sources with Unified Evaluation

Build a natural-language querying layer that routes a question to the right backend — SQL for a relational warehouse, Pandas for an uploaded CSV, or a REST API call for a live service — based on where the relevant data lives, with a shared validation/execution-feedback framework across all three. Include an evaluation harness that measures execution accuracy per source type and an end-to-end "did the final natural-language answer correctly reflect the executed result" faithfulness check. This proves you can generalize the text-to-query pattern beyond SQL and design a system that scales across heterogeneous real-world data sources rather than a single hardcoded database.
