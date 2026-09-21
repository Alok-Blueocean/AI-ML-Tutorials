# Vector Databases

## What Is a Vector Database?

When you turn text, images, or audio into embeddings (numerical vectors that capture meaning), you need somewhere to store and search those vectors efficiently. A **vector database** is a system built specifically for this: it stores millions or billions of high-dimensional vectors and can quickly find the ones most "similar" to a given query vector.

This is the backbone of **semantic search** and **Retrieval-Augmented Generation (RAG)**. Instead of matching exact keywords, you match meaning — "how do I reset my password" can retrieve a document that says "steps to recover account access," even though the words barely overlap.

## Why It Matters

LLMs have two big limitations: they don't know your private/internal data, and they have limited context windows. Vector databases solve both:

- **Grounding LLMs in your data**: You embed your documents, store the vectors, and at query time retrieve the most relevant chunks to feed into the prompt (RAG). This reduces hallucination and lets the model answer from up-to-date, proprietary information it was never trained on.
- **Speed at scale**: Comparing a query against millions of vectors with brute-force math is slow. Vector databases use specialized indexing so searches return in milliseconds even at huge scale.
- **Beyond text**: The same idea works for images, audio, product catalogs, recommendation systems, fraud detection, and de-duplication — anywhere "find things similar to this" matters.

In short: if your LLMOps pipeline does RAG, semantic search, or recommendations, a vector database is almost always part of the stack.

## Main Concepts, in Plain Terms

**Embedding**: A list of numbers (a vector) produced by a model that represents the "meaning" of a piece of content. Similar meanings produce vectors that are close together in that numeric space.

**Similarity search**: Instead of exact matches, you ask "which stored vectors are closest to my query vector?" Closeness is usually measured with:
- **Cosine similarity** — angle between vectors (most common for text embeddings)
- **Euclidean (L2) distance** — straight-line distance
- **Dot product** — related to both, often used when vectors are normalized

**ANN (Approximate Nearest Neighbor) search**: Finding the *exact* closest vectors among billions is too slow. Vector databases use approximate algorithms (like HNSW — Hierarchical Navigable Small World graphs) that trade a tiny bit of accuracy for huge speed gains. This is the core trick that makes vector databases practical.

**Index**: The data structure built over your vectors to make ANN search fast (e.g., HNSW, IVF). You typically choose an index type and a few tuning parameters (like how many neighbors to consider) when you set up a collection.

**Collection / Index (the container)**: A named grouping of vectors, similar to a "table" in a relational database. Each vector usually has an ID, the embedding itself, and **metadata** (e.g., source document, date, category).

**Metadata filtering**: Real-world queries are rarely "just" semantic. You often need "find similar support tickets, but only from the last 30 days and only for Product X." Good vector databases let you combine similarity search with metadata filters in one query.

**Hybrid search**: Combining vector (semantic) search with traditional keyword/full-text search (like BM25), then merging the results. Useful because pure semantic search sometimes misses exact terms (like product codes or names) that keyword search catches easily.

## When Do You Actually Need One?

Not every project needs a dedicated vector database. Rules of thumb:

- **Small scale, prototyping** (a few thousand vectors): A simple library like FAISS in-memory, or even a plain array with cosine similarity, may be enough.
- **Production RAG, growing data, need for persistence, filtering, or multiple users/services**: This is where a real vector database earns its keep — it handles persistence, scaling, concurrent access, filtering, and updates without you rebuilding an index from scratch each time.
- **Millions+ vectors, low-latency requirements, or multi-tenant systems**: A dedicated vector database (or a vector-capable database extension) becomes close to mandatory.

## A Quick Comparison

| | Qdrant | Pinecone | Milvus | Weaviate |
|---|---|---|---|---|
| Type | Open-source, self-hosted or managed cloud | Fully managed cloud (SaaS only) | Open-source, self-hosted or managed cloud | Open-source, self-hosted or managed cloud |
| Written in | Rust | Proprietary | Go/C++ | Go |
| Standout trait | Fast, simple API, strong filtering support | Zero-ops, very easy to get started, mature managed service | Built for very large scale, highly configurable indexes | Built-in vectorization modules, GraphQL-style API, strong hybrid search |
| Good fit when | You want open-source control with production-grade performance | You want to skip infra management entirely | You need to scale to massive datasets with fine-grained index tuning | You want an all-in-one system with built-in embedding pipelines |

None of these is "best" in every situation — the right choice depends on your team's ops preferences (managed vs. self-hosted), scale, and whether you want the database to also handle embedding generation.

## A Simple Example (Qdrant)

Here's the basic shape of using a vector database in Python — create a collection, insert vectors with metadata, then search:

```python
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct

client = QdrantClient(url="http://localhost:6333")

# 1. Create a collection for 384-dimensional embeddings
client.create_collection(
    collection_name="support_docs",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

# 2. Insert a vector with metadata
client.upsert(
    collection_name="support_docs",
    points=[
        PointStruct(
            id=1,
            vector=[0.02, -0.13, 0.44, ...],  # your embedding, 384 numbers
            payload={"source": "faq.md", "topic": "billing"},
        )
    ],
)

# 3. Search for the closest vectors, filtered by metadata
results = client.search(
    collection_name="support_docs",
    query_vector=[0.01, -0.10, 0.40, ...],
    query_filter={"must": [{"key": "topic", "match": {"value": "billing"}}]},
    limit=5,
)
```

The pattern is nearly identical across Pinecone, Milvus, and Weaviate: define a collection/index with a vector dimension and distance metric, upsert vectors with IDs and metadata, then query with a vector (optionally plus filters) to get the top-k most similar results.

## Key Takeaways & Best Practices

- A vector database stores embeddings and answers "what's similar to this?" quickly, using approximate nearest neighbor (ANN) search instead of exact brute-force comparison.
- It's the core infrastructure piece behind RAG and semantic search in most LLMOps pipelines.
- Match your embedding model's dimensionality and distance metric (cosine, dot product, or L2) to how the database is configured — they must agree.
- Store useful metadata alongside vectors so you can filter results (by date, source, category, permissions, etc.) rather than relying on similarity alone.
- Consider hybrid search (semantic + keyword) when exact terms matter, such as product names, codes, or IDs.
- Choose managed (Pinecone) vs. self-hosted (Qdrant, Milvus, Weaviate) based on how much operational overhead your team wants to own.
- Don't over-engineer early: for small prototypes, an in-memory library may be enough; adopt a dedicated vector database when scale, persistence, or filtering needs grow.
- Re-index or re-embed your data when you change embedding models — vectors from different models are not comparable to each other.
