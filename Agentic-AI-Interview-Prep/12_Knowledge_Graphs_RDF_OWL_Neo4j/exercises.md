# Knowledge Graphs (RDF, OWL, Property Graphs, Neo4j) — Exercises

Ordered easy to hard. Mix of hands-on build tasks and conceptual questions.

1. **Model a small domain as RDF triples.** Pick a small domain (a company's org chart, or a movie/actor/director dataset). Write 20-30 RDF triples by hand in Turtle syntax, using consistent IRIs for repeated entities.

2. **Conceptual: RDF vs property graph.** For the same domain from exercise 1, sketch how you'd model it as a Neo4j property graph instead. Identify one thing that's easy in RDF but awkward in the property-graph model, and one thing that's the reverse.

3. **Write SPARQL queries.** Load your triples from exercise 1 into a triple store (e.g., Apache Jena Fuseki, or an in-memory RDF library like rdflib in Python) and write 5 SPARQL queries of increasing complexity (simple pattern match, filter, multi-hop path, aggregation).

4. **Build a small OWL ontology.** Define classes, a subclass hierarchy, and at least 2 property domain/range constraints for your domain. Run an OWL reasoner (e.g., via `owlready2` in Python, or Protégé) and show at least one inferred fact that wasn't explicitly asserted.

5. **Conceptual: OWL vs SHACL.** Explain, with an example from your ontology, the difference between an OWL axiom used for inference and a SHACL shape used for validation, and why you'd need both in a real pipeline.

6. **Load your graph into Neo4j and write Cypher queries.** Recreate your domain as a Neo4j property graph. Write 5 Cypher queries mirroring your SPARQL queries from exercise 3, and compare the query styles.

7. **Run a graph algorithm.** Using Neo4j's Graph Data Science library, run a centrality algorithm (e.g., PageRank or betweenness centrality) and a community-detection algorithm on your graph (or a larger public graph dataset). Interpret the results in plain language.

8. **Build an LLM-based extraction pipeline.** Take 10-15 short news articles or Wikipedia paragraphs about related entities (e.g., companies and their acquisitions). Use an LLM to extract (subject, relation, object) triples via structured output, and load the results into Neo4j or an RDF store.

9. **Entity resolution exercise.** Deliberately include entities referenced under 2-3 different surface forms across your source documents from exercise 8 (e.g., "IBM" and "International Business Machines"). Build a simple entity-resolution step (string/embedding similarity + a merge rule) and show before/after node counts.

10. **Conceptual: why entity resolution is the hard part.** Explain, using a concrete example from exercise 9, why entity resolution is often harder and more error-prone than the NER/relation-extraction steps that precede it.

11. **Build a text-to-Cypher tool.** Given your Neo4j graph's schema, build a prompt template that generates Cypher from natural-language questions, with an execution-feedback loop (similar to the text-to-SQL pattern) that retries on Cypher errors.

12. **Build a GraphRAG-style Q&A system.** Combine your knowledge graph with an LLM: given a natural-language question, retrieve a relevant subgraph (via a Cypher/SPARQL query informed by the question) and pass it as structured context to the LLM for a grounded answer. Test on 5 multi-hop questions that a flat text-chunk RAG system over the same source documents would struggle to answer.

13. **Build a hallucination checker for graph-grounded answers.** For your GraphRAG system from exercise 12, write an automated checker that verifies every relationship claimed in a generated answer actually exists as a triple/edge in the graph, and flags any that don't.

14. **Ontology drift simulation.** Simulate ontology drift by adding a second relationship type meaning the same thing as an existing one (e.g., add `MANAGED_BY` alongside an existing `REPORTS_TO`). Show a query that silently undercounts results because it only checks one relationship type, then propose and implement a fix (a canonical relationship-type mapping layer, or a migration).

15. **End-to-end system critique.** Given a described production knowledge graph built from thousands of ingested documents, where users report both duplicate-looking entities and occasional wrong-relationship answers from a GraphRAG assistant built on top of it, propose a prioritized list of 5 concrete interventions (entity resolution, ontology governance, extraction accuracy, retrieval/query generation, evaluation) and justify the order.
