# Natural Language Querying (NLQ) / Text-to-SQL — Condensed Study Notes

## What NLQ Systems Do

- An NLQ system turns a natural-language question ("how many orders shipped late last quarter?") into an executable query against a structured data source — most commonly SQL, but also Pandas/DataFrame operations or API calls — then executes it and returns a grounded, factual answer, as opposed to an LLM guessing an answer from memory.
- Real-world example: a sales-ops team asks a Slack bot "what's our win rate in EMEA this quarter" and gets a number computed from a live query against the CRM database, not an LLM-estimated guess.

## Core Architecture

```
NL question -> schema linking -> prompt construction (schema + few-shot examples) ->
LLM generates SQL -> validate (syntax + safety) -> execute (read-only role) ->
   if execution error or empty/implausible result -> feed error back to LLM -> regenerate SQL (loop, bounded retries)
   else -> format result -> natural-language answer (optionally with the SQL shown for transparency)
```

- **Schema linking**: mapping words in the question to actual table/column names in the database (e.g., "customers who churned" → `customers.status = 'churned'`). This is the step most text-to-SQL systems get wrong on real, wide, messy schemas with hundreds of tables and ambiguous column names.
- **Few-shot prompting for SQL**: including a handful of (question, correct SQL) example pairs relevant to the target schema in the prompt measurably improves accuracy over zero-shot, especially for schema-specific quirks (naming conventions, which join path is correct).
- **Execution-feedback self-correction**: when generated SQL fails to execute (syntax error) or returns something implausible (e.g., an empty result for a question that should have data), the error message or result is fed back to the LLM to regenerate — a self-correction loop grounded in real execution, not just the model re-reading its own draft. This execution-grounded approach (e.g., agent-style refiners that actually run the SQL and iterate on real errors) has proven more effective than pure "let the model silently re-check its own work" self-reflection.

```python
# Simplified execution-feedback loop
sql = generate_sql(question, schema, few_shot_examples)
for attempt in range(max_retries):
    result, error = execute_readonly(sql)
    if error is None:
        break
    sql = generate_sql(question, schema, few_shot_examples, prior_sql=sql, error=error)
answer = format_answer(question, result)
```

## Handling Ambiguous Questions

- Natural language is often genuinely ambiguous against a schema: "top customers" could mean by revenue, by order count, or by lifetime value; "last month" could mean calendar month or a rolling 30 days.
- Options: ask a clarifying follow-up question when confidence is low, make a documented default assumption and state it explicitly in the answer ("assuming 'top' means by total revenue"), or surface the generated SQL to the user so they can catch a wrong interpretation before trusting the number.
- Real-world example: a finance NLQ tool interpreted "revenue this year" as calendar year when the company's fiscal year starts in April — a syntactically perfect, semantically wrong query that produced a confidently wrong number nobody caught until quarter-end reconciliation.

## Security Concerns

- **SQL injection risk via LLM-generated queries**: the LLM itself can be manipulated (via prompt injection embedded in user input or ingested content) into generating destructive or data-exfiltrating SQL, and even without malicious intent, the LLM can generate valid-but-dangerous SQL (a `DELETE` or unbounded `UPDATE`) if allowed to.
- **Mitigations**: run generated queries through a **read-only database role** (no `INSERT`/`UPDATE`/`DELETE`/`DROP` privileges at the database-grant level, not just prompt instructions — prompt-level restrictions are not a security boundary), enforce **row-level security (RLS)** so a user can never see data outside their permission scope regardless of what SQL the LLM writes, allow-list which tables/schemas are queryable, add statement timeouts and row-limit caps to prevent runaway queries, and never let the agent's database credentials have more privilege than the least-privileged human user it serves on behalf of (the "confused deputy" problem — the agent holds credentials the end user doesn't).
- Real-world example: a filed security report against an open-source LLM-SQL chatbot documented a prompt-injection path leading to resource-exhausting queries — a SQL-based denial-of-service triggered purely through crafted natural-language input, not a traditional exploit.

## Evaluating NLQ Systems

- **Exact match (EM)**: does the generated SQL string match a reference query exactly (often after normalization)? Brittle — many different, equally correct SQL formulations (different join order, aliasing, CTE vs subquery) produce the same result but fail exact match. Now considered a weak metric on its own.
- **Execution accuracy (EX)**: does executing the generated SQL against the actual database produce the same result set as executing the reference SQL? The modern standard, because it credits any correct formulation, not just one canonical phrasing.
- EX itself has known gaps: two different (and differently correct or incorrect) queries can coincidentally return the same result set, especially on small test databases — driving further refinement like soft-F1 (partial row-overlap credit) and, increasingly, LLM-as-judge evaluation of both the SQL logic and the final natural-language answer.
- Real-world example: an internal benchmark using strict exact-match scoring reported "60% accuracy," but a switch to execution-accuracy scoring on the same model's outputs showed 85% — most of the "failures" were stylistically different but functionally correct SQL.

## NLQ over Non-SQL Sources

- **Natural-language-to-Pandas/DataFrame**: instead of generating SQL, the LLM generates and executes Python/Pandas code against an in-memory DataFrame — common for ad hoc data analysis over CSV/Excel exports rather than a live database. Carries similar risks (arbitrary code execution) requiring a sandboxed execution environment, not a bare `exec()`.
- **Natural-language-to-API-call**: the LLM maps a question to a structured API call (parameters, endpoint) instead of SQL — the underlying pattern is the same (schema/spec linking, structured generation, validation, execution feedback) but the "schema" is an API spec (e.g., an OpenAPI definition) rather than a database schema.
- Real-world example: a data-analyst copilot tool lets a non-technical user ask "show me monthly active users by region for the last 6 months" over an uploaded CSV; the system generates and sandboxes a Pandas snippet rather than requiring a live database connection at all.

## Full Prompt Template Example

A realistic text-to-SQL prompt includes the schema (with types and, ideally, sample values or column descriptions for ambiguous names), a few schema-specific examples, and explicit output-format constraints:

```
SYSTEM:
You are a SQL generator for a PostgreSQL analytics database. Only generate SELECT statements.
Never generate INSERT, UPDATE, DELETE, DROP, or ALTER statements under any circumstances.
Return only the SQL query, no explanation.

SCHEMA:
customers(id INT, name TEXT, region TEXT, signup_date DATE)
orders(id INT, customer_id INT REFERENCES customers(id), order_date DATE, total_amount NUMERIC)

EXAMPLES:
Q: How many customers signed up in 2023?
SQL: SELECT COUNT(*) FROM customers WHERE EXTRACT(YEAR FROM signup_date) = 2023;

Q: What is the total revenue by region?
SQL: SELECT c.region, SUM(o.total_amount) AS revenue
     FROM orders o JOIN customers c ON o.customer_id = c.id
     GROUP BY c.region;

QUESTION:
{user_question}
SQL:
```

- Note the explicit "only SELECT statements" instruction — this is a defense-in-depth layer, not the actual security boundary (the database role enforces that), but it reduces how often a destructive statement is even generated in the first place.
- Real-world example: adding just 3-5 schema-specific few-shot examples covering the most common join patterns in a company's actual analytics schema improved execution accuracy substantially more than an equivalent amount of effort spent on more elaborate zero-shot instructions, because the examples resolve schema-specific ambiguity (which of three similarly-named date columns to use) that no amount of generic prompting can.

## Multi-Turn / Conversational NLQ

- Real usage is rarely one-shot: a user asks a question, then a follow-up that implicitly refers to the previous query ("now break that down by region", "what about just last month"). Supporting this requires carrying forward the previous question's resolved SQL (or its structured intent) as conversational context, not just the raw chat history text, so follow-ups can be resolved as modifications to a known-good prior query rather than reinterpreted from scratch.
- Real-world example: a BI copilot handling "show me sales by product" followed by "just for Europe" needs to inject the previous query's table/column resolution into the follow-up's prompt context, or it may regenerate an entirely different (and possibly wrong) query structure instead of simply adding a `WHERE region = 'Europe'` clause to the known-good prior one.

## Testing and Regression Suites for NLQ

- Because schema drift and prompt/model changes can silently break previously-working queries, a maintained regression suite of representative questions (covering common aggregations, joins, date-filtering patterns, and known edge cases) run against execution accuracy on every deployment is the standard safety net, analogous to a unit test suite for a traditional codebase.
- Real-world example: a team's regression suite caught that a routine LLM provider model version upgrade changed how the model handled implicit date-range interpretation ("this quarter" resolving differently), before it reached production users, because the suite included several date-relative questions with known-correct execution results.

## Schema Linking at Scale

- On a toy 5-table schema, schema linking is nearly trivial. On a real enterprise warehouse with hundreds of tables, it becomes the dominant source of error: the relevant tables for a given question have to be retrieved/narrowed before the LLM even sees them, because dumping an entire 300-table schema into a prompt both blows the context budget and dilutes the model's attention across mostly-irrelevant tables.
- A common production pattern is RAG-over-schema: embed table/column descriptions (and sample values) and retrieve only the schema subset relevant to the question before generating SQL, rather than statically including the full schema in every prompt. This makes schema linking itself a retrieval problem, with the same failure modes (missing the right table, retrieving irrelevant ones) discussed in this kit's RAG topic.
- Real-world example: a warehouse with `customer`, `customers_v2`, and `dim_customer` tables (created at different points by different teams) caused an LLM to silently pick the wrong one for a given question; the fix was maintaining a curated, human-reviewed table-description catalog used for schema retrieval, not just relying on raw information-schema metadata.

## Common Production Failure Modes

- **Syntactically valid, semantically wrong SQL**: the query runs without error and returns a plausible-looking number that is simply answering the wrong question (wrong join, wrong filter, wrong aggregation grain) — the most dangerous class of failure because nothing about it looks broken.
- **Schema drift**: a table is renamed, a column's meaning changes, or a new table is added, and the system's schema-linking/few-shot examples go stale, producing wrong or broken queries with no explicit error.
- **Overprivileged execution**: the agent's database role has broader access than intended, turning a prompt-injection or model-error incident into a real data-exposure or data-loss incident instead of a contained one.
- **Non-determinism on repeated identical questions**: the same question asked twice can get different SQL (different join path, different rounding) if the LLM isn't constrained, undermining trust even when both are technically correct.

## Semantic Layers and Metrics Consistency

- A recurring enterprise pattern is pairing NLQ with a **semantic layer** (a metrics/business-logic abstraction sitting between raw tables and the query interface — e.g., a defined "revenue" metric with agreed-upon filters and joins baked in) rather than letting the LLM freely construct aggregation logic from raw tables every time. This ensures "revenue" means the same thing no matter how the question is phrased or which analyst asks it.
- Real-world example: without a semantic layer, two differently-phrased questions ("total sales" vs. "revenue") independently generated SQL with different tax-inclusion assumptions, producing two different numbers for what the business considered the same metric; routing both through a shared semantic-layer definition of "revenue" eliminated the inconsistency entirely.

## Numeric and Date Normalization

- Natural language dates and numbers are a persistent source of subtle bugs: "last quarter," "this month," "Q3," and relative terms like "recently" all need consistent, explicitly-defined resolution logic (ideally handled deterministically outside the LLM, e.g., resolving "last quarter" to exact date boundaries in code before or alongside SQL generation, rather than trusting the LLM's date arithmetic on every call).
- Real-world example: an LLM asked for "sales in the last 7 days" computed the boundary inconsistently across repeated calls (sometimes including today, sometimes not) because the date arithmetic was left to free-form model generation instead of being resolved deterministically and then injected into the schema-linking/prompt step as a fixed date range.

## Monitoring NLQ in Production

- Live signals worth tracking: execution-error rate (queries that fail to run at all), empty-result rate (a proxy for "probably the wrong table/filter" when the true answer likely has data), query latency, and a sampled human-review process for a subset of generated queries against a gold-standard rubric, similar to the RAG monitoring practices in this kit's RAG topic.
- Real-world example: a spike in empty-result rate after a scheduled warehouse migration was the first signal (well before user complaints) that a table rename had broken the system's schema-linking assumptions — treated as an actionable alert, not just a dashboard number nobody watches.

## Single-Shot vs Agentic NLQ Architectures

- **Single-shot**: one LLM call generates SQL directly from the question and schema, with at most a lightweight validation pass. Fast and cheap, works well for simple, well-scoped schemas and questions.
- **Agentic**: the system plans (which tables are relevant, whether the question needs decomposing into sub-questions), retrieves schema context, generates SQL, executes, observes the result, and can loop back to regenerate or ask a clarifying question — closer to a ReAct-style loop than a single pass. Handles complex multi-table, multi-step, or ambiguous questions better, at the cost of more latency and more places something can go wrong.
- Real-world example: a simple "how many orders today" dashboard widget uses single-shot generation because the question space is narrow and speed matters; a general-purpose "ask anything about our data" enterprise tool uses an agentic architecture because the question space is unbounded and getting the wrong answer confidently is worse than taking two extra seconds to verify.

## Natural-Language-to-Visualization

- A closely related capability: instead of (or in addition to) returning a number or table, the system generates a chart specification (which chart type, which fields on which axes) from the natural-language question and the query result — "show me a trend of X over time" implies a line chart; "compare X across regions" implies a bar chart. This adds a second structured-generation problem (chart spec, not just SQL) on top of the query itself, with its own correctness criteria (did it pick a sensible chart type and axis mapping for this data shape).
- Real-world example: a BI copilot answering "how has churn changed over the last year" needs to both generate the correct SQL (time-series query on churn) and pick a line chart with time on the x-axis, rather than defaulting to a table or a bar chart that would technically show the same numbers but obscure the trend the user actually asked about.

## Whiteboarding an NLQ System (Interview Framing)

When asked to design a text-to-SQL/NLQ system in an interview, a strong answer covers, in order: (1) the target schema's scale and how schema linking will work at that scale (full schema in prompt vs. RAG-over-schema), (2) few-shot example curation strategy, (3) the validation and execution-feedback loop design with bounded retries, (4) the security model (read-only role, RLS, allow-listed tables, statement timeouts) stated explicitly as a database-level control, (5) how ambiguity is detected and handled, (6) the evaluation approach (execution accuracy as the headline metric, with a labeled test set spanning easy/ambiguous/unanswerable questions), and (7) production monitoring (execution-error rate, empty-result rate, schema-drift detection). Naming the confused-deputy security risk and execution accuracy vs. exact match unprompted signals real production experience.

## Handling Multi-Table Joins: A Worked Example

Consider the question "which customers placed an order but never left a review" against a schema with `customers`, `orders`, and `reviews` tables. This requires the model to correctly infer an anti-join pattern (customers present in orders but absent from reviews), which is a substantially harder schema-linking and reasoning task than a single-table filter:

```sql
SELECT DISTINCT c.id, c.name
FROM customers c
JOIN orders o ON o.customer_id = c.id
LEFT JOIN reviews r ON r.customer_id = c.id
WHERE r.id IS NULL;
```

- This pattern (anti-joins, "customers who did X but not Y") is one of the most common places models default to a simpler but wrong query — e.g., silently dropping the anti-join condition and just returning all customers who ordered — because the surface phrasing doesn't obviously map to a `LEFT JOIN ... WHERE ... IS NULL` pattern the way a direct filter does. Curated few-shot examples covering this exact pattern for a given schema materially help.

## Cost Considerations

- Schema-in-prompt size directly drives token cost on every single call, which matters at query volume — a 300-table schema pasted into every prompt is both a quality problem (dilutes attention) and a cost problem (pure token overhead repeated on every request). This is the practical reason RAG-over-schema and semantic layers matter beyond correctness alone.
- Execution-feedback retry loops multiply cost on failure — bounding retries (e.g., 2-3 attempts) and monitoring how often the bound is hit is both a cost control and a quality signal (a high retry-exhaustion rate indicates a systemic prompt/schema problem worth fixing upstream, not just tolerating with more retries).

## Quick Gotchas Worth Naming in an Interview

- Never treat "the prompt says read-only" as a security control — the security boundary must be enforced at the database-grant level (a real read-only role, RLS, statement timeouts), because prompts can be bypassed via injection.
- Exact-match evaluation systematically undercounts correct systems; always report execution accuracy (and ideally a soft-match/row-overlap variant) as the headline metric.
- Schema linking, not SQL syntax generation, is usually the actual bottleneck on real enterprise schemas — modern LLMs write syntactically fine SQL almost every time; they get the wrong table/column/join far more often.
- An execution-feedback loop that actually runs the SQL and reads the real database error is meaningfully stronger than a self-reflection loop where the model just re-reads its own draft without executing anything.
