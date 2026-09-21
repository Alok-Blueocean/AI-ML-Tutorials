# Retrieval-Augmented Generation (RAG) — Condensed Study Notes

## Why RAG Exists

- An LLM's knowledge is frozen at training time and has no access to your private/proprietary data. RAG fixes both problems by fetching relevant text at answer time and handing it to the model as context, instead of (or in addition to) relying on parametric memory.

- Real-world example: a support-ticket assistant answers questions about a refund policy that changed yesterday, without any retraining, because the updated policy PDF was re-indexed into the vector store overnight.

## The End-to-End Pipeline

```
documents -> chunk -> embed -> index (vector DB)
                                    |
query -> (optional query rewrite) -> embed(query) -> retrieve top_k -> rerank -> top_n
                                                                              |
                                                          build_prompt(query, top_n) -> LLM -> answer
```

- **Indexing (offline)**: parse documents, split into chunks, embed each chunk, store vectors + metadata in a vector store (FAISS, pgvector, Pinecone, Weaviate, Milvus).
- **Retrieval (online)**: embed the incoming query with the *same* embedding model family used at indexing time, run approximate nearest-neighbor (ANN) search, return top-k candidates.
- **Reranking (online, optional but recommended)**: a more expensive cross-encoder rescoring pass narrows top-k (e.g., 50) down to top-n (e.g., 5) by real relevance, not just embedding-space proximity.
- **Generation**: the LLM answers using only (or primarily) the retrieved text, ideally with citations back to source chunks.

- Real-world example: a legal-research tool retrieves 40 candidate case-law paragraphs via vector search, reranks them with a cross-encoder, and only feeds the top 5 to the LLM — this two-stage design keeps the prompt short (cheap, fast) while keeping quality high (accurate).

## Chunking Strategies and Tradeoffs

- **Fixed-size chunking**: split every N tokens/characters with some overlap. Simple, predictable cost, but can slice a sentence or a table row in half, losing the exact fact you needed.
- **Recursive/structure-aware chunking**: split on natural boundaries first (section → paragraph → sentence), falling back to fixed-size only when a piece is still too large. Keeps chunks semantically coherent; the default choice in most frameworks (e.g., LangChain's `RecursiveCharacterTextSplitter`).
- **Semantic chunking**: group consecutive sentences by embedding similarity so a boundary falls where the topic actually shifts, not at an arbitrary character count. More expensive to compute (needs embeddings at chunk-time), best when documents mix many topics per page.

- The tradeoff is always the same: smaller chunks improve retrieval precision (a query embedding matches a tightly-scoped chunk better) but risk losing surrounding context the model needs to answer fully; larger chunks preserve context but dilute the embedding and waste tokens on irrelevant text, which also increases hallucination risk because the model has to "find the needle" itself.

- Real-world example: a policy-document chunker split "Employees are entitled to 15 days of PTO" and "unless they are on a probationary period" into two different fixed-size chunks. Retrieval returned only the first chunk, and the assistant told an employee they had unconditional PTO — a chunk-boundary information-loss failure, not a model failure.

## Embedding Model Choice

- Tradeoffs: dimensionality (higher = more expressive, more storage/compute), domain fit (a general-purpose embedding model may underperform a domain-tuned one on legal/medical/code text), context length the embedding model accepts, latency, and cost (hosted API vs. self-hosted open model).
- The query and the documents must be embedded with the same model (and ideally the same version) — swapping embedding models without re-indexing the whole corpus silently breaks retrieval, because the new query vectors and the old document vectors no longer live in a comparable space.
- Real-world example: a team upgraded their embedding model for better benchmark scores but forgot to re-embed the existing 2M-document index; retrieval quality quietly collapsed because queries were now compared against vectors from an incompatible embedding space.

## Hybrid Search (Dense + Sparse)

- Dense (embedding) search is strong at paraphrase/semantic matches, weak at exact tokens (SKUs, error codes, person names, acronyms) because embeddings are trained to capture meaning, not exact strings.
- Sparse/keyword search (BM25 — term-frequency weighted by inverse document frequency, with length normalization) is the opposite: exact-match strong, paraphrase-blind.
- Hybrid search runs both and fuses the result lists, commonly with **Reciprocal Rank Fusion (RRF)**: `score(doc) = sum over retrievers of 1 / (k + rank_in_that_retriever)`. This gets both semantic recall and keyword precision without hand-tuned weights.
- Real-world example: an e-commerce search team combined BM25 (to keep exact SKU/model-number matches working) with dense retrieval (to catch "comfortable running shoes for flat feet" style descriptive queries) — pure vector search alone was regressing exact-match queries during A/B testing.

## Reranking with Cross-Encoders

- A **bi-encoder** (used for the initial retrieval) embeds the query and each document independently, so similarity is a single dot product/cosine — fast, scalable to millions of documents, but less accurate because the query and document never actually attend to each other.
- A **cross-encoder** reranker feeds the query and a candidate document together into one model and outputs a single relevance score — much more accurate because it can model query-document interaction directly, but too slow to run over an entire corpus, so it's only applied to the top-k candidates from a cheaper first pass.
- Real-world example: a customer-support RAG system saw a measurable drop in "wrong chunk used" complaints after adding a cross-encoder reranking step on top of its existing dense retrieval, at the cost of ~100-200ms added latency per query.

## Evaluating RAG Systems (RAGAS-style metrics)

RAG evaluation needs to separate retrieval quality from generation quality — a good answer built on the wrong context is still a defect waiting to happen.

- **Context precision**: of the chunks retrieved, what fraction are actually relevant to the question? Low precision = noisy context diluting the prompt.
- **Context recall**: of the chunks that *should* have been retrieved (ground-truth relevant), what fraction were actually retrieved? Low recall = the answer to the question was never even given to the model — a retrieval-miss, not a generation failure.
- **Faithfulness (groundedness)**: does every claim in the generated answer trace back to the retrieved context? Low faithfulness = hallucination even when good context was retrieved.
- **Answer relevancy**: does the generated answer actually address the question asked (independent of whether it's grounded)? A perfectly faithful but off-topic answer still fails this.
- The RAGAS framework operationalizes these four metrics (plus others) using an LLM as the scorer, so an entire RAG pipeline can be regression-tested automatically rather than eyeballed.

```python
# RAGAS-style evaluation loop (conceptual)
from ragas import evaluate
from ragas.metrics import context_precision, context_recall, faithfulness, answer_relevancy

result = evaluate(
    dataset,  # question, retrieved_contexts, answer, ground_truth per row
    metrics=[context_precision, context_recall, faithfulness, answer_relevancy],
)
```

- Real-world example: a team assumed their chatbot was "hallucinating" based on user complaints, but a RAGAS run showed high faithfulness and low context recall — the real bug was retrieval missing the right chunk, not the LLM inventing facts. The fix was in chunking/indexing, not prompting.

## Advanced Patterns

- **Query rewriting**: use the LLM to reformulate a vague or conversational user query into a more retrieval-friendly form (expanding acronyms, resolving pronouns from chat history, generating multiple sub-questions) before embedding it.
- **HyDE (Hypothetical Document Embeddings)**: instead of embedding the raw query, ask the LLM to write a hypothetical answer to the question, then embed *that* and search with it. A hypothetical answer is often closer in embedding space to real answer-bearing documents than the terse original question is.
- **Multi-hop / agentic RAG**: some questions need multiple retrieval steps where the second query depends on what the first retrieval returned (e.g., "who is the CEO of the company that acquired X?" needs to first find "who acquired X" and then a second lookup). An agent loop plans, retrieves, checks if it has enough information, and retrieves again if not, rather than doing one shot retrieve-then-generate.
- **GraphRAG**: builds a knowledge graph (entities + relationships extracted from the corpus) alongside or instead of a flat vector index, so questions requiring reasoning across many documents ("summarize everything about theme X across the whole corpus") can be answered by traversing the graph/community structure instead of relying on top-k similarity search, which struggles with "holistic" questions that aren't localized to a few chunks.
- Real-world example: a research-assistant tool answering "what are the major criticisms of this policy across all 200 submitted comments" performed poorly with plain vector RAG (no single chunk contains "all criticisms") but improved substantially with a GraphRAG-style approach that first clustered themes across the corpus.

## Document Parsing and Preprocessing

- Before chunking even begins, raw documents (PDFs, HTML, Word docs, scanned images) have to be parsed into clean text, and this step quietly determines a large fraction of downstream retrieval quality. PDFs with multi-column layouts, embedded tables, headers/footers repeated on every page, and OCR'd scans are the most common sources of garbage-in-garbage-out failures.
- Tables are a particularly common failure point: naive text extraction flattens a table into a run-on sentence that loses the row/column structure a question like "what was Q3 revenue in the EMEA row" depends on. Production pipelines often extract tables separately (structured, e.g., as markdown tables or key-value pairs) and chunk them independently from surrounding prose.
- Real-world example: a financial-analysis RAG tool answered "what was the gross margin in 2023" incorrectly because the source PDF's table had been flattened into plain text during parsing, scrambling which number belonged to which year and which line item — a preprocessing failure, not a retrieval or generation failure, but one that looked identical to a hallucination from the outside.

## Vector Database Selection

Common options and the axes that actually matter when choosing one:

| | Best for | Notes |
|---|---|---|
| FAISS | Prototyping, single-node, full control | In-process library, not a managed service — you own persistence, scaling, and filtering logic yourself. |
| pgvector | Teams already on Postgres | Vector search alongside relational data and existing operational tooling (backups, auth) in one system; simpler ops than a dedicated vector DB, less specialized at very large scale. |
| Pinecone / Weaviate / Milvus | Production, managed, scale | Handle sharding, replication, metadata filtering, and hybrid search natively; managed pricing/operational tradeoffs vary. |

- The decision usually comes down to: expected corpus scale (millions vs. billions of vectors), whether you need metadata filtering and multi-tenancy built in, whether your team wants to operate the infrastructure itself, and whether you're already committed to a relational database that could absorb this workload instead of adding a new system.
- Real-world example: a team started with FAISS for a proof of concept, but migrated to a managed vector database once they needed per-customer metadata filtering (multi-tenant isolation) and horizontal scaling past what a single-node in-process index could handle reliably.

## Metadata Filtering and Multi-Tenancy

- Real production RAG systems almost never search "the whole index" — they filter by metadata first (document type, department, access level, tenant/customer ID, date range) and search within that filtered subset. This both improves relevance (less irrelevant material to compete with) and is often a hard security requirement, not just an optimization.
- Real-world example: a multi-tenant SaaS support-bot RAG system that failed to filter by tenant ID at the vector-search layer (relying only on prompt instructions to "only discuss this customer's data") returned another customer's internal documentation in response to a support query — a data-isolation failure that metadata filtering enforced at the database query level, not the prompt level, would have prevented.

## Cost and Latency Considerations

- Every stage adds cost and latency: embedding the query, the ANN search itself, reranking (if used), and the generation call — and the generation call's cost scales directly with how many tokens of retrieved context get stuffed into the prompt.
- A common production tension: wider initial retrieval (more candidates) improves the odds reranking finds the truly relevant chunk, but costs more compute at the reranking stage; narrower initial retrieval is cheaper but risks the right chunk never being a candidate at all. This is tuned against a labeled eval set, not fixed by default settings.
- Caching is a standard lever: caching embeddings for unchanged documents (avoid re-embedding on every index rebuild), and caching full responses for frequently repeated queries, materially cuts cost at scale.

## Multi-Modal and Structured RAG

- RAG isn't limited to plain text: tables, images, and even code can be retrieved and passed to a multi-modal model, or a text-extraction/captioning step can convert non-text content into embeddable text first.
- Structured RAG (retrieving from a database or knowledge graph rather than, or in addition to, unstructured documents) blends with the NLQ and GraphRAG patterns discussed elsewhere in this kit — the retrieval step doesn't have to be vector search specifically, it just has to fetch relevant grounding content before generation.
- Real-world example: an engineering-docs assistant retrieves both a relevant text explanation and the actual code snippet it refers to as two linked chunk types, because answering "how do I configure X" often requires showing the code, not just describing it in prose.

## Common Production Failure Modes

- **Stale index**: the underlying documents changed but the vector index wasn't refreshed, so the system confidently serves outdated information. Needs a defined re-indexing trigger (on document change, scheduled, or event-driven), not an ad hoc "someone remembers to re-run it" process.
- **Chunk-boundary information loss**: a fact spans a chunk boundary (a condition and its exception, a table header and its rows) and gets split across two chunks that don't both get retrieved together. Mitigated by chunk overlap, structure-aware chunking, or parent-child ("small-to-big") retrieval where a small chunk is used for matching but its larger parent section is what's actually handed to the model.
- **Retrieval-generation mismatch**: the right chunks were retrieved, but the LLM ignores them and answers from its own parametric memory instead, or over-weights an irrelevant chunk that happened to also be retrieved. Mitigated by explicit grounding instructions ("answer only using the provided context; say you don't know if it's not covered"), lower temperature, and faithfulness evaluation in the eval loop.
- **Embedding/model drift after silent upgrades**: swapping an embedding model or its version without re-indexing the whole corpus (see Embedding Model Choice above).

## Prompt Construction for Grounded Generation

- The prompt that assembles retrieved chunks into a final generation call is its own design surface, not an afterthought. A robust template typically includes: an explicit grounding instruction, the retrieved chunks each labeled with a source identifier, the user's question, and an explicit instruction for what to do when the context doesn't contain the answer.

```
SYSTEM:
You are a support assistant. Answer only using the context below.
If the answer is not contained in the context, say "I don't have that information."
Always cite the source id(s) you used in square brackets, e.g. [doc_3].

CONTEXT:
[doc_1] Refund requests must be submitted within 30 days of purchase.
[doc_3] Store credit refunds do not require a receipt if under $50.

QUESTION:
{user_question}
```

- Real-world example: a team that omitted the explicit "say you don't have that information" clause found their assistant filling gaps with plausible-sounding invented policy details whenever retrieval came back thin; adding that single instruction line measurably raised the correct-abstention rate in their eval set without any retrieval-side changes.

## Streaming Answers and Citations

- Production chat UIs typically stream tokens as they're generated for perceived responsiveness, which interacts with citation design: citations are easier to render reliably as inline markers (`[doc_3]`) resolved to a footnote/source list after the full response completes, rather than trying to interleave rich citation UI mid-stream.
- Real-world example: an internal knowledge-base assistant initially tried to render full document previews inline as citations mid-stream, causing UI jank; switching to lightweight inline markers resolved after streaming completion fixed both the UX and simplified the backend citation-tracking logic.

## Whiteboarding a RAG System (Interview Framing)

When asked to "design a RAG system" in an interview, a strong answer walks through, in order: (1) what documents and how they're ingested/parsed, (2) chunking strategy and why, (3) embedding model and vector store choice with the scale/cost tradeoff named explicitly, (4) whether hybrid search and reranking are included and why, (5) the exact prompt construction pattern for grounding, (6) how staleness/re-indexing is handled, and (7) how the system is evaluated (naming RAGAS-style metrics) before and after shipping changes. Naming failure modes proactively (stale index, chunk-boundary loss, retrieval-generation mismatch) signals production experience beyond having built one RAG demo.

## RAG vs Fine-Tuning vs Long Context

- These three approaches to "getting an LLM to use knowledge it wasn't trained on" are complementary, not competing, and a senior interview answer should name when each is the right tool:
  - **RAG**: best when the knowledge source changes frequently, needs to be traceable/citable, or is too large to bake into weights or a single prompt.
  - **Fine-tuning**: best for teaching a durable behavior, style, or narrow domain pattern that doesn't change often and doesn't need per-answer citation back to a source document.
  - **Long context (just paste everything into the prompt)**: viable for a bounded, moderate-size corpus that fits comfortably in context and doesn't need to scale to millions of documents or per-query filtering, but doesn't remove the need for retrieval once the source corpus outgrows even a very large context window.
- Real-world example: a legal-tech company uses RAG for case-law lookup (constantly updated, must cite sources), a fine-tuned model for consistent legal-brief formatting/style (a stable, durable behavior), and neither for a single 20-page contract review, which fits directly in context without any retrieval step at all.

## Building a RAG Evaluation Dataset

- A good RAG eval set needs, per question: the question itself, a ground-truth answer, and the ground-truth relevant chunk(s) — without the last one, context recall can't be measured at all, only context precision on whatever was retrieved.
- Include three categories deliberately: questions clearly answerable from the corpus, questions with partial/ambiguous answers, and questions with no answer in the corpus at all (to measure correct abstention — does the system say "I don't know" instead of guessing).
- Real-world example: a team's first RAG eval set only contained clearly-answerable questions, so it never caught that the system confidently fabricated answers to out-of-scope questions until a real user asked one in production — adding a deliberate "no answer exists" category to the eval set closed that blind spot before the next release.

## Monitoring a RAG System in Production

- Beyond periodic offline evaluation, a production RAG system needs live signals: retrieval latency per stage, the distribution of similarity scores for retrieved chunks (a sudden drop can indicate an indexing problem or a shift in query patterns), user feedback signals (thumbs up/down, or implicit signals like immediate rephrasing of the same question), and a sample of production transcripts routed to periodic human review.
- Real-world example: a team added a dashboard tracking the average top-1 similarity score per day and caught a silent embedding-service outage (queries were falling back to a degraded default embedding) within hours instead of waiting for user complaints to accumulate over days.

## Quick Gotchas Worth Naming in an Interview

- More retrieved chunks is not strictly better — it increases both cost/latency and the odds of confusing the model with irrelevant context ("lost in the middle" effects on long contexts).
- Recall and precision trade off against each other in retrieval just like in classic IR — a production system usually tunes top-k and reranking cutoffs against a labeled eval set, not intuition.
- RAG does not eliminate hallucination — it reduces it when done well, but a model can still ignore correct retrieved context or blend it with parametric memory incorrectly. Faithfulness must be measured, not assumed.
- Hybrid search is not "vector search plus keyword search as a checkbox" — the fusion method (RRF, weighted scores, or a trained reranker over both) is where the actual quality gain comes from.
