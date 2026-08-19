# Retrieval-Augmented Generation (RAG)

**Retrieval-Augmented Generation (RAG)** is the pattern of retrieving relevant documents and giving them to the LLM as context, so the model can answer questions grounded in specific data. It is the composition of everything covered in Chapter IV and Chapter V — loaders, splitters, embeddings, vector stores, and retrievers — into a complete pipeline.

## The problem RAG solves

LLMs have three limitations that make them insufficient for most real applications on their own:

1. Their training data does not contain your private or internal information.
2. Their knowledge can be outdated.
3. Their answers may not be traceable to specific sources.

An LLM asked "What is our refund policy?" or "How does this internal API authenticate?" simply cannot know — the answer isn't in its training data. Even for well-known topics, the model may hallucinate confidently rather than admit uncertainty.

RAG fixes this by giving the model the information it needs, at query time:

```text
User Question
      ↓
Retriever
      ↓
Relevant Information
      ↓
LLM
      ↓
Grounded Answer
```

The LLM doesn't need to have the information in its original training data — it just needs to be **shown** the right passages.

## The complete RAG architecture

RAG has two distinct phases: **ingestion** (offline) and **query** (online).

### Ingestion — indexing your knowledge base

```text
PDF / Website / Database
          ↓
   Document Loader
          ↓
      Documents
          ↓
    Text Splitter
          ↓
       Chunks
          ↓
    Embedding Model
          ↓
       Vectors
          ↓
    Vector Store
```

This is done once (or on a schedule). It produces a searchable index of your data.

### Query — answering a user question

```text
User Question
      ↓
   Retriever
      ↓
Relevant Documents
      ↓
Prompt + Context
      ↓
      LLM
      ↓
    Answer
```

Or, expanding the retriever step:

```text
User Question
      ↓
Embedding Model
      ↓
Query Vector
      ↓
Vector Store
      ↓
Relevant Chunks
      ↓
Prompt + Context
      ↓
LLM
      ↓
Answer
```

This runs on every query.

## The end-to-end pipeline

```text
             RAG
              │
              ▼
       ┌──────────────┐
       │     Data     │
       └──────┬───────┘
              ↓
      Document Loader
              ↓
         Documents
              ↓
       Text Splitter
              ↓
           Chunks
              ↓
       Embedding Model
              ↓
          Embeddings
              ↓
        Vector Store
              ↓
         Retriever
              ↑
              │
         User Query
              ↓
     Relevant Documents
              ↓
             LLM
              ↓
           Answer
```

Every arrow in that diagram corresponds to a chapter you have already read.

## Why RAG is not just "add documents to the prompt"

You might ask: why not just paste the whole document into the prompt?

- **Context limits** — even large models have finite context windows.
- **Cost** — every extra token costs money and latency.
- **Signal-to-noise** — a focused passage produces a much better answer than a giant blob of loosely related text.
- **Scale** — you can't paste your entire corporate knowledge base into every query.

RAG's job is precisely to select **the right small set of passages** for each individual query.

## The role of top-K

You usually don't want every matching chunk — you want the **top K most relevant chunks**:

```text
Query
 ↓
Similarity Search
 ↓
┌──────────────────────┐
│ Chunk 17 → 0.94      │
│ Chunk 42 → 0.91      │
│ Chunk 81 → 0.87      │
│ Chunk 23 → 0.82      │
└──────────────────────┘
 ↓
Top 4 chunks
 ↓
LLM
```

This keeps the prompt focused, reduces noise, and controls token usage.

## RAG in LangChain — putting components together

A minimal RAG chain in LangChain composes the components you already know:

```text
question
    ↓
RunnableParallel
├── retriever  → relevant docs
└── passthrough → question
    ↓
ChatPromptTemplate  (fills {context} and {question})
    ↓
Chat Model
    ↓
StringOutputParser
    ↓
answer
```

Each stage is a Runnable, connected via LCEL. Because everything conforms to the same interface (see [Runnables](./09-runnables.md) and [Runnable Primitives](./10-runnable-primitives.md)), the whole pipeline is itself a Runnable.

## Building a RAG chain with LCEL

Here is the same pipeline written concretely with LangChain Expression Language (LCEL). Every construct here is something the previous chapters already introduced.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# 1. Retriever (built earlier from a vector store)
retriever = vector_store.as_retriever(search_kwargs={"k": 4})

# 2. Prompt template — expects "context" and "question"
prompt = ChatPromptTemplate.from_messages([
    ("system",
     "Answer the question using ONLY the following context. "
     "If the answer is not in the context, say you don't know.\n\n"
     "Context:\n{context}"),
    ("human", "{question}"),
])

# 3. Helper: turn the retrieved Documents into a single string
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

# 4. The RAG chain
rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | model
    | StrOutputParser()
)

answer = rag_chain.invoke("How do I change my password?")
```

What each piece does:

- The dictionary is a `RunnableParallel`. It runs the retriever and the passthrough at the same time on the input string.
- `RunnablePassthrough` keeps the original question available so the prompt template can insert it into `{question}`.
- `format_docs` turns the retriever's `List[Document]` into a single string suitable for `{context}`.
- The `|` operator chains the whole thing into a sequential Runnable.

The resulting `rag_chain` is itself a Runnable — it exposes `.invoke()`, `.batch()`, and `.stream()`, and it can be composed further.

## Formatting retrieved documents

The retriever returns `Document` objects. The prompt expects a string (or a message list). Between them, you need a formatter — a small `RunnableLambda` (or plain function) that decides how documents are joined.

A few common strategies:

```python
# Simple concatenation
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

# With source citations
def format_docs_with_sources(docs):
    return "\n\n".join(
        f"[Source: {d.metadata.get('source', '?')}]\n{d.page_content}"
        for d in docs
    )

# With numbered citations the model can reference
def format_docs_numbered(docs):
    return "\n\n".join(
        f"[{i+1}] {d.page_content}" for i, d in enumerate(docs)
    )
```

The choice matters. If you want the model to cite sources, expose the source in the formatted context — the model can only cite what it can see.

## Document-chain strategies — stuff, map-reduce, refine

Once you have retrieved documents, there is more than one way to fit them into an LLM prompt. LangChain formalizes three strategies.

### Stuffing (the default)

Put all retrieved documents directly into a single prompt:

```text
Documents
   ↓
[all concatenated into one prompt]
   ↓
LLM
   ↓
Answer
```

- Simple, cheap, and works well when the retrieved content fits in the context window.
- LangChain helper: `create_stuff_documents_chain`.
- The default for most starter RAG apps.

### Map-reduce

Handle each document separately, then combine:

```text
        Documents
     ┌──────┼──────┐
     ↓      ↓      ↓
    LLM   LLM    LLM     ← map: summarize each doc
     │      │      │
     └──────┼──────┘
            ↓
           LLM             ← reduce: combine summaries
            ↓
          Answer
```

- Scales to many documents that would not fit in one prompt.
- Slower and more expensive (many LLM calls).
- Loses cross-document context during the map step.

### Refine

Iteratively update an answer, one document at a time:

```text
Doc 1 → LLM → initial answer
Doc 2 + initial answer → LLM → refined answer
Doc 3 + refined answer → LLM → final answer
```

- Preserves nuance from every document.
- Sequential — cannot be parallelized.
- Order-sensitive; the last document has an outsized effect.

### Choosing a strategy

| Situation                                            | Strategy       |
| ---------------------------------------------------- | -------------- |
| Retrieved context fits in the model's window         | **Stuff**      |
| Retrieved context is too large but chunks are independent | **Map-reduce** |
| Need to weave nuance across many documents           | **Refine**     |

Start with stuffing. Reach for the others only when it stops fitting.

## Conversational RAG

A basic RAG chain treats every question in isolation. Real chat applications need to remember earlier turns — otherwise a follow-up like "and what about for admins?" has no meaning.

The problem is subtle: **the query used for retrieval is not always the same as what the user typed.** If they ask "and what about admins?", embedding that literal string will retrieve the wrong documents.

### History-aware retrieval

The pattern LangChain uses:

```text
Chat History + Latest User Message
              ↓
      LLM: "rewrite as standalone question"
              ↓
     Standalone question
              ↓
          Retriever
              ↓
     Relevant Documents
              ↓
Chat History + User Message + Documents
              ↓
              LLM
              ↓
            Answer
```

Two LLM calls per turn: one to reformulate the query, one to answer.

- LangChain helper: `create_history_aware_retriever`.
- Store history using `SystemMessage`, `HumanMessage`, `AIMessage` (see [Prompts](./05-prompts.md)).
- Splice history into the prompt with a `MessagesPlaceholder`.

### The full conversational RAG chain

```text
history + question
        ↓
create_history_aware_retriever   → rewrites into standalone question
        ↓                            and retrieves documents
        ↓
create_stuff_documents_chain     → stuffs docs + history into a prompt
        ↓                            and calls the model
        ↓
     answer
```

LangChain provides `create_retrieval_chain` to wire these two pieces together.

## Adding citations

Grounded answers only earn user trust when they are traceable. Two common ways to add citations:

### 1. Inline in the formatted context

Include a source marker in the context string; instruct the model to cite it:

```python
def format_docs_with_ids(docs):
    return "\n\n".join(
        f"[{i}] (source: {d.metadata.get('source')})\n{d.page_content}"
        for i, d in enumerate(docs)
    )
```

Prompt:

> "When you use information from a source, cite it as `[N]`."

### 2. Structured output for citations

Ask the model to return both the answer and the list of source IDs it used, using [Structured Outputs](./06-structured-outputs.md):

```python
from pydantic import BaseModel

class Answer(BaseModel):
    answer: str
    sources: list[int]

structured_model = model.with_structured_output(Answer)
```

The application can then render the answer plus the exact sources the model claimed to use.

Both approaches rely on giving the model **identifiable, source-tagged context** — which is why metadata quality (from [Document Loaders](./11-document-loaders.md) onward) matters so much.

## Advanced RAG patterns

Once the basic pipeline works, several patterns improve retrieval quality further.

### Multi-query retrieval

An LLM rewrites the user's question into several variants; the retriever runs on each and results are merged. Covered in [Retrievers](./15-retrievers.md).

```text
Query → LLM → [Q1, Q2, Q3] → parallel retrieval → merged docs
```

### Parent document retrieval

Search small chunks (precise), but return their larger parents (rich context). Covered in [Retrievers](./15-retrievers.md).

### Self-query

The LLM parses the user's natural-language question into a **structured query** — a semantic query plus a metadata filter:

```text
"papers about embeddings published after 2022"
        ↓
       LLM
        ↓
{ query: "embeddings", filter: date > 2022 }
        ↓
Vector store search + filter
```

Combines semantic retrieval with reliable metadata constraints.

### Contextual compression

Retrieve broadly, then run each candidate document through a compressor that keeps only the passages relevant to the query:

```text
Retriever → many docs → Compressor (LLM or extractor) → shorter, focused docs → LLM
```

Reduces prompt size and noise without sacrificing recall.

### Re-ranking

Retrieve a large candidate set, then use a re-ranker model (often a cross-encoder) to reorder them by true relevance:

```text
Retriever → top 50 docs → Re-ranker → top 5 docs → LLM
```

Vector similarity is fast but rough; re-rankers are slower but much more accurate on the shortlist.

### Choosing between them

| Symptom                                         | Try                                   |
| ----------------------------------------------- | ------------------------------------- |
| Queries are ambiguous or under-specified        | Multi-query retrieval                 |
| Precision is good but the LLM lacks context     | Parent document retrieval             |
| Users use structured filters (dates, categories) | Self-query                            |
| Prompts are too long / too noisy                | Contextual compression                |
| Retrieval order is often wrong                  | Re-ranking                            |

Layer these on top of a working baseline; don't add them all at once.

## Prompt engineering for RAG

The system prompt in a RAG chain typically instructs the model to:

- Answer using only the provided context.
- Say "I don't know" if the context doesn't contain the answer.
- Cite sources from metadata when helpful.

Something conceptually like:

```text
System:
You are a helpful assistant. Answer the question using ONLY
the provided context. If the answer is not in the context,
say you don't know.

Context:
{context}

Question:
{question}
```

Prompt design is where you turn "the LLM saw some passages" into "the LLM produced a grounded, honest answer."

## When to use RAG vs alternatives

| Situation                                          | Approach                                    |
| -------------------------------------------------- | ------------------------------------------- |
| Answers depend on private / internal / current data | RAG                                         |
| Small, static context that fits in the prompt      | Just include it in the prompt               |
| The task is generation, not question-answering     | Plain prompt (no retrieval)                 |
| The task requires actions (search, calculate, call APIs) | Agents + tools (see later chapters)   |
| Data is huge but rarely changes                    | RAG + caching                               |
| Model must always be honest about missing info     | RAG with strict "answer only from context"  |

## Common failure modes

Understanding where RAG breaks is critical for building anything real.

- **Bad chunks** — chunk size too large or too small kills retrieval quality.
- **Wrong embedding model** — using different models at ingestion and query time silently returns garbage.
- **Missing metadata** — you can't filter, cite, or debug.
- **Retriever mismatch** — semantic search when you needed keyword, or vice versa.
- **Overly permissive prompt** — the model happily invents an answer even when the context doesn't support it.
- **Context overflow** — too much retrieved text pushes past the model's context window.

Each failure has a corresponding fix — usually in a specific pipeline stage. Diagnose by inspecting: what did the retriever return? Was it correct? Did the model use it?

## RAG vs Retriever

A retriever is a component. RAG is a pattern:

```text
RAG = Retriever + Prompt + LLM (+ Output Parser)
```

You can build a retriever without building RAG (e.g., for semantic search UIs). But you can't build RAG without a retriever.

## RAG vs fine-tuning

Both let a model handle domain-specific information. They are not interchangeable.

| Aspect                | RAG                                       | Fine-tuning                                |
| --------------------- | ----------------------------------------- | ------------------------------------------ |
| Where knowledge lives | External store; retrieved per query        | Baked into model weights                   |
| Updates               | Update the store; instant                  | Retrain the model                          |
| Traceability          | Sources retrievable per answer             | Opaque                                     |
| Cost                  | Ongoing per-query retrieval + generation   | Upfront training cost                      |
| Best for              | Frequently changing knowledge, citations   | Style, format, task-specific behavior      |

Most production systems use **RAG for knowledge** and, if needed, **fine-tuning for style/behavior** — not one instead of the other.

## RAG vs Agents

Both give an LLM more power than a plain prompt. They are different mechanisms.

| Aspect       | RAG                                       | Agent                                          |
| ------------ | ----------------------------------------- | ---------------------------------------------- |
| Interaction  | Fixed: retrieve → augment → generate      | Dynamic: LLM decides which tools to call        |
| Data access  | Retrieval from a knowledge base           | Any tool the agent has (retrieval + APIs, etc.) |
| Predictability | High                                    | Lower — depends on model reasoning              |
| Complexity   | Moderate                                  | Higher                                          |
| Best for     | Grounded Q&A over documents               | Multi-step tasks requiring decisions            |

An agent can *use* RAG as one of its tools. See [Tools and Tool Calling](./17-tools-and-tool-calling.md) and [Agents](./18-agents.md) for the agent side.

## Where RAG fits in the LangChain mental model

Think of building a digital library:

- **Document Loader** — get the books in.
- **Text Splitter** — break them into useful sections.
- **Embedding Model** — convert meaning into coordinates.
- **Vector Store** — store the coordinates.
- **Retriever** — find sections relevant to the reader's question.
- **LLM** — read those sections and answer.

**RAG is the whole library working together to answer a question.**

## Key takeaways

- RAG grounds an LLM in specific, retrievable documents.
- It has two phases: **ingestion** (loader → splitter → embedding → vector store) and **query** (retriever → prompt → LLM).
- Every stage is a LangChain component you've already seen.
- Build the pipeline with LCEL: `{"context": retriever | format, "question": passthrough} | prompt | model | parser`.
- The retriever returns `Document`s; a formatter turns them into a string for the prompt.
- Three document-chain strategies: **stuff** (default), **map-reduce** (scales), **refine** (nuance).
- **Conversational RAG** needs a history-aware retriever that rewrites the follow-up into a standalone question before searching.
- Citations only work if you expose source metadata to the model — either inline in the context or as a structured output field.
- Advanced patterns (multi-query, parent doc, self-query, contextual compression, re-ranking) layer on top of the baseline once you need them.
- RAG and fine-tuning solve different problems; RAG is the right tool for updateable, traceable knowledge.
- Agents can call RAG as a tool, but RAG itself is a fixed pattern.

### Final mental model

```text
        Ingestion (offline)
   ──────────────────────────
     Data
       ↓
   Loader → Splitter → Embedding → Vector Store

        Query (online)
   ──────────────────────────
     Question
       ↓
   Retriever → Prompt (context + question) → LLM → Answer
```

> **RAG is not a component. It's the composition of every retrieval component into a grounded, honest question-answering system.**

---

Next: [Tools and Tool Calling](./17-tools-and-tool-calling.md) — how an LLM can invoke external capabilities, not just read them.
