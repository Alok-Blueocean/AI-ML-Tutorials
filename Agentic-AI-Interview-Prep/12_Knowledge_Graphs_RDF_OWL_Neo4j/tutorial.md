# Knowledge Graphs (RDF, OWL, Property Graphs, Neo4j) — Condensed Study Notes

## What a Knowledge Graph Is and Why It Exists

- A knowledge graph represents data as **entities (nodes)** connected by **typed relationships (edges)**, with both nodes and edges optionally carrying properties — instead of forcing real-world facts into rigid rows and columns. This makes it a natural fit for data that is fundamentally about connections (who reports to whom, which drug interacts with which condition, which document cites which other document), where the relationships themselves are often more valuable than the entities in isolation.
- Real-world example: a pharmaceutical company modeling drug-gene-disease interactions found that a relational schema needed a new join table (and a schema migration) every time a new relationship type was discovered between research entities; a knowledge graph absorbed new relationship types without any schema migration, because the graph model doesn't require relationships to be pre-declared as fixed foreign-key columns.

## Two Knowledge Graph Models: RDF vs Property Graphs

| | RDF (Resource Description Framework) | Property Graph (e.g., Neo4j) |
|---|---|---|
| Basic unit | Triple: `(subject, predicate, object)` | Node and relationship, each with key-value properties |
| Standardization | W3C standard (RDF, OWL, SPARQL, SHACL) | No single universal standard; vendor-specific (Cypher for Neo4j, Gremlin for others) |
| Schema/ontology | First-class, formal (OWL classes, properties, axioms) | Usually informal/implicit, enforced by application code or optional constraints |
| Query language | SPARQL | Cypher (Neo4j), Gremlin (TinkerPop-based stores), GQL (emerging ISO standard) |
| Reasoning | Native support for logical inference (subclass/subproperty inference, consistency checking) via an OWL reasoner | Not native; reasoning-like behavior is usually implemented in application logic or via graph algorithms |
| Typical use case | Interoperable, standards-based data (life sciences, government/linked open data, semantic web) | Application-embedded graphs (fraud detection, recommendation, enterprise knowledge graphs, GraphRAG) |

- Real-world example: a life-sciences consortium publishing shared, cross-institution biomedical data uses RDF/OWL specifically because SPARQL and OWL ontologies (like standard biomedical ontologies) let independently-built datasets interoperate without every party agreeing on a single proprietary schema; an internal enterprise team building a customer-360 knowledge graph for their own application uses Neo4j property graphs because they don't need cross-organization standardization and want faster iteration and simpler querying.

## RDF: Triples, IRIs, and Literals

- Every RDF fact is a **triple**: `subject — predicate — object`, e.g., `<Company_A> <acquired> <Company_B>`. Subjects and predicates are typically **IRIs** (Internationalized Resource Identifiers, a generalization of URIs) that globally and unambiguously identify a resource; objects can be an IRI (linking to another entity) or a **literal** (a plain value like a string, number, or date).
- Collections of triples form a graph. RDF's strict subject-predicate-object structure is what makes data from completely different sources mergeable — as long as two datasets reuse the same IRIs for the same real-world entities, their triples combine into one coherent graph automatically.

```turtle
# Example RDF triples in Turtle syntax
<http://example.org/Company_A> <http://example.org/acquired> <http://example.org/Company_B> .
<http://example.org/Company_B> <http://example.org/foundedIn> "2015"^^<http://www.w3.org/2001/XMLSchema#gYear> .
```

- Real-world example: a government open-data initiative publishing city budget data as RDF lets a completely separate researcher's dataset about the same cities link automatically via shared IRIs for city names, without any negotiated integration project between the two data owners.

## OWL: Ontologies and Reasoning

- **OWL (Web Ontology Language)** lets you formally define classes, subclass relationships, properties (including domain/range constraints), and logical axioms on top of an RDF graph — e.g., "every `Employee` is a `Person`," "the `manages` property's domain is `Manager` and range is `Employee`," or "no `Person` can be their own `parent`."
- This formal structure enables **inference**: an OWL reasoner can derive new facts that were never explicitly asserted (if `Employee` is a subclass of `Person`, then anything typed as `Employee` is automatically inferable as a `Person`) and can detect logical inconsistencies (two asserted facts that can't both be true given the ontology's axioms).
- Real-world example: an insurance company's ontology defined "PolicyHolder" and "Beneficiary" as disjoint classes (no individual can be both with respect to the same policy); an OWL reasoner flagged a data-entry error where the same person had been asserted as both roles on one policy, catching a real data-quality bug purely from the ontology's logical structure, without any custom validation code being written for that specific case.

## SPARQL and SHACL

- **SPARQL** is the standard query language for RDF graphs, structurally similar in spirit to SQL but pattern-matching over triples instead of rows.

```sparql
SELECT ?company ?founded
WHERE {
  ?company <http://example.org/acquiredBy> <http://example.org/Company_A> .
  ?company <http://example.org/foundedIn> ?founded .
}
```

- **SHACL (Shapes Constraint Language)** validates that an RDF graph conforms to expected structural rules (e.g., "every `Person` must have exactly one `birthDate`") — distinct from OWL, which is about logical inference, not validation; SHACL is the RDF-world equivalent of a JSON Schema or a database constraint.
- Real-world example: a data-integration pipeline ingesting RDF from multiple external partners uses SHACL to reject any incoming batch where required properties are missing or have the wrong cardinality, before that malformed data ever reaches the production graph.

## Neo4j and Cypher

- Neo4j is the most widely used property-graph database, queried with **Cypher**, a declarative, pattern-matching query language designed to read like an ASCII-art description of the graph pattern you're looking for.

```cypher
// Find companies acquired by Company A, and what they make
MATCH (target:Company)-[:ACQUIRED_BY]->(acquirer:Company {name: "Company A"})
MATCH (target)-[:MAKES]->(product:Product)
RETURN target.name, product.name
```

- As of Neo4j 2025.06, Cypher itself forked into two tracks: **Cypher 25** (where all new features land going forward) and **Cypher 5** (frozen — no new features, maintained for backward compatibility). Worth knowing this exists so an unfamiliar version reference in a job posting or codebase doesn't catch you off guard.

- Neo4j's **Graph Data Science (GDS) library** adds graph algorithms (community detection, centrality, pathfinding, node embeddings/graph neural network support) directly on top of the stored graph — useful for tasks like finding influential nodes, detecting clusters/communities, or generating graph embeddings for downstream ML.
- Real-world example: a fraud-detection team used Neo4j GDS's community-detection algorithm to surface a cluster of accounts connected through shared devices/addresses that no single-account risk score would have flagged in isolation — the fraud pattern was only visible at the graph-structure level.

## Building a Knowledge Graph Pipeline: Entity Extraction, Normalization, Relationship Discovery

```
raw text/documents -> NER (entity extraction) -> entity normalization/linking ->
relation extraction -> triple/graph construction -> load into graph store -> validate
```

- **Entity extraction (NER)**: identify mentions of entities (people, organizations, products) in unstructured text, using either a traditional NER model (e.g., spaCy) or an LLM prompted/fine-tuned for extraction.
- **Entity normalization / entity resolution**: the same real-world entity often appears under different surface forms ("IBM," "International Business Machines," "I.B.M.") across documents — normalization/resolution merges these into one canonical node instead of creating duplicate nodes for the same thing. This is frequently the hardest and most error-prone step in the whole pipeline.
- **Relationship discovery**: extract the typed relationships between recognized entities (e.g., "acquired," "works for," "located in"), either via rule-based patterns, a supervised relation-extraction model, or an LLM prompted to output structured (subject, relation, object) triples directly from text.
- Real-world example: an enterprise building a knowledge graph from thousands of press releases initially created three separate nodes for "Meta," "Meta Platforms," and "Facebook, Inc." because entity resolution wasn't implemented, fragmenting the graph's connectivity and silently undercounting that company's actual relationship count until a normalization pass merged them.

```python
# Conceptual LLM-based extraction step
extraction_prompt = """
Extract (subject, relation, object) triples from the text below.
Use canonical entity names where possible. Output as JSON list.
TEXT: {document_text}
"""
triples = llm.generate(extraction_prompt, response_schema=triple_list_schema)
for s, r, o in triples:
    graph_db.merge_node(s)
    graph_db.merge_node(o)
    graph_db.merge_relationship(s, r, o)
```

## Entity Resolution in Depth

- Entity resolution (also called record linkage or deduplication) typically combines: blocking (cheaply narrowing candidate pairs to compare, since comparing every entity to every other entity doesn't scale), similarity scoring (string similarity, embedding similarity, or a trained classifier) between candidate pairs, and a clustering/merge step that groups matched pairs into canonical entities.
- Real-world example: a customer-data knowledge graph used embedding-based similarity plus a trained classifier to merge "Jon Smith, 123 Main St" and "Jonathan Smith, 123 Main Street" into one canonical customer node, while correctly keeping "Jon Smith" at a different address as a separate node — a purely exact-string-match approach would have either merged unrelated people or missed genuine duplicates.

## Integrating LLMs with Knowledge Graphs

There are two main directions of integration, and a mature system typically uses both:

- **Text-to-graph (construction)**: using an LLM to extract entities and relationships from unstructured text to build or enrich a knowledge graph (as shown above) — this is how LLMs help *build* the graph.
- **Graph-to-text (retrieval and reasoning, "GraphRAG")**: using the knowledge graph as a grounding/retrieval source for LLM generation — instead of (or alongside) vector similarity search over unstructured chunks, the system retrieves a relevant subgraph (entities, relationships, and their properties) and passes that structured context to the LLM. This is especially valuable for questions that require multi-hop reasoning across explicitly connected entities, or "holistic" questions that summarize themes across an entire corpus, both of which plain vector-chunk RAG structurally struggles with (see this kit's RAG topic).
- **Text-to-Cypher / text-to-SPARQL**: a specialized case of the natural-language-querying pattern (see this kit's NLQ topic) where the target query language is a graph query language instead of SQL — the same core challenges apply (schema/ontology linking, execution-feedback self-correction, read-only execution, execution-accuracy evaluation), with the added wrinkle that graph query languages express traversal patterns (variable-length paths, pattern matching) that have no direct SQL equivalent.
- Real-world example: an internal "ask about our org chart and project history" assistant answers "which engineers have worked on every project that touched the payments system" by translating the question into a Cypher graph-pattern query with path constraints, something a flat vector-search-over-documents approach could not answer at all, because the answer depends on graph connectivity, not textual similarity to any single document.

- **A note on tooling status (know this before an interview)**: Microsoft's GraphRAG project (the paper and reference implementation that popularized the term) is now described by its own maintainers as largely in maintenance mode — bug/security fixes only, no new features — with the explicit rationale that frontier model capabilities have changed substantially since its mid-2024 release. **LightRAG** has emerged as the more actively developed, more-starred alternative, positioning itself as a lower-cost, more efficient graph-RAG framework. A senior answer should name both and be ready to discuss the tradeoff, rather than presenting "GraphRAG" as one static, currently-cutting-edge thing. Similarly, Neo4j's own GenAI tooling is not one monolithic product — it's at least three separately-maintained projects at different maturity levels: the `neo4j-graphrag-python` package (flagship, actively maintained), the LLM Knowledge Graph Builder (a hosted/self-deployable product for turning documents into a graph), and `text2cypher` (a smaller, lower-activity Neo4j Labs research repo for NL-to-Cypher datasets/fine-tuning) — worth naming all three distinctly rather than treating "Neo4j GraphRAG" as a single offering.

## Reasoning Over Knowledge Graphs with LLMs

- Multi-hop reasoning: some questions require traversing several relationship hops ("who is the manager of the person who approved this expense report's original submitter's team"). An LLM-driven agent can decompose this into a sequence of graph queries, using each result to inform the next query — analogous to the multi-hop/agentic RAG pattern in this kit's RAG topic, but operating over explicit graph structure instead of implicit chunk similarity.
- Ontology-guided reasoning: because OWL ontologies formally define class hierarchies and relationship constraints, an LLM can be given the ontology schema as context to generate more accurate queries and to explain *why* an answer follows from the graph (citing the specific path of relationships traversed) — a form of grounding that's more auditable than citing a raw text chunk, because the reasoning path is a structured, inspectable graph path.
- Real-world example: a compliance-question-answering tool over a regulatory knowledge graph can show its exact reasoning chain ("Regulation X applies to Entity Type Y because Y is a subclass of Z, and Regulation X's scope includes Z") as a literal graph path, giving auditors a verifiable trail that a purely text-based RAG answer's citation to a paragraph could not provide with the same precision.

## Evaluation Frameworks for AI-Generated Outputs (KG Context)

Evaluating AI-generated outputs in a knowledge-graph-augmented system spans both the graph-construction side and the graph-augmented-generation side:

- **Accuracy**: for extraction, precision/recall of extracted entities and relationships against a human-labeled gold set; for generation, whether the final natural-language answer correctly reflects what the graph actually contains.
- **Groundedness / faithfulness**: does every claim in a generated answer trace back to an actual path/fact in the graph, analogous to RAG faithfulness scoring but checked against structured graph facts rather than unstructured text chunks — often easier to verify automatically than text-based faithfulness, precisely because graph facts are structured and can be checked programmatically (does this exact triple exist in the graph) rather than requiring a fuzzy NLI/LLM-judge comparison.
- **Completeness**: for graph construction, what fraction of the true relationships in the source text were actually captured (a recall-style metric on the extraction pipeline); for generation, whether an answer omitted relevant connected facts the graph actually contains.
- **Consistency**: repeated runs of the same extraction or query-answering pipeline on the same input should produce the same (or compatibly merged) result — inconsistency in entity resolution or extraction output over time silently degrades graph quality even when any single run looks reasonable in isolation.
- **Hallucination detection**: specifically, does the generated answer assert a relationship or fact that does not exist in the graph at all, or does it correctly decline to answer when the graph has no relevant path? This is checkable with high precision in a KG-augmented system compared to unstructured RAG, because you can programmatically verify whether a claimed triple exists.
- Real-world example: an evaluation harness for a GraphRAG compliance assistant automatically checks every claimed relationship in a generated answer against the actual graph (a simple, deterministic existence check per triple) as its primary hallucination-detection signal, only falling back to LLM-as-judge scoring for aspects that can't be reduced to a structured fact check, like overall coherence or completeness of coverage.

## Ontology Design Best Practices

- Start from existing, widely-adopted ontologies/vocabularies where they fit (e.g., schema.org, FOAF, domain-specific standards) rather than designing from scratch, both to save effort and to maximize interoperability with external data.
- Model relationships, not just entities, as first-class citizens with their own well-defined semantics and, where useful, their own properties (a `WORKED_ON` relationship might carry a `role` and `date_range` property rather than requiring a separate join entity).
- Version the ontology explicitly and treat schema changes with the same discipline as an application database migration — a knowledge graph's flexibility doesn't mean its schema is exempt from change management, especially once other teams or an LLM extraction pipeline depend on specific relationship-type names.
- Real-world example: a team that renamed a core relationship type from `EMPLOYED_BY` to `WORKS_AT` without a migration plan silently broke every downstream Cypher query and every LLM few-shot example referencing the old name — the fix going forward was treating relationship-type renames as breaking schema changes requiring a coordinated rollout, not a quick find-and-replace.

## Common Production Failure Modes

- **Duplicate/fragmented entities**: entity resolution gaps create multiple nodes for the same real-world entity, silently fragmenting the graph's connectivity and undercounting true relationship density (see the "Meta/Meta Platforms/Facebook" example above).
- **Ontology drift**: the schema/ontology evolves informally over time without versioning discipline, so different parts of the graph (or different pipeline runs) use inconsistent relationship types or property names for the same concept.
- **Stale graph vs. live source data**: like RAG's stale-index failure mode, a knowledge graph built from a snapshot of source documents/systems can silently diverge from current reality if there's no defined re-ingestion/update process.
- **LLM-generated Cypher/SPARQL that's syntactically valid but traverses the wrong relationship path**: the graph-query equivalent of a syntactically-valid-but-semantically-wrong SQL query, often caused by ontology/schema-linking ambiguity (multiple relationship types that sound similar) rather than query-language syntax errors.
- Real-world example: a knowledge graph that had accumulated three different relationship types over time all informally meaning "reports to" (`REPORTS_TO`, `MANAGED_BY`, `SUPERVISED_BY`, added by different teams at different times) caused an org-chart query to silently undercount management chains depending on which relationship type happened to be used for a given employee — a direct consequence of ontology drift without a single source of truth for relationship semantics.

## Quick Gotchas Worth Naming in an Interview

- RDF/OWL and property graphs (Neo4j) are not competing on "which is better" in the abstract — they solve different problems (standards-based interoperability and formal reasoning vs. flexible, fast, application-embedded graph querying), and a senior answer should name the actual tradeoff rather than a blanket preference.
- Entity resolution, not extraction accuracy, is usually the real bottleneck in production knowledge-graph pipelines — a system can have excellent NER/relation-extraction and still produce a low-quality graph if duplicate entities aren't merged.
- GraphRAG's advantage over plain vector RAG is specifically for multi-hop and "holistic" (corpus-wide theme) questions — it's not a strict upgrade for simple, localized fact-lookup questions, where plain vector retrieval is often faster and simpler.
- Hallucination detection is genuinely easier to make deterministic in a KG-augmented system than in unstructured RAG, because a claimed relationship can be checked against the graph as a simple existence query — this is a concrete, defensible advantage worth naming explicitly in an interview about evaluation frameworks.
- Treat ontology/schema changes (renaming a relationship type, changing a class hierarchy) as breaking changes requiring the same migration discipline as a relational schema change — the graph's structural flexibility does not mean its meaning is exempt from versioning and change management.
