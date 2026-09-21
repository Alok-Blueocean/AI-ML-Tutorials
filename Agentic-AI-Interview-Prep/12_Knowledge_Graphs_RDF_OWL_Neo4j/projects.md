# Knowledge Graphs (RDF, OWL, Property Graphs, Neo4j) — Projects

## Small: Domain Knowledge Graph with Dual Query Interfaces

Model a small real domain (a company's org chart plus project history, or a public dataset like a film/actor/director graph) both as RDF/OWL (queryable with SPARQL, with a small ontology defining classes and key constraints) and as a Neo4j property graph (queryable with Cypher). Build a short comparison writeup of which model felt more natural for which kinds of questions. This proves you understand both graph paradigms concretely, not just in the abstract, and can articulate a real tradeoff rather than a memorized definition.

## Medium: LLM-Powered Knowledge Graph Construction Pipeline with Entity Resolution

Build a pipeline that ingests a moderate-size corpus of unstructured text (news articles, company filings, or a public document set), uses an LLM (via structured-output prompting) to extract entities and relationships, applies an entity-resolution step to merge duplicate entities across documents, and loads the result into Neo4j. Include a small evaluation set with hand-labeled gold triples to measure extraction precision/recall, and report before/after entity counts to demonstrate the impact of your resolution step. This proves you can build the unglamorous but critical middle of a real knowledge-graph pipeline, not just query a pre-built graph.

## Medium: GraphRAG Q&A System with Deterministic Hallucination Checking

Build a question-answering system over your constructed knowledge graph that retrieves a relevant subgraph per question (via a generated Cypher query, with an execution-feedback self-correction loop) and generates a grounded natural-language answer from it. Build an automated evaluation harness that checks every relationship claimed in a generated answer against the actual graph (a deterministic existence check), separately from an LLM-as-judge score for completeness/coherence. Test specifically on multi-hop questions a flat vector-chunk RAG system over the same source text would fail to answer. This proves you understand GraphRAG's specific advantage over plain RAG and can build a hallucination-detection approach that exploits the graph's structure rather than relying solely on generic LLM-as-judge scoring.

## Large: Ontology-Governed Multi-Source Knowledge Graph Platform

Build a knowledge graph ingesting from multiple heterogeneous sources (structured CSVs/databases plus unstructured documents) into a single coherent graph, with a formally versioned ontology (OWL or a documented property-graph schema) governing what entity types and relationship types are valid, SHACL-style validation rejecting non-conforming ingested data, and a change-management process for ontology evolution (e.g., relationship-type deprecation with a migration path rather than silent renaming). Include a monitoring dashboard tracking entity-resolution duplicate rate, ontology-conformance violations, and graph-growth-over-time metrics. This proves you can operate a knowledge graph as a governed, production data platform at the scale where ontology drift and multi-source integration become the real engineering problems, not just build a one-off graph in a notebook.
