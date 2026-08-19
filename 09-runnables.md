# Runnables

A **Runnable** is one of the most important abstractions in LangChain. It provides a **standard interface for components** — prompts, LLMs, retrievers, parsers, and workflow pieces all conform to it.

The core idea is simple:

> **A Runnable takes an input, does some work, and produces an output through a common interface.**

## Why were Runnables introduced?

Originally, LangChain components had **different interfaces**:

```text
LLM          → predict()
Prompt       → format()
Retriever    → get_relevant_documents()
Parser       → parse()
```

Because each component exposed different methods, connecting them required custom **glue code**:

```text
Prompt
   ↓
manual conversion
   ↓
LLM
   ↓
manual extraction
   ↓
Retriever
   ↓
manual conversion
   ↓
Parser
```

As LangChain grew, the number of specialized chains and integrations grew as well, making the framework harder to learn and maintain.

## The solution: Runnable

LangChain introduced a common abstraction:

```text
                    Runnable
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      Prompt          LLM         Retriever
        ↓              ↓              ↓
      Parser        Custom Code     ...
```

Each component follows the same basic interface. The most important method is:

```python
invoke(input)
```

So instead of learning a different method for every component, you learn **one fundamental interface**.

## The `.invoke()` method

The core contract is:

```text
input
  ↓
invoke()
  ↓
output
```

For example:

```python
result = runnable.invoke(input)
```

Conceptually:

```text
Runnable
   │
   ├── accepts input
   │
   ├── performs its task
   │
   └── returns output
```

`.invoke()` is the primary/common method that **every** Runnable supports.

## Runnable = unit of work

A useful mental model:

> **Runnable = one unit of work with a standardized interface.**

For example:

```text
Prompt Runnable    → creates prompt
LLM Runnable       → generates response
Parser Runnable    → parses response
Retriever Runnable → retrieves documents
```

The components perform completely different tasks, but they expose the same interface.

## The LEGO analogy

> **Runnables are like LEGO blocks.**

Each block has a standardized connection point:

```text
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Prompt  │ →  │   LLM   │ →  │ Parser  │
└─────────┘    └─────────┘    └─────────┘
```

Because they follow the same interface, they can be snapped together.

And importantly:

> **A chain made from Runnables can itself behave as a Runnable.**

## Recursive composition

This is one of the most important consequences.

Suppose:

```text
Prompt → LLM → Parser
```

is a Runnable.

You can then put that entire workflow inside another workflow:

```text
Chain A
   ↓
Chain B
   ↓
Chain C
```

Each chain is itself a Runnable. So:

```text
Runnable
   ↓
Runnable
   ↓
Runnable
```

can become:

```text
Complex Runnable
```

This is called **composability**, and it is the reason LangChain workflows scale from a two-step pipeline to a sophisticated agent without changing the fundamental mechanics.

## Why standardization is powerful

Without a standard interface:

```text
Prompt → custom adapter → LLM
LLM → custom adapter → Parser
Parser → custom adapter → Retriever
```

With Runnables:

```text
Prompt → LLM → Parser → Retriever
```

The integration logic becomes much simpler.

### Benefits

- Less boilerplate.
- Easier composition.
- Easier reuse.
- Cleaner code.
- Easier maintenance.
- Easier extension.
- Components are interchangeable.

## Task-specific components are Runnables

Many LangChain components implement the Runnable interface:

- Prompt templates
- LLMs
- Chat models
- Retrievers
- Parsers
- Text splitters
- Other workflow components

The important point isn't that they do the same task — they don't. The important point is:

> **They expose a standardized way of being executed and composed.**

## A simple Runnable pipeline

A basic pipeline:

```text
Input
  ↓
Prompt
  ↓
LLM
  ↓
Parser
  ↓
Output
```

Every component is a Runnable, so the data flow is really:

```text
Input
  ↓
Runnable 1
  ↓
Runnable 2
  ↓
Runnable 3
  ↓
Output
```

The output of one Runnable becomes the input of the next.

## Custom Runnables

You aren't limited to LangChain's built-in components. You can write your own:

```python
class MyRunnable(Runnable):

    def invoke(self, input):
        # custom logic
        return output
```

The only requirement is implementing `invoke()`. Your custom logic now participates in the same workflow as any LangChain component.

### Why custom Runnables matter

Imagine you need custom processing:

```text
Input
 ↓
Custom Processing
 ↓
LLM
 ↓
Parser
```

Without Runnables, you'd manually integrate your function with every downstream component.

With a Runnable:

```text
Custom Runnable
       ↓
      LLM
       ↓
    Parser
```

Your custom component now follows the same interface. This makes LangChain extremely extensible.

## Additional Runnable methods

Conceptually, LangChain provides an abstract base:

```text
Runnable
   │
   ├── invoke()
   │
   ├── batch()
   │
   └── stream()
```

`.invoke()` is the core; the others cover additional execution patterns.

### `invoke(input)`

Process a single input and get a result.

```text
Input → invoke() → Output
```

The fundamental operation.

### `batch(inputs)`

Instead of processing one input at a time:

```text
Input 1 → Runnable
Input 2 → Runnable
Input 3 → Runnable
```

batch handles multiple inputs together:

```text
[Input 1, Input 2, Input 3]
             ↓
          batch()
             ↓
[Output 1, Output 2, Output 3]
```

### `stream(input)`

Streaming produces results progressively instead of waiting for the complete result:

```text
Input
 ↓
Runnable
 ↓
Output chunk 1
 ↓
Output chunk 2
 ↓
Output chunk 3
 ↓
...
```

Especially useful for interactive LLM applications (chat UIs that show tokens as they arrive).

`.invoke()` remains the core; streaming and batching are additional capabilities.

## Runnables and chains

The distinction is subtle but important.

### Chain

A **workflow**:

```text
Prompt → LLM → Parser
```

### Runnable

The **standard execution interface** components in the workflow follow:

```text
invoke(input) → output
```

And importantly:

> **Chains themselves can be Runnables.**

The relationship:

```text
                 Runnable
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
     Prompt         LLM        Parser
                    │
                    ↓
                  Chain
                    │
                    ↓
              Also Runnable
```

## Why Runnables solved LangChain's complexity problem

Early LangChain accumulated many specialized chain implementations:

```text
Many components
     +
Many chain types
     +
Different interfaces
     ↓
Large codebase
     ↓
Steep learning curve
```

Runnables provide a more fundamental abstraction:

```text
Many components
     ↓
Common Runnable interface
     ↓
Composable workflows
```

This removes the need to create a separate specialized chain class for every combination of components.

## Historical evolution

```text
LLM APIs become widely available
            ↓
LangChain
            ↓
Connect different LLMs
            ↓
More components added
            ↓
Prompts + retrievers + parsers +
document processing + memory + ...
            ↓
Many specialized chains
            ↓
Complexity / codebase bloat
            ↓
Runnables introduced
            ↓
Standardized component interface
```

Runnables are the current, unified way of thinking about LangChain composition.

## Runnables vs traditional functions

A normal function:

```text
function(input) → output
```

A Runnable turns this into a standardized LangChain component:

```text
Runnable
   ↓
invoke(input)
   ↓
output
```

The advantage is that a Runnable now participates in the larger LangChain composition model, including chains, parallel execution, branching, batching, and streaming.

## Runnable primitives vs task components

There are two conceptual categories:

### Task-specific Runnables

Actually perform some useful task:

```text
Prompt
LLM
Retriever
Parser
```

### Runnable primitives

Control how Runnables interact:

```text
RunnableSequence
RunnableParallel
RunnableBranch
RunnableLambda
RunnablePassthrough
```

So:

```text
Task-specific Runnable → "Do something"
Runnable Primitive     → "Control how things are executed"
```

The primitives get their own chapter — see [Runnable Primitives](./10-runnable-primitives.md).

## Complete architecture

```text
                         Runnable
                            │
             ┌──────────────┴──────────────┐
             │                             │
      Task-Specific                   Primitives
             │                             │
    ┌────────┼────────┐          ┌─────────┼─────────┐
    ↓        ↓        ↓          ↓         ↓         ↓
  Prompt     LLM   Retriever   Sequence  Parallel  Branch
    │        │        │             │        │        │
    └────────┼────────┘             └────────┼────────┘
             ↓                              ↓
          Processing                    Orchestration
             │                              │
             └──────────────┬───────────────┘
                            ↓
                     Complex Workflow
```

## Key takeaways

- **Runnable** — a standardized unit of work in LangChain.
- **`.invoke()`** — the fundamental interface: input → processing → output.
- **Standardization** — prompts, LLMs, retrievers, parsers, and custom components all work together without custom glue code.
- **Composability** — Runnables can be combined into larger Runnables, including chains.
- **Runnable primitives** — control execution patterns (sequential, parallel, conditional, pass-through, custom-function).
- **Batch & streaming** — additional execution modes beyond single `invoke`.
- **Custom Runnables** — your own logic can plug into LangChain workflows.

## Final mental model

```text
                       LANGCHAIN
                           │
                       RUNNABLE
                           │
             "Standard interface for work"
                           │
                    invoke(input)
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
       Components                   Primitives
             │                           │
     ┌───────┼───────┐          ┌────────┼────────┐
     ↓       ↓       ↓          ↓        ↓        ↓
  Prompt     LLM  Retriever   Sequence Parallel Branch
     │       │       │          │        │        │
     └───────┼───────┘          └────────┼────────┘
             ↓                           ↓
          Process                    Orchestrate
             └─────────────┬─────────────┘
                           ↓
                    Complex AI Workflow
```

### The one sentence to remember

> **Runnables are LangChain's standardized building blocks: every component exposes a common execution interface, allowing them to be composed like LEGO blocks into sequential, parallel, conditional, and more complex AI workflows.**

---

Next: [Runnable Primitives](./10-runnable-primitives.md) — the specific building blocks that let you compose Runnables into any workflow shape.
