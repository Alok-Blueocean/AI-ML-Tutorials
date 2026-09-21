# Books — Module 08: LLM Latency, Cost, and Deployment Strategy

---

### AI Engineering: Building Applications with Foundation Models
**Author:** Chip Huyen
**Publisher:** O'Reilly Media, 2025
**What it teaches:** The most directly relevant book-length treatment of this module's subject matter available today. Chip Huyen (previously at NVIDIA and Snorkel AI, and author of *Designing Machine Learning Systems*) devotes substantial chapters to inference optimization (batching strategies, quantization, distillation trade-offs) and to AI engineering architecture, including the economics of building on top of foundation models — model selection under cost/latency/quality constraints, the build-vs-buy decision between API providers and self-hosting, and evaluation of total cost of ownership once you count engineering time, not just per-token price.
**Difficulty:** Intermediate to Advanced — assumes the reader is already comfortable with transformer basics and production software engineering; this is not an ML-fundamentals book.
**Estimated reading time:** Full book: 15–20 hours. The chapters specifically on inference optimization and AI engineering architecture: 3–4 hours combined.
**Why it matters for this module:** This is the closest thing to a required companion text for this module. Where the module's transcripts give you a fast-moving practitioner's tour (case studies, cost levers, a serving-config pattern), this book gives you the surrounding conceptual scaffolding — why these particular levers exist, how to reason about model selection systematically, and how to structure a cost-and-latency argument for a technical leadership audience. Recommended as the first "go deeper" reading after finishing this module.

---

### Designing Machine Learning Systems
**Author:** Chip Huyen
**Publisher:** O'Reilly Media, 2022
**What it teaches:** A holistic, deployment-and-operations-first view of ML systems, predating the LLM-specific wave but foundational for the *general* engineering discipline this module applies to LLMs specifically: online vs. batch prediction trade-offs, model compression, and the monitoring/retraining decisions that surround a deployed model. Its chapter on model deployment covers the same underlying latency-vs-throughput tension (serving one request fast vs. serving many requests cheaply) that this module teaches through the lens of TTFT and batching — just for classical ML models rather than autoregressive decoding.
**Difficulty:** Intermediate.
**Estimated reading time:** Full book: 12–15 hours. Deployment-focused chapters: 2–3 hours.
**Why it matters for this module:** Read this if you want the "why does this general pattern exist" grounding — the batching, caching, and online/offline serving trade-offs this module teaches for LLMs specifically are instances of patterns this book already established for ML systems in general. Most useful for engineers newer to production ML who haven't yet internalized why serving-side engineering is its own discipline distinct from model training.

---

### LLM Engineer's Handbook
**Authors:** Paul Iusztin, Maxime Labonne
**Publisher:** Packt Publishing, 2024
**What it teaches:** An end-to-end, hands-on walkthrough of building and deploying an LLM-based system: data pipelines, fine-tuning, RAG, deployment to AWS, and production monitoring, framed around LLMOps best practices and Domain-Driven Design. Includes practical deployment chapters covering inference optimization and cloud deployment cost/architecture decisions, with a full companion GitHub repository (see `github.md`).
**Difficulty:** Intermediate — hands-on and code-first rather than conceptual, assumes working Python and basic cloud (AWS) familiarity.
**Estimated reading time:** Full book: 10–14 hours; deployment/inference-optimization chapters: 2 hours.
**Why it matters for this module:** Complements this module's cost/latency framework with a concrete, code-backed deployment narrative — useful if you learn best by reading a full working system end-to-end rather than isolated code samples. Its AWS deployment chapters are a good real-world anchor for the module's "hybrid architecture case study."

---

### Designing Data-Intensive Applications
**Author:** Martin Kleppmann
**Publisher:** O'Reilly Media, 2017 (2nd edition in progress as of 2026)
**What it teaches:** Not an LLM book at all — this is the standard reference for the underlying distributed-systems and caching theory (cache invalidation, TTL and eviction policies, consistency trade-offs, replication) that a production semantic-caching layer (Redis-backed or otherwise) is built on top of. Its chapters on caching, replication, and partitioning give the vocabulary and mental models (cache stampede, thundering herd, eviction policy trade-offs like LRU vs. LFU) that the module's Redis semantic-caching section assumes some familiarity with.
**Difficulty:** Intermediate to Advanced — a systems-engineering classic, not written for an ML audience specifically.
**Estimated reading time:** Full book: 20+ hours; the caching/replication chapters relevant here: 2–3 hours.
**Why it matters for this module:** If the phrase "68% cache hit rate" in this module's transcripts makes you want to ask "hit rate under what eviction policy, and what happens when the cache is cold after a deploy?" — this is the book that gives you the vocabulary to ask (and answer) that question rigorously. Recommended specifically for engineers who want to reason about caching correctness and failure modes at senior-interview depth, not just implement the happy path.

---

## Reading order recommendation

1. Finish this module's transcripts and code samples first — they are your fast, concrete on-ramp.
2. Read the *AI Engineering* chapters on inference optimization and architecture for the conceptual scaffolding around cost/latency trade-offs.
3. If you're newer to production ML generally, backfill with *Designing Machine Learning Systems*'s deployment chapter.
4. If you want a full worked system to study end-to-end, work through the *LLM Engineer's Handbook* deployment chapters and its companion repo.
5. If you're specifically going deep on the caching layer (for a senior-level system-design interview, for example), read the relevant *Designing Data-Intensive Applications* chapters to be able to defend eviction-policy and consistency choices, not just describe them.
