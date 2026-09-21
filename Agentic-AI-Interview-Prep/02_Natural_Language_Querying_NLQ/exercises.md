# NLQ / Text-to-SQL — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual questions.

1. **Build a minimal text-to-SQL prompt.** Using a small SQLite schema (e.g., a 4-5 table e-commerce schema: customers, orders, order_items, products), write a prompt template that includes the schema (table/column names + types) and generates SQL for 10 test questions. Manually check correctness.

2. **Conceptual: schema linking failure.** Explain, with a concrete example, why a schema with columns named `cust_id`, `custid`, and `customer_id` across different tables in the same database is a schema-linking trap for an LLM, even though the SQL syntax it would generate is perfectly valid.

3. **Add few-shot examples.** Add 5 hand-written (question, correct SQL) pairs specific to your schema to the prompt from exercise 1. Re-run the same 10 test questions and note which ones improved.

4. **Build an execution-feedback self-correction loop.** Wrap your text-to-SQL generator so that if the generated SQL fails to execute against SQLite, the error message is fed back to the LLM for a bounded number of retries (e.g., 3). Test against 10 deliberately tricky questions and report how many were fixed by the loop vs. never fixed.

5. **Conceptual: exact match vs execution accuracy.** Write two SQL queries that are textually very different (different join style, different column order) but return identical results for a given question. Explain why an exact-match evaluator would fail one of them.

6. **Build an execution-accuracy evaluator.** For 20 test questions with reference SQL, write an evaluator that runs both the generated and reference SQL against the same SQLite database and compares result sets (not query strings). Report execution accuracy vs. a naive string-match accuracy on the same outputs.

7. **Ambiguity handling.** Write 5 deliberately ambiguous questions against your schema (e.g., "top customers" with no metric specified). Modify your system to either ask a clarifying question or state its assumption explicitly in the answer. Show both behaviors.

8. **Security: enforce read-only execution.** Configure a SQLite (or Postgres) role/connection that can only run `SELECT` statements, and demonstrate that a generated `DELETE`/`UPDATE` statement is rejected at the database level even if the LLM produces one. Explain why this must be enforced at the connection/role level, not just via a prompt instruction.

9. **Conceptual: the confused-deputy problem.** Explain, using your own words, why an NLQ agent's database credentials being broader than the end user's actual permissions turns a prompt-injection incident into a data-exposure incident. Propose a concrete row-level-security or per-user-credential design that avoids this.

10. **Row-level security test.** Simulate a multi-tenant schema (e.g., orders table with a `tenant_id` column) and enforce row-level security so that a natural-language question from "tenant A" can never return tenant B's rows, regardless of what SQL the LLM generates. Try to break it with an adversarial question that tries to reference another tenant explicitly.

11. **Natural-language-to-Pandas.** Build a small tool that takes a natural-language question over an uploaded CSV and generates + executes Pandas code in a sandboxed subprocess (not bare `exec()`), returning the result. Test with 10 questions of varying complexity (filter, groupby, join across two CSVs).

12. **Schema drift simulation.** Rename a table or column in your schema after your few-shot examples and prompt were built. Show the text-to-SQL system silently failing or producing wrong SQL, then propose and implement a fix (schema introspection at query time, or an automated staleness check).

13. **Natural-language-to-API-call.** Given a small OpenAPI spec (a public one, or a toy one you write), build a system that maps a natural-language request to a specific API call with correctly extracted parameters, validated against the spec before execution.

14. **Non-determinism audit.** Ask your system the same question 10 times with temperature > 0. Compare the generated SQL for consistency (same join path, same aggregation). Discuss whether and how you'd constrain this for a production reporting tool where consistency matters as much as correctness.

15. **End-to-end system critique.** Given a described production NLQ tool over a 200-table enterprise warehouse with reported "SQL looks right but numbers are wrong" complaints, propose a prioritized list of 5 concrete interventions (schema linking, few-shot curation, execution feedback, evaluation, security) and justify the order.
