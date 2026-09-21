# NLQ / Text-to-SQL — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise.

## Benchmarks

- Spider — the original cross-domain text-to-SQL benchmark (Yale), 10,181 questions / 5,693 SQL queries / 200 databases. Its leaderboard stopped accepting submissions in Feb 2024, superseded by Spider 2.0. https://yale-lily.github.io/spider
- Spider 2.0 — successor benchmark targeting realistic, long, multi-step SQL workflows rather than single clean queries. https://spider2-sql.github.io/
- BIRD-SQL — "BIg Bench for LaRge-scale Database Grounded Text-to-SQL," 12,751 question-SQL pairs across 95 databases (33.4GB) and 37+ domains. Its leaderboard is explicitly organized around Execution Accuracy (EX), reinforcing that EX (not exact match) is the modern standard metric. https://bird-bench.github.io/ (active spin-offs: BIRD-CRITIC-1, BIRD-Interact, at github.com/bird-bench)

## Papers

- "A Survey of Text-to-SQL in the Era of LLMs: Where are we, and where are we going?" — covers model, data, evaluation, and error-analysis axes. https://arxiv.org/abs/2408.05109 (not independently URL-verified this session beyond abstract search — high confidence given consistent cross-source hits, but re-check the arXiv id before citing)
- DIN-SQL — "Decomposed In-Context Learning of Text-to-SQL with Self-Correction." https://arxiv.org/abs/2304.11015 — note: its self-correction step is intrinsic (the model re-reads its own SQL without executing it) and ablations show it contributes only a small accuracy gain; follow-on multi-agent approaches (e.g., MAC-SQL) use true execution-grounded feedback loops instead, which this kit's tutorial recommends as the stronger pattern. (not URL-verified this session beyond search corroboration)

## Tools

- LangChain SQL agent / "chat with your database" docs. The docs explicitly warn: database connection permissions should always be scoped as narrowly as possible, and the built-in database tools are "minimal wrappers for demonstration purposes only... not intended to be secure or used in production" without added application-specific validation and narrowly scoped permissions. https://docs.langchain.com/oss/python/langgraph/sql-agent
- PandasAI — natural-language-to-DataFrame query library. Active, MIT-licensed, current canonical repo. https://github.com/sinaptik-ai/pandas-ai (docs: docs.pandas-ai.com/v3)
- Vanna.ai — RAG-based open-source text-to-SQL tool with agentic retrieval and row-level-security/audit-logging features. **Important: this repository was archived (made read-only) on March 29, 2026 and is no longer maintained** — do not present it as an actively maintained option in an interview without noting this. https://github.com/vanna-ai/vanna

## Security

- Kyle Kitlinski, "AI Agents and SQL: How to Give LLMs Safe Database Access" — concrete Postgres read-only role examples for LLM-driven SQL agents. (not independently URL-verified this session — surfaced via search with credible, specific technical content; recommended domain: kkit.dev)
- Rietta, "Protect Production SQL Databases from AI/LLM Agentic SQL Query Risks" — covers row-level security for multi-tenant systems and a read-replica isolation pattern for agentic SQL access. (not independently URL-verified this session; recommended domain: rietta.com)
- Real filed security report documenting a prompt-injection-to-SQL-denial-of-service path in an open-source LLM SQL chatbot: chatchat-space/Langchain-Chatchat GitHub Issue #5446, "Security Vulnerability Report: Prompt Injection Leading to SQL-Based Denial of Service." (not independently URL-verified this session — surfaced via search; a concrete, citable real-world incident of the failure mode described in this kit's tutorial and scenario files)

## Articles / Interview Prep

- "Why 90% Accuracy in Text-to-SQL is 100% Useless" (Towards Data Science) — argues exact-match scoring penalizes valid syntactic variants, and that the real enterprise risk is hallucinated table names, misinterpreted filters, and confidently-wrong SQL driving bad business decisions at scale. https://towardsdatascience.com/why-90-accuracy-in-text-to-sql-is-100-useless/

## Notes on What to Prioritize

There is no single canonical "text-to-SQL interview questions" article the way there is for general NLP/LLM interviews — build your prep from the survey paper plus the production-failure articles above rather than expecting a dedicated interview-question listicle. Interview signal consistently points to: execution accuracy vs. exact match (know why EM is considered outdated and be ready to name EX's own limitations), schema linking as the real bottleneck (not SQL syntax), execution-grounded self-correction loops over pure self-reflection, and security as a database-grant-level concern, not a prompt-level one. Flag clearly in an interview that Vanna.ai — a formerly prominent open-source option — was archived in March 2026, since name-dropping it as "the tool I'd currently use" would read as outdated.
