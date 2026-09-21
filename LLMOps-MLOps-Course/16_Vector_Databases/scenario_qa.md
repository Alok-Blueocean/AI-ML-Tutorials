# Vector Databases — Scenario-Based Q&A

**Situation:** Your team prototypes a RAG feature using an in-memory FAISS index with 3,000 document chunks, and it works great in the demo. Product now wants to ship it to all users, and the corpus will grow to 2 million chunks with multiple teams writing to it concurrently. What would you do and why?

Model answer: Recognize this as a scale/operational-requirements threshold, not just "make FAISS bigger." An in-memory library has no persistence (a process restart loses the index), no concurrent-write safety, and no built-in filtering, all of which production now requires. Migrate to a dedicated vector database (Qdrant, Milvus, Weaviate, or a managed option like Pinecone) chosen based on the team's ops appetite — self-hosted if you want control, managed if you want to skip infra work. Re-embed the corpus into the new store, validate recall parity against the old FAISS baseline on a fixed query set before cutover, and build the ingestion path to handle concurrent upserts from multiple teams safely.

---

**Situation:** After switching your embedding provider from an older model to a newer, better-benchmarking one, your RAG system's retrieval quality collapses in production even though nothing else changed. What would you do and why?

Model answer: This is almost certainly an embedding-space mismatch: the existing index holds vectors from the old model, but new queries are now embedded with the new model, and the two are not comparable even though both are "embeddings." Confirm by checking whether similarity scores across the board dropped to near-random levels. Fix by re-embedding and re-indexing the entire corpus with the new model before routing any query traffic to it — never mix vectors from two different embedding models in one collection. Treat the embedding model as a versioned dependency with an explicit migration plan (dual-write old and new indexes during a transition window, cut over once re-indexing is validated) rather than a drop-in swap.

---

**Situation:** A stakeholder asks why your vector database costs are climbing steeply month over month even though user traffic has only grown modestly. What would you do and why?

Model answer: Investigate the storage and query-volume drivers separately before assuming it's "just scale." Check whether the corpus has stale, duplicate, or never-deleted chunks accumulating (a common cause — ingestion pipelines that insert but never clean up superseded documents), whether the embedding dimensionality is larger than necessary for the task, and whether queries are unnecessarily wide (e.g., retrieving top-50 when top-10 would do, or re-embedding the same content repeatedly instead of caching). Right-size the index by pruning stale vectors, consider a lower-dimensional embedding model if quality holds up in an A/B test, and confirm whether a managed provider's pricing tier no longer matches actual usage patterns before paying for headroom you don't need.

---

**Situation:** An engineer wants to store one collection per customer in a multi-tenant SaaS product "for clean separation," but the product now has 40,000 customers. What would you do and why?

Model answer: Push back on a collection-per-tenant design at this scale — most vector databases were not built to efficiently manage tens of thousands of small collections, and the operational overhead (schema/index management, connection overhead, uneven collection sizes) grows linearly with tenant count in a way that doesn't scale. Recommend a single shared collection with a `tenant_id` metadata field and metadata filtering applied on every query instead, which keeps one index to operate and tune while still guaranteeing tenant isolation at the query layer. Validate that the chosen vector database's filtering is efficient at this cardinality (some ANN indexes handle pre-filtering better than post-filtering) before committing, and add a test that verifies a query for tenant A can never return tenant B's vectors.

---

**Situation:** Your e-commerce search team wants to add semantic search on top of their existing BM25 keyword search, but worries it will hurt exact-match queries like SKU numbers and exact model names. What would you do and why?

Model answer: Don't replace BM25 — run hybrid search, combining vector similarity with keyword search and fusing the two result sets (commonly via Reciprocal Rank Fusion or a trained reranker over the union of both candidate lists). This keeps BM25's exact-token strength for SKUs and model numbers while adding dense retrieval's strength on descriptive, paraphrased queries. Validate with an A/B test tracking click-through rate and zero-result rate specifically, since the real goal is fewer "no results" pages without regressing exact-match precision on the queries that already worked well.

---

**Situation:** Your vector index has grown to 80 million vectors and p95 query latency has crossed 1.5 seconds; the business wants this fixed without a quality regression. What would you do and why?

Model answer: Profile first to find the actual bottleneck — is it the ANN search itself, the embedding call for the incoming query, or a downstream reranking step? If the index is the bottleneck, move from a flat/brute-force search to an approximate index (HNSW or IVF-based) if not already using one, and consider sharding across nodes. Reduce the candidate pool upstream with metadata filtering (restrict to the relevant tenant/category before vector search) rather than searching the full 80M every time. Re-measure recall after any ANN parameter change, since approximate search trades a small — sometimes silent — recall loss for speed, and that tradeoff needs to be measured against a labeled eval set, not assumed acceptable.

---

**Situation:** A teammate proposes storing full document text as vector-database metadata payloads "for convenience, so we don't need a separate lookup," for documents averaging 50KB each. What would you do and why?

Model answer: Flag this as a likely anti-pattern before it ships. Vector databases are optimized for vector storage and ANN search, not for storing and serving large blobs of text as query-time payloads — doing so bloats the index, slows down every query that returns payloads, and increases storage cost disproportionately to the actual need. Recommend storing only a chunk ID and lightweight filterable metadata (source, date, category, permissions) in the vector database, with the full document text kept in a separate object store or document database keyed by that same ID, fetched only when a chunk is actually selected for the final context. This keeps the vector index lean and fast while still giving full-text access when needed.

---

**Situation:** Your team needs to support "delete all data belonging to this user" for GDPR compliance, but vectors for a given user are scattered across many documents with no easy way to find them all. What would you do and why?

Model answer: This is a metadata design problem that should have been solved at ingestion time, not retrofitted under compliance pressure — but it's fixable now. Ensure every vector's payload includes a `user_id` (or `owner_id`) field at write time going forward, and backfill it for existing vectors by cross-referencing the source document's ownership records. Once that field exists, a deletion request becomes a straightforward metadata-filtered delete query rather than a full-corpus scan. Build and test this deletion path before a real request arrives, log every deletion for audit purposes, and confirm the vector database actually performs a hard delete (not just marking records invisible) if your compliance obligations require it.

---

**Situation:** A junior engineer asks whether cosine similarity, dot product, and Euclidean distance will "basically give the same results" for their text embeddings, and whether it matters which one they configure the collection with. What would you do and why?

Model answer: Explain that it matters and is not a free choice — the distance metric must match what the embedding model was trained/normalized for, and mismatches silently degrade retrieval quality without throwing any error. Most modern text embedding models are trained and normalized for cosine similarity (or an equivalent normalized dot product), so configuring the collection with raw Euclidean distance on unnormalized vectors can produce meaningfully worse rankings even though the query "succeeds." Check the embedding model's documentation for its intended distance metric, configure the collection to match exactly, and verify with a small labeled query set that results look sane before trusting the default.

---

**Situation:** Leadership asks whether the company should build semantic search on an existing PostgreSQL database using `pgvector` instead of adopting a dedicated vector database like Qdrant or Pinecone, to avoid adding a new piece of infrastructure. What would you do and why?

Model answer: Frame it as a real tradeoff, not a clear win either way. `pgvector` is a strong choice when the vector workload is moderate in scale, when keeping vectors co-located with existing relational data simplifies joins and transactional consistency, and when the team wants to avoid operating a new system. A dedicated vector database becomes the better choice once you need very large scale, sub-100ms latency at high query volume, or advanced ANN tuning and index types that `pgvector` doesn't yet match as well. Recommend benchmarking `pgvector` against the actual expected scale and latency SLA before deciding, rather than choosing based on "fewer moving parts" alone or defaulting to a dedicated system out of habit.

---

**Situation:** A RAG pipeline's retrieval quality looks fine in offline evaluation but users report getting outdated answers about a policy that changed two weeks ago. What would you do and why?

Model answer: Suspect a stale-index problem before suspecting the retrieval algorithm itself. Check whether the ingestion pipeline that re-embeds and upserts updated documents is actually running on the expected schedule, and whether the old (superseded) chunks were ever deleted or just left alongside the new ones — if both versions exist, the retriever may still be surfacing the outdated chunk simply because it scores similarly. Fix the ingestion pipeline's freshness (a monitored, scheduled or event-driven re-index job) and add an explicit delete/replace step for superseded content rather than pure appends. Add a synthetic canary query tied to a known-current fact to your monitoring so index staleness is caught automatically rather than via user complaints.

---

**Situation:** Your team is debating whether to use a managed vector database (Pinecone) or a self-hosted open-source one (Qdrant/Milvus/Weaviate) for a new product that's still finding product-market fit. What would you do and why?

Model answer: Weigh the tradeoff against the project's current stage, not just steady-state cost. A managed service removes operational burden (no cluster ops, scaling, or upgrades to manage) at a point where engineering time is the scarcest resource and the product's scale/requirements are still changing rapidly — that favors Pinecone early on. Once scale, cost sensitivity, or data-residency/compliance requirements become clearer and larger, self-hosting can become materially cheaper and gives more control over tuning. Recommend starting managed to move fast, with an explicit note to revisit the decision once monthly vector-database spend or compliance requirements cross a defined threshold, rather than either defaulting to self-hosted "to save money" prematurely or staying managed indefinitely without checking whether the tradeoff still holds.

---

**Situation:** A user asks a "holistic" question — "what are the recurring complaints across all 10,000 support tickets this quarter?" — and your standard top-k vector RAG gives a shallow answer reflecting only a handful of tickets. What would you do and why?

Model answer: Recognize this as a structural limitation of top-k similarity retrieval, not a bug to fix by tuning k higher — no single top-k retrieval, however large, reliably surfaces a complete picture across 10,000 documents in one shot, and pushing k up mostly adds noise and cost. Reach for a different pattern for aggregate questions: a map-reduce summarization pass over the full corpus, or a pre-clustering/theme-extraction step (potentially graph-based) that summarizes the corpus ahead of time so the question can be answered by traversing precomputed summaries. Set the expectation with stakeholders up front that plain vector RAG is built for localized fact lookup, not corpus-wide aggregation, so the right tool gets used for the right question.

---

**Situation:** During a security review, someone points out that your vector database allows any authenticated service to query across all tenants' vectors, relying only on the application layer to filter by `tenant_id` before displaying results. What would you do and why?

Model answer: Treat this as a real risk, not a theoretical one — relying solely on application-layer filtering means any bug in that filtering logic (a missing `WHERE`-equivalent clause, a misconfigured query) becomes a cross-tenant data leak with no defense in depth. Push the tenant boundary down into the query itself as a mandatory, non-optional filter (not an optional parameter the caller can forget to pass), and where the vector database supports it, use namespace/collection-level isolation or row-level security equivalents rather than trusting every call site to remember the filter. Add an automated test that specifically attempts a cross-tenant query and asserts it returns nothing, and treat that test as a release gate, not a nice-to-have.

---

**Situation:** Your team's vector database query latency is fine on average, but a small fraction of queries take 5-10x longer than the rest, and nobody can explain why. What would you do and why?

Model answer: Investigate whether the slow queries share a pattern rather than assuming random noise — common culprits are overly broad metadata filters that force the index to scan far more candidates than a typical query, requests with unusually large `top_k` values, or queries hitting a cold shard/replica that hasn't warmed its cache. Instrument query latency with the actual filter and `top_k` parameters attached so slow outliers can be correlated with their inputs, not just their timestamps. Once the pattern is identified, fix it at the source — cap `top_k`, add a pre-filter index, or rebalance shard load — rather than papering over it with a blanket timeout that would silently drop legitimate slow-but-important queries.

---

**Situation:** A new hire proposes re-computing all embeddings and rebuilding the entire vector index from scratch every night "to keep things fresh," even though only a small fraction of documents change daily. What would you do and why?

Model answer: Push back on full nightly rebuilds as wasteful and risky at this scale — recomputing embeddings for millions of unchanged documents burns compute and API cost for no benefit, and a full rebuild also means a window where either the old or new index is incomplete unless carefully staged. Recommend incremental updates instead: track which documents actually changed (via a content hash, last-modified timestamp, or a change-data-capture feed) and only re-embed and upsert those, leaving the rest of the index untouched. Reserve full rebuilds for genuine breaking changes, like an embedding model migration, and even then stage them with a blue-green cutover rather than rebuilding in place.

---

**Situation:** Your company is evaluating whether to add a built-in vectorization/embedding pipeline offered by a vector database vendor, versus continuing to generate embeddings yourselves and just storing vectors. What would you do and why?

Model answer: Weigh convenience against lock-in and flexibility. A vendor's built-in embedding pipeline reduces integration work and can simplify the stack, which is attractive for a small team or fast-moving prototype. But it also couples your retrieval quality to that vendor's embedding model choices and update cadence, makes it harder to A/B test alternative embedding models, and can complicate a future migration to a different vector database since your embeddings become entangled with the vendor's pipeline. Recommend generating and owning embeddings yourselves once the product has real production traffic and retrieval quality matters, reserving vendor-managed embedding pipelines for early prototypes or genuinely low-stakes use cases where switching cost is not a concern.
