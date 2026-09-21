# NLQ / Text-to-SQL — Scenario-Based Q&A

**Situation:** Your text-to-SQL tool generates a query that executes successfully and returns a plausible-looking revenue number, but it turns out to be aggregating at the wrong grain (per-order-line instead of per-order), silently double-counting revenue. What would you do and why?

Model answer: This is the most dangerous class of NLQ failure — syntactically valid, semantically wrong, and nothing about the output looks broken. First, build execution-accuracy-based regression tests around known-correct aggregation grains for common business metrics (revenue, count of orders, active users) so this class of error is caught before shipping, not after a stakeholder notices a number is off. Add few-shot examples specifically covering the correct join/aggregation grain for this schema's most common metrics, since grain errors are usually a schema-linking problem, not a SQL-syntax problem. Also consider surfacing the generated SQL alongside the answer so a data-literate user can sanity-check the join before trusting the number.

---

**Situation:** A user asks "show me our top customers" and the system confidently returns a ranked list — but "top" was never defined (revenue? order count? lifetime value?), and the business had a different metric in mind than the one the model picked. What would you do and why?

Model answer: Treat this as an ambiguity-handling gap, not a SQL-generation bug. Either have the system detect the ambiguity and ask a clarifying question before executing, or — for a lower-friction UX — make a documented default assumption and state it explicitly in the answer ("ranked by total revenue over the selected period"), so the user can catch a mismatch immediately rather than silently trusting a wrong interpretation. Track which default assumptions get corrected most often in practice and use that to refine the system's default choices or trigger clarification more proactively for those specific ambiguous terms.

---

**Situation:** A security review flags that your NLQ agent's database connection has full read-write access, and someone points out that a cleverly crafted natural-language input (or a poisoned document the agent reads) could trigger a destructive query. What would you do and why?

Model answer: Fix this at the database-grant level immediately, not the prompt level — create a dedicated read-only database role (no INSERT/UPDATE/DELETE/DROP grants) and point the agent's connection at that role, since prompt-level instructions like "never write to the database" are not a security boundary and can be bypassed by injected instructions. Add statement timeouts and row-limit caps to prevent runaway or resource-exhausting queries, allow-list the queryable tables/schemas, and treat the agent's credentials as a "confused deputy" risk — never grant it more access than the least-privileged human user it serves. Log every generated and executed query for audit.

---

**Situation:** Your NLQ system passed all internal QA with 90%+ accuracy measured by exact string match against reference SQL, but users are reporting it "feels wrong more often than that." What would you do and why?

Model answer: Suspect the evaluation metric before suspecting the model. Exact-match scoring penalizes any stylistically different but functionally correct SQL (different join order, aliasing, CTE vs. subquery), so it both undercounts real failures that happen to string-match a reference by luck on trivial queries, and overstates confidence broadly. Re-evaluate using execution accuracy — run generated and reference SQL against the actual database and compare result sets — which is the modern standard precisely because it credits any correct formulation. Expect the two numbers to diverge meaningfully, and use the execution-accuracy failures (the real ones) to prioritize fixes.

---

**Situation:** Three months after launch, a table your NLQ system depends on gets renamed as part of a data-warehouse migration, and nobody updates the NLQ tool's prompt or few-shot examples. What would you do and why?

Model answer: This is a schema-drift failure, and it's structural, not a one-off bug — any static, hand-maintained schema description or few-shot set will eventually go stale as the underlying warehouse evolves. The fix is process, not a one-time patch: pull the schema description dynamically via introspection at query time (or on a scheduled refresh) rather than hardcoding it, and add a canary/health check that periodically runs a fixed set of known-good questions and alerts if execution accuracy drops, so schema drift is caught by monitoring rather than by a user complaint.

---

**Situation:** Leadership wants to let external customers (not just internal analysts) type natural-language questions against their own account data through a support portal, and asks if that's safe. What would you do and why?

Model answer: The core requirement is airtight row-level security enforced at the database layer, not the application layer — every generated query must run under a database role/session scoped to that specific customer's tenant ID, ideally via a database-native RLS policy, so that no phrasing of a question (including deliberately adversarial attempts to reference another customer's ID) can return another tenant's rows. Pair this with a read-only role, row-count/time limits, and comprehensive audit logging. Pilot with a small customer cohort and adversarially test the RLS boundary (attempt cross-tenant references) before any broader rollout — treat this as a security launch, not a feature launch.

---

**Situation:** A data analyst complains that asking the exact same question twice in the same session sometimes returns SQL with a different join path or slightly different rounding, and it's undermining trust in the tool even though both answers are technically correct. What would you do and why?

Model answer: Acknowledge that non-determinism is a real UX/trust problem even when each individual answer is correct, especially for a reporting tool where users expect a fixed metric to always compute the same way. Reduce sampling temperature toward zero for SQL generation specifically (as opposed to any natural-language explanation text, which can stay more flexible), and consider caching the generated SQL per canonicalized question so repeat questions reliably re-execute the same query rather than regenerating it. If different join paths are both "correct," pick and enforce one canonical path per metric via few-shot examples or a metrics-layer abstraction rather than leaving it to the model's discretion each time.

---

**Situation:** You're asked to add "ask your data" functionality over a set of uploaded CSVs (not a live database), and someone suggests just having the LLM generate Python/Pandas code and running it directly. What would you do and why?

Model answer: Treat generated Pandas/Python code exactly like generated SQL from a security standpoint — it's arbitrary code execution, not a safely bounded query language, so it must run in a sandboxed environment (subprocess with resource/time limits, no filesystem or network access, no arbitrary imports) rather than a bare `exec()` in the main process. Functionally, apply the same evaluation discipline as text-to-SQL: measure whether the executed result is correct (execution-accuracy equivalent), not just whether the code runs without erroring, since code that runs cleanly but computes the wrong aggregation is the Pandas equivalent of a semantically-wrong-but-valid SQL query.
