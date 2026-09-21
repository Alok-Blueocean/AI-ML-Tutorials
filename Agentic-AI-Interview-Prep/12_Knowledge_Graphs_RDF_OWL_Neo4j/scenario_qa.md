# Knowledge Graphs (RDF, OWL, Property Graphs, Neo4j) — Scenario-Based Q&A

**Situation:** After ingesting thousands of press releases into a knowledge graph, an analyst notices the graph has three separate, unconnected nodes for what is clearly the same company, each with a slightly different name. What would you do and why?

Model answer: This is a classic entity-resolution gap, not an extraction-accuracy problem — the NER/relation-extraction steps may be working fine, but without a normalization/resolution step, every surface-form variant of an entity's name creates a new node, silently fragmenting the graph's real connectivity. Fix it by adding an entity-resolution stage (blocking to narrow candidate pairs, then similarity scoring via string and/or embedding similarity, then a merge step) between extraction and graph loading, and backfill the existing graph by running resolution against already-loaded nodes. Add an ongoing duplicate-rate metric to production monitoring so this doesn't silently recur as new documents are ingested.

---

**Situation:** A GraphRAG-based compliance assistant confidently states that "Regulation X applies to Company Type Y" but this relationship doesn't actually exist anywhere in the knowledge graph — the model appears to have inferred it rather than retrieved it. What would you do and why?

Model answer: Treat this as a hallucination in the generation step, and take advantage of the fact that graph-grounded hallucination is far easier to detect deterministically than in unstructured RAG — build an automated checker that verifies every relationship claimed in a generated answer against an actual existence query in the graph, and fail/flag any answer containing a claimed triple that isn't present. Longer term, tighten the generation prompt to explicitly instruct the model to state only relationships present in the retrieved subgraph and to say "no such relationship exists in the graph" rather than inferring a plausible-sounding one, and add this exact case to a permanent regression set.

---

**Situation:** Your team has been building out a knowledge graph for two years, and different contributors have independently added `REPORTS_TO`, `MANAGED_BY`, and `SUPERVISED_BY` relationship types that all informally mean the same thing. An org-chart query is now silently undercounting management chains depending on which relationship type happens to be used for a given employee. What would you do and why?

Model answer: This is ontology drift — the schema was never formally governed, so semantically identical relationships accumulated different names over time with no single source of truth. Treat this as a schema migration, not a quick patch: pick one canonical relationship type, write a migration to consolidate the others into it (or add a mapping layer that treats them as equivalent at query time while the migration is in progress), and going forward require new relationship types to go through a lightweight review process (an ontology owner or a documented schema registry) before being introduced. Communicate the change to every downstream consumer (dashboards, LLM few-shot examples referencing relationship names) since a silent rename would break them the same way this drift did.

---

**Situation:** A stakeholder asks why you're proposing RDF/OWL for a new project instead of "just using Neo4j like everyone else," given the team has more Neo4j experience.

Model answer: Frame it as a fit-for-purpose decision, not a default preference. RDF/OWL is the stronger choice specifically when interoperability with external, independently-built datasets matters (a shared vocabulary and standard query language let disparate parties' data merge without a bespoke integration project), or when formal logical reasoning/consistency-checking over the schema itself is a real requirement. If this project's data lives entirely inside the organization, doesn't need external interoperability, and the priority is fast iteration and simpler application-embedded querying, Neo4j's property-graph model is the better fit, and the team's existing expertise is a legitimate additional factor. Recommend making the decision explicitly on these criteria rather than by default in either direction.

---

**Situation:** An LLM-generated Cypher query for "which employees have never worked on a project led by their own manager" runs without error and returns a plausible-looking list, but on manual review, half the results are wrong because the query silently used the wrong relationship direction. What would you do and why?

Model answer: This is the knowledge-graph equivalent of a syntactically valid but semantically wrong SQL query — nothing about the failure looks broken from the output alone. Build an execution-accuracy-style evaluation set of representative graph questions with known-correct results, and specifically include queries with negation and relationship-direction ambiguity (a well-known trap, similar to anti-join patterns in SQL), since these are where LLMs most often default to a simpler but wrong pattern. Add curated few-shot Cypher examples covering exactly this "never did X" negation/anti-pattern for your schema, and treat relationship direction as something the schema documentation given to the model should state unambiguously (e.g., explicitly noting `(:Employee)-[:MANAGES]->(:Employee)` direction) rather than assuming the model will infer it correctly.

---

**Situation:** Leadership wants a hard number on whether your knowledge-graph extraction pipeline (built from LLM-based entity/relationship extraction) is "good enough" to stop having a human review every batch.

Model answer: Build a held-out gold-labeled evaluation set (a sample of documents with human-annotated ground-truth entities and relationships) and measure precision and recall of the extraction pipeline against it, separately from measuring the entity-resolution step's duplicate rate, since these are different failure modes that a single blended accuracy number would obscure. Track both over time as a regression suite, since model updates or shifts in document format/style can silently move these numbers. Recommend keeping human review for a sampled percentage of batches even after automation looks strong, calibrated to the measured error rate and the cost of a bad fact entering the graph, rather than eliminating review entirely based on a one-time evaluation snapshot.

---

**Situation:** Your knowledge graph was built from a snapshot of source documents six months ago, and a user just got an answer from your GraphRAG assistant reflecting an organizational structure that changed three months ago. What would you do and why?

Model answer: This is the knowledge-graph equivalent of RAG's stale-index failure mode — the underlying source of truth changed, but there's no defined re-ingestion/refresh process keeping the graph current. Diagnose whether this is a one-off gap or a structural process gap (no scheduled or event-driven re-ingestion at all), and fix it at the process level: define a re-ingestion trigger (scheduled batch refresh, or event-driven updates when source systems change), and add a "graph last-updated" timestamp surfaced in the assistant's answers so users have some visibility into potential staleness even before the refresh cadence is tightened.

---

**Situation:** A junior team member proposes skipping formal ontology design entirely and just letting the LLM extraction pipeline invent whatever entity types and relationship names seem natural for each new batch of documents, to move faster.

Model answer: Push back with a concrete cost argument: without a governed set of entity/relationship types, you get exactly the ontology-drift problem described earlier — semantically identical concepts get inconsistent names across batches, breaking downstream queries and LLM few-shot examples silently over time. Recommend a lightweight middle ground appropriate for the team's speed needs: define a core, versioned set of entity and relationship types up front (even a small one), have the extraction pipeline map free-form LLM output onto that controlled vocabulary (flagging anything that doesn't fit for human review rather than silently inventing new types), and evolve the ontology deliberately as real new needs are identified — faster than heavyweight upfront ontology engineering, but without the compounding technical debt of no governance at all.

---

**Situation:** A data scientist on your team argues that since GraphRAG showed strong results in a recent internal pilot, all RAG use cases should be migrated to a graph-based approach.

Model answer: Push back on treating this as a strict upgrade. GraphRAG's real advantage is for multi-hop reasoning and "holistic," corpus-wide theme questions that plain vector-chunk retrieval structurally cannot answer well — it is not automatically better for simple, localized fact-lookup questions, where building and maintaining a knowledge graph (with its entity-resolution and ontology-governance overhead) adds real cost and complexity for no corresponding benefit. Recommend evaluating on the actual question distribution the use case needs to serve: if most real user questions are single-fact lookups, plain vector RAG is likely simpler and cheaper to maintain; if a meaningful fraction require multi-hop or aggregate reasoning across explicitly connected entities, a graph-augmented approach earns its keep specifically there.
