# Text Splitters

A **Text Splitter** divides large documents into smaller pieces of text ("chunks") that are suitable for embedding, storage, and retrieval. It is the second stage of the RAG pipeline, immediately after [Document Loaders](./11-document-loaders.md).

## Why do we need text splitters?

Imagine you have a 200-page PDF. You do not want to retrieve or send the entire PDF to the LLM for every question. There are three reasons:

1. **Context limits** — LLMs have a maximum number of tokens they can accept.
2. **Cost** — larger prompts cost more, both in money and in latency.
3. **Retrieval quality** — searching over huge documents dilutes relevance; smaller pieces let you locate exactly the passage you need.

So we split:

```text
200-page document
       ↓
     Split
       ↓
Small chunks
```

For example:

```text
Document
│
├── Chunk 1
├── Chunk 2
├── Chunk 3
├── Chunk 4
├── ...
└── Chunk 500
```

These chunks are what we'll eventually embed and store.

## What a splitter actually does

A text splitter takes `Document`s from a loader and produces smaller `Document`s (chunks) with the same shape (`page_content` + `metadata`).

```text
Documents
    ↓
Text Splitter
    ↓
Smaller Documents (chunks)
```

Metadata is preserved (and sometimes enriched — e.g., recording which chunk index came from which parent).

## The recursive character splitter

The most commonly used general-purpose splitter is `RecursiveCharacterTextSplitter`:

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)

chunks = splitter.split_documents(documents)
```

Two parameters do most of the work: `chunk_size` and `chunk_overlap`.

## `chunk_size`

The splitter tries to create chunks that are approximately this large (usually measured in characters).

```python
chunk_size = 1000
```

Conceptually:

```text
Original document:

AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC

                 ↓

Chunk 1: AAAAAAAAAAAAAAAAAAAAAAAA
Chunk 2: BBBBBBBBBBBBBBBBBBBBBBBB
Chunk 3: CCCCCCCCCCCCCCCCCCCCCCCC
```

The actual splitting behavior depends on the splitter and its separators.

## `chunk_overlap`

`chunk_overlap` is crucial and often overlooked. Consider:

```text
Chunk 1:
The company provides health insurance to all
full-time employees after completing their probation.

Chunk 2:
Employees become eligible for additional benefits...
```

If you split exactly at a boundary, context can get lost — the second chunk mentions "additional benefits" but has no idea of the probation context. With overlap:

```text
Chunk 1:
The company provides health insurance to all
full-time employees after completing their probation.

Chunk 2:
after completing their probation.
Employees become eligible for additional benefits...
```

The overlapping section preserves some context between chunks.

Typical values:

```python
chunk_size    = 1000
chunk_overlap = 200
```

This means neighbouring chunks share approximately 200 characters.

### The trade-off

- **Larger overlap** → better context continuity, but more storage and more redundant retrieval.
- **Smaller overlap** → less storage, but higher risk of losing important context at boundaries.

## Why "Recursive" character splitter?

The idea is to **try sensible boundaries first** rather than blindly cutting text.

Conceptually, the splitter prefers boundaries in this order:

```text
Paragraph
   ↓
Sentence
   ↓
Word
   ↓
Character
```

So instead of:

```text
"This is a sentence that gets cut ri"
"ght in the middle..."
```

it tries to preserve meaningful units where possible.

That's why `RecursiveCharacterTextSplitter` is often a good general-purpose starting point.

## Different text splitters for different data

LangChain has different strategies depending on the data.

### Recursive character splitter

Good general-purpose option:

```python
RecursiveCharacterTextSplitter
```

Works well for unstructured or lightly structured text.

### Markdown splitter

Useful for structured Markdown:

```python
MarkdownHeaderTextSplitter
```

It can understand headers such as:

```markdown
# Introduction

## Installation

## Configuration
```

and split so that each chunk stays inside a logical section.

### Code splitting

For source code, you may want splitting strategies that respect programming-language structure — for example, splitting at function or class boundaries instead of mid-body.

### Token-based splitting

Sometimes you care about **tokens rather than characters**, especially when working with model context limits. Token-based splitters count tokens in the same way the LLM does, so chunk sizes translate directly into how much of the model's context window they consume.

## Choosing a splitter

| Data type                | Splitter                                | Why                                              |
| ------------------------ | --------------------------------------- | ------------------------------------------------ |
| General text / PDF       | `RecursiveCharacterTextSplitter`        | Balanced, respects boundaries                    |
| Markdown / docs          | `MarkdownHeaderTextSplitter`            | Respects headings and section structure           |
| Source code              | Language-specific splitter              | Respects functions/classes                       |
| Token-sensitive workloads| Token-based splitter                    | Aligns chunk size with model context limits      |

## The relationship between loader, splitter, and chunk

To reiterate a very common source of confusion:

| Stage                | Input             | Output                    |
| -------------------- | ----------------- | ------------------------- |
| **Document Loader**  | External source   | `Document`s (often big)   |
| **Text Splitter**    | `Document`s       | `Document` chunks (small) |
| **Embedding Model**  | `Document` chunks | Vectors                   |

Loaders and splitters are **complementary**. A loader hands you raw material at whatever size the source provided; the splitter turns it into pieces that are actually useful for retrieval.

## Text Splitter vs Embedding Model

Another comparison worth being explicit about:

| Component            | What it does                                       |
| -------------------- | -------------------------------------------------- |
| **Text Splitter**    | Divides text into chunks                           |
| **Embedding Model**  | Converts each chunk into a numeric vector          |

Splitters do not embed. Embeddings do not split. They run one after the other:

```text
Text Splitter → Embedding Model → Vector Store
```

## Chunk size — a practical decision

Choosing a chunk size is one of the most impactful decisions in a RAG system.

- **Too small** → chunks lack enough context to be useful; retrieval may return snippets that don't answer the question on their own.
- **Too large** → retrieval is imprecise; multiple topics in one chunk confuse semantic similarity; more tokens spent per query.

Common starting points:

- `chunk_size = 500 – 1500` characters
- `chunk_overlap = 100 – 250` characters

Tune these based on your data and retrieval quality.

## Practical tips

- Splitters preserve metadata — use it. If you know which page or section a chunk came from, keep that data on the chunk.
- If your retrieval quality is poor, before blaming the model, check chunk size and overlap first.
- Different document types may deserve different splitters in the same pipeline.
- Splitting is deterministic; you can re-run it during ingestion without side effects.

## Key takeaways

- A splitter breaks large documents into smaller, embeddable chunks.
- `RecursiveCharacterTextSplitter` is the general-purpose default.
- `chunk_size` and `chunk_overlap` are the two most important parameters.
- Overlap preserves context across boundaries.
- Different data types benefit from different splitters (Markdown, code, tokens).
- Loading, splitting, and embedding are **three separate stages** — do not conflate them.

### Final mental model

```text
Documents (from loader)
        ↓
   Text Splitter
        ↓
Chunks (with metadata)
        ↓
  Embedding Model
```

> **Loaders get the data in; splitters make it retrievable.**

---

Next: [Embeddings](./13-embeddings.md) — how each chunk becomes a vector that captures its meaning.
