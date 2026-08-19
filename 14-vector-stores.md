# Vector Stores

A **Vector Store** stores embeddings and lets you search them by similarity. If [Embeddings](./13-embeddings.md) turn meaning into vectors, a vector store is the database that keeps those vectors and finds the closest ones to a given query.

## Where vector stores fit

```text
Documents
    ↓
Chunks
    ↓
Embeddings
    ↓
Vectors
    ↓
Vector Store      ← this chapter
    ↓
Retriever
    ↓
LLM
```

The vector store is the persistence layer for the RAG pipeline.

## What is a Vector Store?

A vector store keeps two things together:

- The **vector** for each chunk.
- The **document** (text + metadata) that the vector came from.

```text
Chunk 1 → [0.12, 0.43, ...]
Chunk 2 → [0.82, 0.11, ...]
Chunk 3 → [0.34, 0.92, ...]
Chunk 4 → [0.21, 0.56, ...]
```

Given a new vector (typically an embedded query), the store returns the stored vectors that are most similar to it — along with the associated documents.

Examples of vector stores commonly used with LangChain:

- Chroma
- FAISS
- Pinecone
- Qdrant
- Weaviate
- Milvus
- pgvector

They differ in scale, hosting model, filtering support, and performance — but they all expose the same conceptual interface.

## Why do we need a Vector Store?

Suppose the user asks:

> "How can I change my password?"

We convert the question into an embedding:

```text
Question
   ↓
Embedding Model
   ↓
[0.13, 0.44, 0.79, ...]
```

We then need to find the stored chunks whose vectors are **most similar** to this query vector. Doing that quickly, at scale, is what vector stores are built for.

```text
Query
  │
  ▼
Vector
  │
  ▼
Similarity Search
  │
  ├── Chunk 17 → 0.94 similarity
  ├── Chunk 83 → 0.89 similarity
  ├── Chunk 41 → 0.82 similarity
  └── Chunk 92 → 0.31 similarity
```

The top-scoring chunks become the retrieved context for the LLM.

## Similarity search

The fundamental operation is `similarity_search`:

```python
results = vector_store.similarity_search(
    "How can I change my password?",
    k=4
)
```

`k=4` means: **return the top 4 results**.

The exact similarity metric depends on the store and how you configure it. Common ones:

- **Cosine similarity** — angle between vectors (very common in RAG).
- **Euclidean distance** — straight-line distance.
- **Dot product** — proportional to cosine similarity when vectors are normalized.

You do not need to memorize the math. The key idea:

> **Compare the query vector with stored vectors and return the closest ones.**

## Ingestion — how vectors get into the store

Typical ingestion:

```text
Documents
    ↓
Text Splitter
    ↓
Chunks (with metadata)
    ↓
Embedding Model
    ↓
Vectors
    ↓
Vector Store  (stores vector + document + metadata)
```

Ingestion is usually done offline (or on a schedule) — separately from serving user queries.

## Query — how vectors come out

Typical retrieval at query time:

```text
User Query
    ↓
Embedding Model
    ↓
Query Vector
    ↓
Vector Store
    ↓
Top-K Chunks
    ↓
Prompt + Context
    ↓
LLM
```

The result is a list of `Document` objects, ranked by similarity.

## Metadata filtering

Most vector stores support filtering on metadata **in addition to** similarity. For example, given chunks tagged with a source and a language, you can restrict retrieval to:

- Documents from a specific source.
- A specific language.
- A specific date range.
- A specific user or tenant.

This is where the metadata you preserved during loading and splitting pays off:

```text
Query + filter
     ↓
Vector Store
     ↓
Similarity search restricted to matching metadata
     ↓
Top-K relevant chunks (all satisfying the filter)
```

Without metadata, you get similarity across your entire corpus, whether you wanted it or not.

## Vector Store vs Embedding Model

These are two of the most-confused components.

| Component            | Responsibility                                     |
| -------------------- | -------------------------------------------------- |
| **Embedding Model**  | Converts text into a vector                        |
| **Vector Store**     | Stores those vectors and searches them by similarity |

They are strictly complementary:

```text
Text
 ↓
Embedding Model → Vector → Vector Store → Similarity Search
```

An embedding model with no store gives you numbers you can't search. A vector store with no embeddings has nothing to store.

## Vector Store vs Retriever

This is the other common source of confusion — and one worth understanding well before the [Retrievers](./15-retrievers.md) chapter.

### Vector Store

Responsible for:

```text
Store vectors
Search vectors
Return documents
```

It is a specific piece of infrastructure with a specific search mechanism (usually similarity search over embeddings).

### Retriever

Responsible for:

```text
Given a query
      ↓
Retrieve relevant documents
```

It is an **abstraction** — a `Runnable` that takes a query and returns documents. **How** it retrieves them is an implementation detail.

Relationship:

```text
                Retriever
                    │
                    ▼
              Vector Store
                    │
                    ▼
            Relevant Documents
```

The retriever is the **interface for retrieval**; the vector store is one common way it fetches results — but not the only one. A retriever might use vector similarity, keyword search, hybrid strategies, or something entirely custom.

> **Retriever ≠ Vector Store.** A vector store is one possible mechanism a retriever can use.

## Practical considerations

- **Vector dimensionality** — the store must be configured to match the embedding model's output dimension. Mismatch = error or garbage results.
- **Consistency** — always use the same embedding model at ingestion and query time.
- **Persistence** — some stores are in-memory (great for prototyping), others are persistent databases (needed for production).
- **Scale** — millions of vectors need a store designed for that scale; a small demo can use an in-memory FAISS index.
- **Filtering** — support and performance vary widely between backends; check before committing to one.
- **Cost** — hosted stores may charge per stored vector or per query.

## Choosing a vector store

There is no single "best" store. A rough guide:

| Situation                                    | Consider                                    |
| -------------------------------------------- | ------------------------------------------- |
| Prototyping, small corpus                    | Chroma, FAISS (in-memory)                   |
| Self-hosted, production                      | Qdrant, Weaviate, Milvus, pgvector          |
| Managed / SaaS                               | Pinecone, hosted Qdrant/Weaviate            |
| Already have a Postgres database             | pgvector                                    |

Because LangChain wraps them behind the same abstraction, migrating stores later is usually cheaper than migrating embedding models.

## Where vector stores sit in the mental model

Think of building a digital library:

> **Vector Store = "Store the coordinates of every book section, and quickly find the ones nearest to any query."**

They are the retrieval infrastructure — the fast lookup that makes semantic search feasible at scale.

## Key takeaways

- A vector store persists embeddings and lets you search them by similarity.
- `similarity_search(query, k=N)` is the fundamental operation.
- Metadata filtering lets you narrow retrieval before ranking by similarity.
- Vector store ≠ embedding model; vector store ≠ retriever.
- The store is one possible backend for a retriever, not the retriever itself.
- Choice of store matters for scale, hosting model, and filtering — but LangChain's abstraction keeps switching relatively cheap.

### Final mental model

```text
        Vectors
           │
           ▼
     Vector Store
       │      │
       │      └── metadata filter
       ▼
  Similarity Search
       │
       ▼
  Top-K Documents
       │
       ▼
    Retriever
       │
       ▼
       LLM
```

> **Vector stores turn "find text with similar meaning" into a fast lookup. Retrievers use them (or something else) to serve the actual queries.**

---

Next: [Retrievers](./15-retrievers.md) — the abstraction above vector stores that unifies all retrieval strategies.
