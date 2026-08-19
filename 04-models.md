# Models

The **Models component** is one of the core parts of LangChain. Its purpose is to provide a **common interface for working with different AI models**, including language models and embedding models.

## What is the Models component?

At the highest level, LangChain groups AI models into two categories:

```text
                 Models
                   │
          ┌────────┴────────┐
          ↓                 ↓
   Language Models    Embedding Models
          │                 │
     ┌────┴────┐            ↓
     ↓         ↓          Vectors
    LLM    Chat Model
```

### Language models

Take text as input and produce text as output. Used for:

- Question answering
- Summarization
- Code generation
- General NLP tasks

### Embedding models

Convert text into numerical vectors. Used for:

- Semantic search
- Document similarity
- Retrieval
- RAG

## LLM vs Chat Model

The distinction between LLM and Chat Model matters because they are used differently.

### LLM

An LLM works with plain text:

```text
Text
 ↓
LLM
 ↓
Text
```

Well-suited to general text-generation and single-turn NLP tasks.

### Chat Model

Chat models are designed around **conversations**. They understand concepts such as:

```text
System
User
Assistant
```

and can work with conversation history:

```text
System Message
      ↓
User Message
      ↓
Assistant Response
      ↓
User Message
      ↓
Assistant Response
```

So:

> **LLM → general text in / text out**
>
> **Chat Model → conversational interaction with roles and history**

### When to use which

| Situation                                       | Use              |
| ----------------------------------------------- | ---------------- |
| Single-turn generation, summarization, snippets | LLM              |
| Multi-turn conversation, chat UI                | Chat Model       |
| Any modern OpenAI/Anthropic/Google model        | Chat Model       |
| Simple text-to-text prototypes                  | Either           |

In practice, most modern LangChain applications use Chat Models because they align with how contemporary providers expose their APIs.

## Why does LangChain have a Models abstraction?

There are many model providers:

```text
OpenAI
Anthropic
Google
Hugging Face
Open-source models
...
```

Without an abstraction layer, application code becomes tightly coupled to each provider. LangChain provides a common interface:

```text
                Your Application
                       ↓
                   LangChain
                       ↓
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Provider A   Provider B   Provider C
```

The application can interact with different models using a similar programming interface.

## The `invoke()` method

LangChain models are invoked through a standard interface:

```python
response = model.invoke("Explain embeddings")
```

Conceptually:

```text
Input
  ↓
invoke()
  ↓
Model
  ↓
Response
```

This is part of the Runnable interface, which we will cover in the [Runnables](./09-runnables.md) chapter. For now, remember that `invoke()` is the single, consistent way to call a model regardless of which provider is behind it.

## Closed-source vs open-source models

There are two broad approaches to running language models.

### Closed-source / API models

You call a remotely hosted model through an API:

```text
Your Application
      ↓
     API
      ↓
Cloud Model
      ↓
Response
```

| Advantages                             | Disadvantages                     |
| -------------------------------------- | --------------------------------- |
| No need to manage model infrastructure | Usage costs                       |
| Easier to get started                  | Dependency on provider            |
| Usually strong model quality           | Less control/customization        |
| No local GPU requirement               | Data/privacy considerations       |

### Open-source models

Open-source models can be accessed through platforms such as Hugging Face or downloaded and run locally:

```text
Your Application
      ↓
Local Model
      ↓
Response
```

| Advantages                              | Disadvantages                     |
| --------------------------------------- | --------------------------------- |
| More control                            | Hardware requirements             |
| Greater customization                   | Model management                  |
| Privacy advantages                      | Potentially slower inference      |
| No per-request API cost when local      | More operational complexity       |

### Hugging Face

Hugging Face provides a large ecosystem of open-source models. You can interact with them in two ways:

```text
                 Hugging Face
                      │
             ┌────────┴────────┐
             ↓                 ↓
          API Usage        Local Model
```

- **API** — the application communicates with a remotely hosted model.
- **Local** — you download the model and run inference yourself.

The second approach gives more control but requires appropriate hardware.

## Model parameters

Two parameters are particularly important for controlling generation behavior.

### Temperature

Controls the randomness/creativity of the output. The typical range is approximately `0` to `2`.

```text
Low Temperature (≈ 0)
      ↓
More predictable
More deterministic
Less variation

High Temperature (≈ 1.5)
      ↓
More creative
More varied
Less predictable
```

The right value depends on the task:

- For classification, extraction, or structured output — prefer lower temperature.
- For creative writing or ideation — a higher temperature is often useful.

### Max tokens

`max_tokens` controls the maximum amount of output the model can generate:

```text
max_tokens = 100

Model
 ↓
Can generate up to approximately
the configured output limit
```

Model configuration is not just about choosing the model; you also control aspects of its generation behavior.

## API key management

When using hosted model APIs, **do not hardcode secrets directly in source code**.

Bad:

```python
api_key = "my-secret-key"
```

Instead, store secrets in environment variables, commonly loaded from a `.env` file:

```text
.env
 ↓
Environment Variable
 ↓
Application
 ↓
Model API
```

This is a widely recommended practice and keeps credentials out of your version control history.

## Embedding models

This is the second major model category.

An embedding model converts text into a vector:

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
[0.13, -0.42, 0.77, ...]
```

The vector captures semantic information about the text.

## Why embeddings matter

Embeddings allow us to compare pieces of text based on **meaning**, not just literal words.

Suppose:

```text
Text A: "How can I reset my password?"
Text B: "I forgot my password. How do I change it?"
```

Their embeddings should be relatively close because their meanings are similar.

While:

```text
Text C: "What is the capital of France?"
```

would likely be much farther away.

This enables:

- Semantic search
- Document similarity
- Retrieval
- Clustering
- RAG

## Cosine similarity

One common way of comparing embedding vectors is **cosine similarity**.

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

Higher similarity means the vectors point in more similar directions.

Cosine similarity is covered in detail in the [Embeddings](./13-embeddings.md) chapter.

## Embeddings → Semantic search → RAG

This connects directly to the retrieval side of LangChain.

Ingestion:

```text
Documents
    ↓
Embedding Model
    ↓
Document Vectors
    ↓
Vector Store
```

Query:

```text
User Query
    ↓
Embedding Model
    ↓
Query Vector
    ↓
Similarity Search
    ↓
Relevant Documents
    ↓
LLM
    ↓
Answer
```

Embeddings are the foundation of RAG retrieval — see the [RAG](./16-rag.md) chapter.

## The Models component in the bigger picture

```text
                    LangChain
                        │
                        ▼
                     Models
                        │
             ┌──────────┴──────────┐
             ↓                     ↓
      Language Models       Embedding Models
             │                     │
       ┌─────┴─────┐               │
       ↓           ↓               ↓
      LLM     Chat Model         Vectors
       │           │               │
       └─────┬─────┘               │
             │                     │
             ↓                     ↓
        Generation              Retrieval
             │                     │
             └─────────┬───────────┘
                       ↓
                 LLM Application
```

## Key takeaways

### Models component

LangChain's Models component provides a unified interface for working with different AI models.

### Language models

```text
Text → Model → Text
```

Used for generation tasks.

### LLM vs Chat Model

```text
LLM         → general text generation
Chat Model  → conversational interaction, roles, multi-turn
```

### Embedding models

```text
Text → Vector
```

Used for semantic search and retrieval.

### Closed vs open source

```text
Closed-source
→ Easy + powerful
→ API cost + less control

Open-source
→ Control + customization
→ Hardware + operational cost
```

### Important parameters

```text
Temperature → Randomness / creativity
Max Tokens  → Output length limit
```

### Most important mental model

> **Language models generate text; embedding models convert text into vectors for semantic comparison and retrieval.**

And LangChain's Models abstraction lets you work with different model providers through a consistent interface.

---

Next: [Prompts](./05-prompts.md) — the inputs and instructions you send to those models.
