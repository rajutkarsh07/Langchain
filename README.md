# LangChain

A progressive, conceptual introduction to **LangChain** — the open-source framework for building applications powered by Large Language Models (LLMs).

This documentation is designed to be read from start to finish. Each chapter builds on the previous one, so a developer who works through the entire path finishes with a clear mental model of **what each LangChain component does, why it exists, how it differs from similar components, and how they fit together into a complete LLM application**.

# Table of contents

- **Chapter I — Fundamentals**

  - [Introduction to LangChain](./01-introduction.md)
  - [Why do we need LangChain?](./02-why-langchain.md)
  - [Core Components Overview](./03-core-components.md)

- **Chapter II — Core LangChain Components**

  - [Models](./04-models.md)
  - [Prompts](./05-prompts.md)
  - [Structured Outputs](./06-structured-outputs.md)
  - [Output Parsers](./07-output-parsers.md)

- **Chapter III — Composition and Workflows**

  - [Chains](./08-chains.md)
  - [Runnables](./09-runnables.md)
  - [Runnable Primitives](./10-runnable-primitives.md)

- **Chapter IV — Document Processing**

  - [Document Loaders](./11-document-loaders.md)
  - [Text Splitters](./12-text-splitters.md)

- **Chapter V — Retrieval**

  - [Embeddings](./13-embeddings.md)
  - [Vector Stores](./14-vector-stores.md)
  - [Retrievers](./15-retrievers.md)
  - [Retrieval-Augmented Generation (RAG)](./16-rag.md)

- **Chapter VI — Agentic Applications**

  - [Tools and Tool Calling](./17-tools-and-tool-calling.md)
  - [Agents](./18-agents.md)

- **Appendix**

  - [Component Reference](#component-reference)
  - [Learning Path](#learning-path)

# What is this documentation?

LangChain contains many components — Models, Prompts, Chains, Runnables, Retrievers, Agents, and more. Many of them look similar at first glance, and it is easy to be confused by questions such as:

- Why do we need this component if another component already exists?
- What problem does this component actually solve?
- When should I use A vs B?
- How do these components fit together?
- Is this component replacing another one, or are they complementary?
- Where does this component belong in the overall LangChain architecture?

This documentation is written specifically to answer these questions. Each chapter follows a consistent structure:

```text
What is it?
     ↓
What problem does it solve?
     ↓
Why can't we simply use X?
     ↓
How does it work?
     ↓
How does it connect to other components?
     ↓
When should we use it?
```

The reader should understand **why** each component exists, not just memorize its API.

# Learning Path

LangChain is best learned in the following order. Each stage builds on the previous one.

```text
Fundamentals
     ↓
Core Components
     ↓
Composition & Workflows
     ↓
Document Processing
     ↓
Retrieval
     ↓
RAG
     ↓
Tools & Tool Calling
     ↓
Agents
```

The overall progression can also be viewed as three broad areas:

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

# Component Reference

A quick, one-line reminder for what each component does:

| Component              | Think of it as                                                    |
| ---------------------- | ----------------------------------------------------------------- |
| **Models**             | Interface to any LLM or embedding model                           |
| **Prompts**            | Reusable, validated instructions sent to the model                |
| **Structured Outputs** | The model natively returns typed, schema-conformant data          |
| **Output Parsers**     | Convert raw model text into typed, structured data                |
| **Chains**             | Multi-step workflows connecting components together               |
| **Runnables**          | The common execution interface every component follows            |
| **Runnable Primitives**| Building blocks that control how Runnables are combined           |
| **Document Loaders**   | Bring external data into LangChain's `Document` format            |
| **Text Splitters**     | Break large documents into smaller, embeddable chunks             |
| **Embeddings**         | Convert text into vectors that capture semantic meaning           |
| **Vector Stores**      | Store and search vectors by similarity                            |
| **Retrievers**         | Given a query, return relevant documents                          |
| **RAG**                | Retrieve → Augment prompt → Generate — grounded LLM answers       |
| **Tools**              | External capabilities the LLM can invoke                          |
| **Tool Calling**       | The mechanism by which a model requests a tool                    |
| **Agents**             | An LLM that decides which tools to call and in what order         |

# The Big Picture

LangChain's central value is not that it provides an LLM. Its value is helping you connect and orchestrate the many components required to build a complete LLM application.

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

> **LangChain gives you composable building blocks for going from a basic LLM call → a RAG application → a tool-using agent.**

---

Start with the first chapter: [Introduction to LangChain](./01-introduction.md).
