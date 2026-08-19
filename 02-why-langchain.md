# Why Do We Need LangChain?

Building an LLM application is usually **not just calling an LLM**. Real applications involve multiple components — data sources, prompts, retrieval, memory, tools — and connecting them reliably is where most of the engineering effort actually goes. This chapter explains the problems LangChain was designed to solve.

## The core problem

A real LLM application may need:

```text
User
 ↓
Query Understanding
 ↓
Search / Retrieval
 ↓
Documents / Data
 ↓
LLM
 ↓
Response
```

And several other components may be involved:

- Document storage
- Document loading
- Text splitting
- Embeddings
- Vector databases
- LLM APIs
- Memory / state
- Tools
- Multiple processing steps

The challenge is making all of these components **work together reliably**.

## Example: a document chatbot

Consider an application where a user uploads a document (say, a PDF) and asks questions about it.

The application might support:

- Asking questions
- Context-aware answers
- Summarization
- Generating quizzes
- Extracting notes

The chatbot UI is the easy part. The **hard part is the system behind it**:

```text
PDF
 ↓
Storage
 ↓
Document Processing
 ↓
Semantic Search
 ↓
Relevant Information
 ↓
LLM
 ↓
Answer
```

## Keyword search vs semantic search

A basic search system searches for exact words:

```text
Query:
"How does authentication work?"

Keyword search looks for:
authentication
```

But the relevant document might say:

```text
"The system verifies a user's identity
before granting access."
```

There is no exact match for "authentication", even though the passage is directly relevant.

### Semantic search

Semantic search focuses on **meaning** rather than exact words.

```text
Query
 ↓
Embedding
 ↓
Meaning represented as vector
 ↓
Compare with document vectors
 ↓
Relevant content
```

Semantic search is the foundation of RAG applications, and it is one of the main reasons LangChain provides embedding and vector-store abstractions.

## Embeddings — a quick preview

An embedding converts text into a numerical vector representing its semantic meaning:

```text
"How does authentication work?"
              ↓
       Embedding Model
              ↓
     [0.12, -0.42, 0.81, ...]
```

Document paragraphs are also converted into vectors:

```text
Paragraph 1 → [0.10, -0.39, 0.79, ...]
Paragraph 2 → [0.82,  0.14, 0.21, ...]
Paragraph 3 → [0.15, -0.41, 0.80, ...]
```

The query vector is compared with the paragraph vectors to find the most relevant passages. Embeddings are covered in depth in the [Embeddings](./13-embeddings.md) chapter.

## The LLM as the "brain" of the application

The LLM in an application performs two major jobs:

### 1. Natural language understanding

Understand what the user is asking.

### 2. Text generation

Generate an answer using the available context.

```text
User Question
      ↓
     LLM
      ↓
Understand question
      ↓
Use retrieved context
      ↓
Generate answer
```

However, giving the LLM an entire large document is inefficient and often exceeds the model's context limit. Instead, semantic search first finds the relevant sections and gives only those to the LLM.

## A complete RAG system

Putting the pieces together:

```text
PDF Upload
    ↓
Cloud Storage
    ↓
Document Loader
    ↓
Text Chunking
    ↓
Embedding Generation
    ↓
Embedding Database
    ↓
User Query
    ↓
Query Embedding
    ↓
Similarity Search
    ↓
Top-K Chunks
    ↓
Construct Prompt
    ↓
LLM
    ↓
Final Answer
```

This is essentially the architecture behind a typical document-based RAG application.

## Why use top-K?

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

This keeps the context focused, reduces noise, and controls token usage.

## Why LLM applications are difficult to build

There are three broad classes of challenges LangChain addresses.

### Challenge 1 — Understanding and generating language

You need a system that can understand natural-language queries and produce useful responses. Modern transformer-based LLMs solve much of this problem, and LangChain gives you a common way to use them.

### Challenge 2 — Computational cost

Running large models yourself requires substantial:

- GPU resources
- Memory
- Infrastructure
- Maintenance

Most applications use hosted LLM APIs instead:

```text
Your Application
      ↓
LLM API
      ↓
Hosted Model
      ↓
Response
```

This avoids operating the model infrastructure yourself. LangChain provides a uniform interface across different hosted providers — see [Models](./04-models.md).

### Challenge 3 — Orchestrating everything

This is where LangChain becomes particularly valuable.

A RAG system may involve:

```text
Storage
   ↓
Loader
   ↓
Splitter
   ↓
Embedding Model
   ↓
Vector Database
   ↓
Retriever
   ↓
LLM
```

Manually connecting all these pieces results in a lot of integration code. LangChain provides abstractions and standard interfaces so you compose these components instead of writing custom glue between every pair.

## What LangChain actually provides

LangChain provides **modular, pluggable components** for building LLM applications:

```text
                 LangChain
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
     Models        Retrieval       Tools
       │             │             │
       ↓             ↓             ↓
    Prompts       Vector DBs     APIs
       │             │             │
       └─────────────┼─────────────┘
                     ↓
               Application
```

The goal is to reduce the amount of boilerplate required to connect these pieces.

## Chains — the workflow abstraction

One of LangChain's central concepts is a **Chain**. A chain connects multiple operations together:

```text
Input
 ↓
Prompt
 ↓
LLM
 ↓
Output Parser
 ↓
Final Output
```

The output of one step becomes the input to the next. More complex workflows can also contain:

- Sequential steps
- Parallel operations
- Conditional logic

### Mental model

> **A chain is a workflow made by connecting multiple operations.**

Chains are covered fully in the [Chains](./08-chains.md) chapter.

## Model-agnostic architecture

LangChain is deliberately designed so that your application is **not tightly coupled to any one model provider**:

```text
                Application
                     ↓
                 LangChain
                     ↓
        ┌────────────┼────────────┐
        ↓            ↓            ↓
      Model A      Model B      Model C
```

Changing the underlying model or API requires much less application-level refactoring than a direct integration would.

## Connectors and integrations

LangChain provides integrations for many external systems:

- Document loaders (PDF, CSV, HTML, cloud storage, and more)
- Text splitters
- Embedding models
- Vector databases
- LLM providers
- Other external services

Your application composes these components rather than implementing every integration from scratch.

## Memory and state

For conversational applications, the system may need to maintain information across multiple interactions:

```text
User Message 1
      ↓
    State
      ↓
User Message 2
      ↓
    State
      ↓
User Message 3
```

Memory lets applications maintain conversational continuity.

## What can you build with LangChain?

LangChain is used for several categories of applications.

### Conversational chatbots

```text
User ↔ LLM
```

Applications that communicate with users through natural language.

### AI knowledge assistants

```text
User
 ↓
Retriever
 ↓
Custom Knowledge
 ↓
LLM
```

The assistant can answer questions using custom data via RAG.

### AI agents

```text
User
 ↓
Agent
 ↓
Choose Tool
 ↓
Execute
 ↓
Observe
 ↓
Continue / Answer
```

Agents can perform tasks rather than simply generate text.

### Workflow automation

```text
Input
 ↓
Step 1
 ↓
Step 2
 ↓
Decision
 ↓
Step 3
 ↓
Output
```

### Summarization and research

Large amounts of information can be processed and retrieved without simply putting everything into a single LLM prompt.

## LangChain alternatives

There are other frameworks in this space, most notably:

- **LlamaIndex**
- **Haystack**

The right framework depends on:

- Requirements
- Integrations needed
- Ease of use
- Cost
- Ecosystem

This documentation focuses on LangChain, but the mental models it teaches transfer directly to the alternatives.

## The most important mental model

The biggest takeaway from this chapter is:

> **LangChain's main value is not that it provides an LLM. Its value is helping you connect and orchestrate the many components required to build an LLM application.**

```text
                 LLM Application
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
      Models         Retrieval          Tools
        │               │                │
     Prompts        Documents           APIs
        │           Embeddings           │
        │          Vector Store          │
        │           Retriever            │
        └───────────────┼────────────────┘
                        ↓
                    LangChain
                        ↓
                   Application
```

## Key takeaways

### Why LangChain?

- Simplifies building LLM applications.
- Connects different components.
- Reduces integration and boilerplate code.
- Provides reusable abstractions.
- Supports different models and services.

### RAG architecture (at a glance)

```text
Documents
 ↓
Load
 ↓
Split
 ↓
Embed
 ↓
Store
 ↓
Retrieve
 ↓
LLM
 ↓
Answer
```

### Core LangChain concepts introduced

```text
Models
Prompts
Chains
Document Loaders
Text Splitters
Embeddings
Vector Stores
Retrievers
Memory / State
Tools
Agents
```

### The fundamental idea

```text
LLM alone
   ↓
Can generate text

LLM + Data
   ↓
Can answer using external knowledge

LLM + Data + Tools + Orchestration
   ↓
Can build sophisticated AI applications
```

The question LangChain answers is essentially:

> *How do I connect an LLM with data, retrieval, tools, memory, and multiple processing steps to build a complete application?*

---

Next: [Core Components Overview](./03-core-components.md) — a bird's-eye view of the six main LangChain components before diving into each in detail.
