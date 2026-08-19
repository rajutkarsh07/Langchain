# Embeddings

An **embedding** converts text into a numerical vector that represents its **meaning**. Embeddings are the mechanism that lets a machine compare two pieces of text by what they mean, not just by which words they contain. They are the foundation of semantic search, vector stores, retrievers, and ultimately RAG.

## Where embeddings fit

In the RAG pipeline:

```text
Documents
    ↓
Chunks (from splitter)
    ↓
Embeddings         ← this chapter
    ↓
Vectors
    ↓
Vector Store
    ↓
Retriever
    ↓
LLM
```

Embeddings sit between the splitter and the vector store. Without them, vector stores and retrievers would have nothing to store or search.

## What is an embedding?

An embedding model takes text and returns a vector of numbers:

```text
Text
 ↓
Embedding Model
 ↓
Vector
```

For example:

```text
"How does authentication work?"
              ↓
       Embedding Model
              ↓
[0.13, -0.42, 0.77, 0.21, ...]
```

Each element of the vector corresponds to a dimension in a high-dimensional space. Text with similar meaning ends up **near** in that space; text with different meaning ends up **far**.

## Why embeddings are useful

Traditional search compares strings. Two documents with the same meaning but different words look completely different to a keyword matcher:

```text
Query:
"How does authentication work?"

Passage:
"The system verifies a user's identity before granting access."
```

A keyword search sees no overlap. An embedding-based search sees these as close, because their **meanings** are close.

### Example

```text
Text A: "How can I reset my password?"
Text B: "I forgot my password. How do I change it?"
Text C: "What is the capital of France?"
```

A and B should have very similar embeddings. C should be far from both.

This is what makes embeddings powerful for:

- Semantic search
- Document similarity
- Clustering
- Recommendation
- RAG

## Comparing vectors — cosine similarity

Given two vectors, we need a way to say how similar they are. The most common measure in LangChain / RAG contexts is **cosine similarity**.

Cosine similarity measures the angle between two vectors, ignoring their magnitude:

- Value close to **1** → vectors point in nearly the same direction → very similar meaning.
- Value close to **0** → vectors are orthogonal → unrelated meaning.
- Value close to **-1** → vectors point in opposite directions → opposite meaning.

Conceptually:

```text
Query
 ↓
Embedding
 ↓
Vector
 ↓
Compare with document vectors
 ↓
Similarity scores
 ↓
Rank results
```

Example:

```text
Document A → 0.92
Document B → 0.81
Document C → 0.37
```

Higher similarity means the vectors point in more similar directions, and we treat those documents as more relevant.

Other measures exist too — Euclidean distance and dot product are common. You do not need to memorize the math. The key idea is:

> **Compare the query vector with stored vectors and return the closest ones.**

## Embedding models in LangChain

An embedding model is just another instance of the [Models](./04-models.md) component:

```text
                 Models
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
   Language Models      Embedding Models
```

The interface is small. Given a piece of text, it returns a vector. Given a list of texts, it returns a list of vectors.

LangChain supports many providers (commercial APIs, open-source models via Hugging Face, and locally hosted models). Because they all share a common interface, you can change embedding providers without rewriting the pipeline.

## Ingestion vs query — same model, different roles

There are two phases where an embedding model is used, and they must use the **same embedding model** (or at least a compatible one) for the results to be meaningful:

### Ingestion time

Each chunk is embedded and stored:

```text
Chunks
   ↓
Embedding Model
   ↓
Vectors
   ↓
Vector Store
```

### Query time

The user's question is embedded and compared to the stored vectors:

```text
User Question
      ↓
Embedding Model
      ↓
Query Vector
      ↓
Similarity Search
```

If ingestion and query use **different** models, the vectors live in incompatible spaces and similarity search becomes meaningless.

## Embedding vs Language Model

These are both "models" — they can be confused, especially early on.

| Aspect              | Language Model                        | Embedding Model                        |
| ------------------- | ------------------------------------- | -------------------------------------- |
| Input               | Text                                  | Text                                   |
| Output              | Text                                  | Vector of numbers                      |
| Purpose             | Generate answers, summaries, dialogue | Compare meaning across texts           |
| Used in RAG for     | Generating the final answer           | Retrieval — finding relevant chunks    |
| Parameters like temperature | Yes                          | No — no generation, so no randomness   |

Both are needed for a RAG application. The language model **thinks and writes**; the embedding model **finds what to think about**.

## Embedding vs Vector Store

Also easy to conflate.

| Component        | Responsibility                                 |
| ---------------- | ---------------------------------------------- |
| **Embedding Model** | Turns text into a vector                    |
| **Vector Store**    | Stores those vectors and searches them by similarity |

They are complementary:

```text
Text
 ↓
Embedding Model  ── produces vector ──▶  Vector Store
```

An embedding model with no vector store gives you numbers you can't search. A vector store with no embeddings has nothing to store. See the [Vector Stores](./14-vector-stores.md) chapter next.

## Why we split *before* embedding

You cannot usefully embed an entire 200-page PDF. The embedding would summarize hundreds of unrelated ideas into a single vector, and similarity search would be noisy. Splitting first (see [Text Splitters](./12-text-splitters.md)) makes each embedding correspond to a small, semantically focused chunk.

```text
Chunks (small, focused)
       ↓
Embedding Model
       ↓
Vectors (also focused)
       ↓
Precise retrieval
```

## Practical considerations

- **Dimensionality** — different embedding models produce vectors of different sizes. Downstream vector stores must be configured to expect the right dimension.
- **Cost / latency** — embedding thousands of chunks costs time and money. Cache embeddings when possible.
- **Model choice** — some models are optimized for retrieval quality; others for speed or cost. Match the model to the use case.
- **Language coverage** — not every embedding model handles every language well. Check before ingesting non-English data.
- **Determinism** — embeddings are typically deterministic; the same text produces the same vector on the same model. This is why caching works.

## Where embeddings sit in the mental model

Think of building a digital library:

- **Document Loader** — gets the books into the library.
- **Text Splitter** — breaks books into useful sections.
- **Embedding Model** — converts the meaning of each section into coordinates.
- **Vector Store** — stores those coordinates and the associated sections.
- **Retriever** — finds sections relevant to the user's question.
- **LLM** — reads those sections and formulates the answer.

Embeddings are the step that converts human meaning into machine-comparable coordinates.

## Key takeaways

- An embedding is a vector that represents the meaning of a piece of text.
- Similar meaning → nearby vectors; different meaning → distant vectors.
- **Cosine similarity** is the most common way to compare embeddings.
- Ingestion and query must use the **same** embedding model.
- Language models generate; embedding models compare. Both are needed for RAG.
- Embeddings alone don't search anything — they need a **vector store** (next chapter).

### Final mental model

```text
    Text
     ↓
Embedding Model
     ↓
   Vector
     ↓
Vector Store  ── similarity search ──▶  Relevant Documents
```

> **Embeddings turn meaning into geometry. Everything about semantic retrieval flows from that.**

---

Next: [Vector Stores](./14-vector-stores.md) — how those vectors are stored and searched efficiently.
