# Introduction to LangChain

## What is LangChain?

**LangChain is an open-source framework for building applications powered by Large Language Models (LLMs).**

An LLM by itself can generate text. A real application usually needs much more than that: prompts, data, tools, retrievers, databases, memory, and other integrations. LangChain provides reusable components that make it easier to connect an LLM to all of these:

```text
                LLM
                 │
     ┌───────────┼───────────┐
     ↓           ↓           ↓
  Prompts   Data Sources   Tools
     │           │           │
  Retrievers  Databases   APIs
     │           │           │
     └───────────┼───────────┘
                 ↓
           Application
```

Instead of manually building every integration from scratch, LangChain provides **abstractions and components** that make these applications easier to develop.

The most important idea to internalize is:

> **LangChain is not an LLM. It is a framework around LLMs.**

## Why does LangChain exist?

Building an LLM application is usually **not just calling an LLM**. A production system may need document storage, embeddings, vector databases, memory, tools, and multiple processing steps. LangChain provides a **common framework** for wiring these components together without writing every integration by hand.

LangChain supports:

- Different LLM providers
- Prompt engineering
- Structured output
- Retrieval-Augmented Generation (RAG)
- Tool calling
- Agents
- External data sources
- Databases
- Retrieval systems

Conceptually:

```text
                LangChain
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
     Models       Tools       Retrieval
       │            │            │
       └────────────┼────────────┘
                    ↓
              LLM Application
```

The next chapter, [Why do we need LangChain?](./02-why-langchain.md), explores this motivation in depth.

## The three main learning areas

LangChain content is best organized into three progressive areas:

```text
Fundamentals
     ↓
RAG
     ↓
AI Agents
```

These three cover a large portion of practical LangChain usage.

### Fundamentals

The building blocks used to construct any LLM-based application.

```text
Application
     ↓
LangChain
     ↓
LLM
```

Important fundamental concepts:

- [Models](./04-models.md) — how LangChain interacts with different LLMs.
- [Prompts](./05-prompts.md) — the instructions and context sent to the model.
- [Structured Outputs](./06-structured-outputs.md) and [Output Parsers](./07-output-parsers.md) — turning free-form model text into structured data.
- [Chains](./08-chains.md) — connecting multiple operations into a workflow.
- [Runnables](./09-runnables.md) — the shared interface every component follows.

### RAG (Retrieval-Augmented Generation)

RAG allows an LLM to answer questions using external knowledge.

```text
Documents
    ↓
Document Loaders
    ↓
Text Splitters
    ↓
Embeddings
    ↓
Vector Database
    ↓
Retriever
    ↓
Relevant Context
    ↓
LLM
    ↓
Answer
```

RAG solves a real limitation of LLMs:

- Their training data may not contain your private or internal information.
- Their knowledge can become outdated.
- You may need answers grounded in specific documents.

RAG retrieves relevant information and gives it to the LLM as context:

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

The LLM doesn't need to have the information in its original training data.

RAG topics covered in later chapters:

- [Document Loaders](./11-document-loaders.md)
- [Text Splitters](./12-text-splitters.md)
- [Embeddings](./13-embeddings.md)
- [Vector Stores](./14-vector-stores.md)
- [Retrievers](./15-retrievers.md)
- [Retrieval-Augmented Generation (RAG)](./16-rag.md)

### AI Agents

Agents take the idea of LLM applications further. Instead of a fixed pipeline:

```text
Question → LLM → Answer
```

an agent can **decide what to do**:

```text
User Request
     ↓
   Agent
     ↓
Decide what to do
     ↓
Choose Tool
     ↓
Execute Tool
     ↓
Observe Result
     ↓
Decide next step
     ↓
Final Answer
```

Agents combine:

```text
LLM
 +
Tools
 +
Decision Making
 +
Execution Loop
```

Agent topics covered in later chapters:

- [Tools and Tool Calling](./17-tools-and-tool-calling.md)
- [Agents](./18-agents.md)

## The overall LangChain mental model

LangChain is best understood as a collection of composable building blocks:

```text
                    LangChain
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
   Fundamentals        RAG             Agents
        │               │                │
   ┌────┼────┐      ┌───┼────┐       ┌───┼────┐
   ↓    ↓    ↓      ↓   ↓    ↓       ↓   ↓    ↓
 Models Prompts   Loaders Embeddings Retrievers Tools
 Chains Runnables Splitters Vector DBs          Tool Calling
 Parsers                                        Agents
```

These components can be combined to build real applications.

## Recommended learning order

The rest of this documentation follows the order below. Each stage prepares the reader for the next:

```text
1. LangChain fundamentals
       ↓
2. Models
       ↓
3. Prompts
       ↓
4. Structured Outputs
       ↓
5. Output Parsers
       ↓
6. Chains
       ↓
7. Runnables
       ↓
8. Runnable Primitives
       ↓
9. Document Loaders
       ↓
10. Text Splitters
       ↓
11. Embeddings
       ↓
12. Vector Stores
       ↓
13. Retrievers
       ↓
14. RAG
       ↓
15. Tools
       ↓
16. Tool Calling
       ↓
17. Agents
```

## Key takeaways

### LangChain

- Open-source framework for building LLM applications.
- Provides abstractions and integrations around LLMs.
- Supports different models, tools, databases, and data sources.
- The framework itself is **not an LLM**.

### Learning progression

```text
Fundamentals
     ↓
RAG
     ↓
Agents
```

### Fundamentals focus

```text
Models
Prompts
Output Parsers
Runnables
Chains
Memory
```

### RAG focus

```text
Document Loaders
Text Splitters
Embeddings
Vector Stores
Retrievers
```

### Agents focus

```text
Tools
Tool Calling
Agents
```

### Core idea to remember

> **LangChain gives you composable building blocks for going from a basic LLM call → a RAG application → a tool-using agent.**

---

Next: [Why do we need LangChain?](./02-why-langchain.md) — the motivation and the problems LangChain actually solves.
