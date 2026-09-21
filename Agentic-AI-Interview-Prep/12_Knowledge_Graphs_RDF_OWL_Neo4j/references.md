# Knowledge Graphs (RDF, OWL, Property Graphs, Neo4j) — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise. Note: this session's WebSearch quota was exhausted before general "interview questions" article searches could run, so this file leans more heavily on primary-source standards/docs/repos than curated interview-prep articles — treat that as a feature, not a gap, since primary sources are more reliable anyway.

## Standards

- RDF 1.1 Concepts and Abstract Syntax (W3C Recommendation). https://www.w3.org/TR/rdf11-concepts/
- OWL 2 Web Ontology Language, Document Overview, 2nd Edition (W3C Recommendation). https://www.w3.org/TR/owl2-overview/
- SPARQL 1.1 Query Language (W3C Recommendation). https://www.w3.org/TR/sparql11-query/
- SHACL — Shapes Constraint Language, for validating RDF graphs (W3C Recommendation). https://www.w3.org/TR/shacl/

## Neo4j

- Cypher Manual — current docs. Note: as of Neo4j 2025.06, Cypher forked into "Cypher 25" (active development) and "Cypher 5" (frozen, backward-compatibility only) — know this split before an interview. https://neo4j.com/docs/cypher-manual/current/
- Graph Data Science (GDS) library docs — graph algorithms (centrality, community detection, pathfinding) and ML procedures. https://neo4j.com/docs/graph-data-science/current/
- `neo4j-graphrag-python` — Neo4j's official, actively maintained GraphRAG Python package (~1.3k stars, supports OpenAI/Google/Anthropic/Cohere LLMs). This is the flagship, best-maintained piece of Neo4j's GenAI tooling. https://github.com/neo4j/neo4j-graphrag-python
- LLM Knowledge Graph Builder — Neo4j Labs product turning PDFs/documents/web pages/YouTube transcripts into a knowledge graph via an LLM; hosted app and self-deployable via Docker. https://neo4j.com/labs/genai-ecosystem/llm-graph-builder/
- `text2cypher` — Neo4j Labs research repo with datasets, evaluation notebooks, and fine-tuning instructions for natural-language-to-Cypher generation. Real and current, but notably lower activity (~244 stars, 27 commits) than the flagship GraphRAG package — present it as a research-stage effort, not a polished product. https://github.com/neo4j-labs/text2cypher

## LLM + Knowledge Graph Frameworks

- LangChain's `LLMGraphTransformer` (in `langchain_experimental.graph_transformers`) — a real, well-known feature for extracting a graph from text via an LLM. LangChain's docs site was recently restructured (old `python.langchain.com` URLs now redirect to a new unified reference site), so the specific current doc page should be verified fresh rather than trusted from an old bookmark. (not URL-verified this session to a specific stable page — flagged as "docs churn," verify at build/interview-prep time)
- LlamaIndex Property Graph Index — current official docs, note the doc host itself moved from `docs.llamaindex.ai` to `developers.llamaindex.ai` (old links redirect). https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/
- Microsoft GraphRAG — https://github.com/microsoft/graphrag (~36k stars). **Important: the project is now largely in maintenance mode** per its own repo notice — bug/security fixes only, no new features or PRs being accepted, with the maintainers explicitly citing that frontier model capabilities have changed substantially since the project's July 2024 release. Underlying paper: Edge et al., "From Local to Global: A Graph RAG Approach to Query-Focused Summarization," arXiv:2404.16130.
- LightRAG — https://github.com/HKUDS/LightRAG (~39.8k stars, now more-starred than Microsoft GraphRAG). Positions itself explicitly as a lighter-weight, lower-LLM-call-cost alternative to Microsoft GraphRAG, using a dual-layer knowledge-graph-plus-vector architecture. The most prominent current GraphRAG alternative — know this name for a "what's newer than GraphRAG" interview question.

## Entity Resolution

- Splink — https://github.com/moj-analytical-services/splink (~2.4k stars, actively engineered, "Splink 4" recently released, multi-backend SQL support via DuckDB/Spark/etc., backed by the UK Ministry of Justice). The more current, production-scale-oriented choice.
- Dedupe — https://github.com/dedupeio/dedupe (~4.5k stars, more total stars but lower confirmed recent-activity signal than Splink). A legitimate, simpler earlier alternative, particularly for smaller-scale deduplication tasks.

## Entity/Relation Extraction

- REBEL (`Babelscape/rebel-large`) — a BART-based end-to-end relation-extraction model outputting (head, relation, tail) triplets directly, 200+ relation types, reported F1 93.4% on the NYT relation-extraction dataset. A genuine, commonly used tool for automated knowledge-graph population from text. https://huggingface.co/Babelscape/rebel-large
- spaCy — industry-standard NLP library commonly used for the NER step of knowledge-graph construction pipelines. (well-established, not independently re-verified this session, but not in question)

## Evaluation

- RAGAS — general-purpose LLM/RAG evaluation framework (faithfulness, response groundedness, answer accuracy, factual correctness, context precision/recall, response relevancy). Actively maintained and current, but note it is a general RAG framework, not graph-specific — most teams adapt its faithfulness/groundedness metrics to a graph context (e.g., checking a claimed relationship against an actual graph existence query) rather than using a dedicated, standardized "KG hallucination" benchmark, because no single such standard has crystallized as dominant yet. https://docs.ragas.io/en/stable/
- A unified graph-based RAG benchmarking framework paper exists (arXiv:2503.04338) comparing retrieval methods across graph-RAG approaches, but it addresses retrieval-method comparison, not a general-purpose KG-hallucination diagnostic standard. (not independently verified beyond search-level confirmation this session)

## Notes on What to Prioritize

There is no single canonical "knowledge graph interview questions" article to point to (this session's search budget was exhausted before that pass could run) — build interview readiness from the primary standards/docs above plus the recent-change findings, which are themselves strong interview material. Interview signal to prioritize: the RDF/OWL vs. property-graph tradeoff (interoperability/formal reasoning vs. flexible application-embedded querying), knowing that Microsoft GraphRAG is now maintenance-mode with LightRAG as the rising alternative, knowing Neo4j's GenAI tooling is three separately-maintained projects at different maturity levels (not one product), and being ready to explain why deterministic triple-existence checks make hallucination detection more tractable in a KG-augmented system than in unstructured RAG. Treat any specific LangChain `LLMGraphTransformer` doc URL as needing a fresh check before an interview, since LangChain's docs site was recently restructured.
