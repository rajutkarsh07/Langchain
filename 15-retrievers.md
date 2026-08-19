# Retrievers

A **Retriever** is a LangChain abstraction whose single job is:

> **Given a query, return relevant documents.**

That is the whole contract. Everything else — vector similarity, keyword matching, multi-query expansion, hybrid ranking — is an implementation choice hidden behind that interface.

## Where retrievers fit

```text
Documents
    ↓
Chunks
    ↓
Embeddings
    ↓
Vector Store
    ↓
Retriever          ← this chapter
    ↓
Relevant Documents
    ↓
LLM
    ↓
Answer
```

The retriever sits between raw storage/search infrastructure and the LLM. It converts a natural-language question into the specific documents the LLM should read.

## The retriever contract

Every retriever is a [Runnable](./09-runnables.md):

```python
retriever = vector_store.as_retriever(search_kwargs={"k": 4})

docs = retriever.invoke("How can I change my password?")
```

Input: a query string.
Output: a list of `Document`s.

Because retrievers are Runnables, they plug into chains, pipelines, and agents the same way any other component does.

## Retriever vs Vector Store — restated

This is the single most important distinction in the retrieval part of LangChain.

### Vector Store

Concrete infrastructure that stores vectors and answers similarity queries:

```text
Vector Store
   ├── Store vectors
   ├── Search vectors
   └── Return documents
```

### Retriever

An **abstraction** — an interface for "given a query, return documents":

```text
Retriever
   ↓
"Given a query, retrieve relevant documents"
```

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

The retriever is the **interface**; the vector store is one possible **backend**.

> **Retriever ≠ Vector Store.** A vector store is one possible mechanism used by a retriever.

## A retriever does not have to use vectors

This is the key insight — and where the abstraction pays off.

Different retrievers can use completely different strategies:

### Vector retriever

```text
Query
 ↓
Embedding
 ↓
Vector similarity
 ↓
Documents
```

Uses a vector store under the hood.

### BM25 retriever

```text
Query
 ↓
Keyword matching
 ↓
Documents
```

BM25 is a classical information-retrieval algorithm. It scores documents based on term frequency and rarity — no embeddings involved.

Suppose the user searches:

```text
"HTTP 401 authentication error"
```

Exact terms like `HTTP`, `401`, and `authentication` really matter. A keyword-based retriever can handle this precisely, while a pure vector retriever might drift semantically.

### Multi-query retriever

```text
User Query
    ↓
Generate multiple queries (LLM)
    ↓
Q1 ──→ Search
Q2 ──→ Search
Q3 ──→ Search
    ↓
Combine results
```

Uses an LLM to generate multiple rephrasings of the original query, searches with each, and merges the results.

### Parent document retriever

Searches over small chunks (which give precise similarity) but returns their larger parent sections (which give more context).

```text
Query
 ↓
Search small chunks
 ↓
Find relevant chunk
 ↓
Find parent
 ↓
Return larger context
 ↓
LLM
```

Solves the trade-off between **precise retrieval** and **sufficient context**.

## Why the abstraction matters

Because everything conforms to the retriever interface, you can:

- Swap a vector retriever for a keyword retriever without touching downstream code.
- Combine strategies (hybrid retrieval — see below) behind a single retriever.
- Insert re-ranking, filtering, or query expansion transparently.
- Use retrievers inside chains, agents, or any Runnable pipeline.

Your LLM-side code does not care **how** the retriever finds documents — only that it does.

## Hybrid retrieval

Production RAG systems often use **hybrid retrieval** — combining multiple strategies:

```text
        User Query
           │
   ┌───────┴───────┐
   ↓               ↓
Vector Search   BM25 Search
   │               │
   └───────┬───────┘
           ↓
        Merge / Re-rank
           │
           ▼
     Top-K Documents
```

Vector search catches semantic overlap ("password reset" ≈ "credential recovery"). Keyword search catches exact terms that matter ("HTTP 401", "SKU 8842"). Hybrid retrieval gets both.

From the outside, the whole thing is still just a Retriever.

## Multi-query retrieval — why it helps

User questions are often ambiguous or under-specified:

> "How does authentication work?"

A single embedding of that query might miss documents that describe the same concept from a different angle. Multi-query retrieval lets an LLM generate variations:

```text
How does user authentication work?
How are users authenticated?
What authentication mechanism is used?
How are authentication tokens handled?
```

Then search using all of them:

```text
                User Query
                    │
                    ▼
                   LLM
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
         Q1        Q2        Q3
          │         │         │
          ▼         ▼         ▼
       Search    Search    Search
          │         │         │
          └─────────┼─────────┘
                    ▼
              Combined Results
```

This typically improves **recall** — you find more of the relevant documents that a single-query search would have missed.

## Parent document retrieval — trade-off resolved

Small chunks are great for precision (a focused embedding matches a focused query). But small chunks alone often lack enough context for the LLM to answer.

Parent document retrieval:

- **Search** using small chunks — precise.
- **Return** the larger parent section — rich enough for the LLM to reason with.

```text
Query
 ↓
Search small chunks
 ↓
Find relevant chunk
 ↓
Find parent
 ↓
Return larger context
 ↓
LLM
```

This is a good example of why the retriever abstraction matters — the caller just gets useful documents, without knowing about the two-stage machinery inside.

## Choosing a retriever

| Situation                                   | Retriever                             |
| ------------------------------------------- | ------------------------------------- |
| General semantic Q&A                         | Vector retriever                      |
| Exact terms and codes matter (e.g., logs)   | BM25 / keyword retriever              |
| Best of both worlds                          | Hybrid retriever                      |
| Ambiguous user questions                    | Multi-query retriever                 |
| Need precise search + enough context        | Parent document retriever             |

Often the right answer is a combination — retrieval quality is a tuning problem, not a one-shot choice.

## Retriever vs RAG

Another distinction worth being explicit about.

| Concept       | What it does                                              |
| ------------- | --------------------------------------------------------- |
| **Retriever** | Given a query, returns relevant documents                 |
| **RAG**       | The full pattern: retrieve → augment prompt → generate    |

A retriever is a **component**. RAG is a **pattern** built out of a retriever, a prompt, and an LLM. See the [RAG](./16-rag.md) chapter next.

## Practical considerations

- Always measure retrieval quality separately from LLM quality. If retrieval returns the wrong documents, even a perfect LLM can't answer correctly.
- Log the retrieved documents. Debugging RAG is 80% "what did the retriever return?"
- Use metadata filtering to keep results scoped.
- Consider re-ranking if you have many candidate documents.
- Start simple — a vector retriever with sensible chunking usually goes further than a complex hybrid setup with bad chunks.

## Where retrievers sit in the mental model

Think of building a digital library:

> **Retriever = "Given the reader's question, find the sections they should read."**

Whether the librarian uses semantic index, keyword catalog, or a rephrased search — the reader doesn't need to know. They just need the right sections.

## Key takeaways

- A retriever is defined by one thing: query in, relevant documents out.
- **Retriever ≠ Vector Store.** Vector stores are one possible backend.
- Different retrievers use different strategies: vector, BM25, multi-query, parent document, hybrid.
- Because retrievers are Runnables, they slot into any LangChain pipeline.
- Retrievers are a component; **RAG** is the pattern that uses them.

### Final mental model

```text
              User Query
                  │
                  ▼
              Retriever
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
 Vector         BM25         Multi-query
 Search        Search         Search
    │             │             │
    └─────────────┼─────────────┘
                  ▼
           Relevant Documents
                  ▼
                 LLM
```

> **A retriever is a Runnable that answers one question: "given this query, what should the LLM read?"**

---

Next: [Retrieval-Augmented Generation (RAG)](./16-rag.md) — the pattern that ties loaders, splitters, embeddings, stores, and retrievers into a complete grounded-answer pipeline.
