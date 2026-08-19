# Runnable Primitives

The previous chapter, [Runnables](./09-runnables.md), introduced the idea that every LangChain component follows a common `.invoke(input) → output` interface. **Runnable Primitives** are the specific building blocks that let you compose those Runnables into workflows of any shape — sequential, parallel, conditional, pass-through, or wrapping arbitrary Python.

## Task-specific Runnables vs Runnable primitives

There are two conceptual categories:

```text
                Runnable
                   │
       ┌───────────┴───────────┐
       ↓                       ↓
Task-specific            Primitives
Runnables                (composition)
       │                       │
Prompt, LLM,          Sequence, Parallel,
Parser, Retriever     Branch, Lambda,
                      Passthrough
```

- **Task-specific Runnables** — "do something" (call a model, format a prompt, retrieve documents).
- **Runnable Primitives** — "control how things are executed" (in what order, in parallel, conditionally).

You use primitives to arrange task-specific Runnables into a workflow.

## The five most important primitives

```text
RunnableSequence
RunnableParallel
RunnableBranch
RunnableLambda
RunnablePassthrough
```

Each one is small on its own. Combined, they can express any workflow shape.

## 1. RunnableSequence

The most common composition — sequential execution:

```text
Runnable A
    ↓
Runnable B
    ↓
Runnable C
```

Data flows automatically:

```text
Input
 ↓
A
 ↓
B
 ↓
C
 ↓
Output
```

This corresponds to the sequential chain pattern from [Chains](./08-chains.md). In LangChain Expression Language (LCEL), sequences are written with the pipe operator:

```text
Prompt | Model | Parser
```

which is equivalent to `RunnableSequence(Prompt, Model, Parser)`.

### When to use

> **Use RunnableSequence when step B needs the output of step A.**

## 2. RunnableParallel

Runs multiple Runnables **concurrently** on the same input:

```text
             ┌→ Runnable A ─┐
Input ───────┤              ├→ Results
             └→ Runnable B ─┘
```

Each component receives the same input and produces an independent output.

### Why parallel matters

- Independent tasks finish sooner because they run at the same time.
- Multiple views of the same input can be produced without extra input plumbing.

Conceptually:

```text
Input
  ↓
RunnableParallel
  ├── Chain A
  ├── Chain B
  └── Chain C
  ↓
Combined Results
```

The result is a dictionary (or similar structure) with one entry per parallel branch.

### When to use

> **Use RunnableParallel when independent operations should run at once, or when you want to produce multiple outputs from the same input.**

## 3. RunnableBranch

Conditional execution — pick a branch based on runtime data:

```text
Input
  ↓
Condition
  ↓
 ┌───────┴───────┐
 ↓               ↓
Runnable A     Runnable B
```

The branch selected depends on the input.

### Example

Suppose you classify a message as `positive` or `negative`, then run a different chain in each case:

```text
User Message
     ↓
Sentiment Classifier
     ↓
 ┌───┴────┐
 ↓        ↓
Positive Negative
 ↓        ↓
Chain A   Chain B
```

`RunnableBranch` is the mechanism that expresses this "if/elif/else" logic inside a chain.

### When to use

> **Use RunnableBranch when the next step depends on a condition computed at runtime.**

Recall from [Chains](./08-chains.md) that conditional chains benefit enormously from structured outputs — the branching condition needs to be a **reliable, discrete value**, which is exactly what Pydantic or structured output produces.

## 4. RunnableLambda

Wraps an arbitrary Python function so it can participate in a Runnable pipeline:

```text
Python Function
      ↓
RunnableLambda
      ↓
Runnable Pipeline
```

Useful for operations that aren't inherently LLM-related:

- Data transformation
- Text processing
- Calculations
- Custom preprocessing
- Validation logic

### Example

```text
Input (str)
  ↓
RunnableLambda(str.strip)
  ↓
Prompt
  ↓
LLM
```

Whatever the function is, it now flows through the same `.invoke()` mechanism as any Runnable.

### When to use

> **Use RunnableLambda whenever you need custom Python inside a chain without writing a whole Runnable subclass.**

## 5. RunnablePassthrough

Sometimes you don't want to modify the input — you want to forward it as-is:

```text
Input
 ↓
RunnablePassthrough
 ↓
Same Input
```

At first this seems useless. It becomes important in **parallel workflows** where you want to preserve the original input alongside processed outputs.

### Example

Suppose your chain needs both the raw question and a retrieved document set to build a RAG prompt:

```text
                 Question
                     │
        ┌────────────┴────────────┐
        ↓                         ↓
RunnablePassthrough          Retriever
        │                         │
   original question         relevant docs
        └────────────┬────────────┘
                     ↓
                 Prompt
                     ↓
                    LLM
```

`RunnablePassthrough` is the mechanism that keeps the original question available at the prompt stage.

### When to use

> **Use RunnablePassthrough when you need to route the original input downstream unchanged while other branches process it.**

## Primitives in one table

| Primitive                | Shape                     | Purpose                                     |
| ------------------------ | ------------------------- | ------------------------------------------- |
| **RunnableSequence**     | A → B → C                 | Execute in order, chaining outputs           |
| **RunnableParallel**     | A → {B, C, D}             | Execute concurrently, combine results        |
| **RunnableBranch**       | A → B / C                 | Choose a branch based on a condition         |
| **RunnableLambda**       | A → f(A)                  | Wrap arbitrary Python as a Runnable          |
| **RunnablePassthrough**  | A → A                     | Forward input unchanged                      |

## Choosing the right primitive

```text
Do steps depend on each other?
    ↓
   Yes → RunnableSequence
    ↓
   No  → Do you need multiple outputs from the same input?
              ↓
             Yes → RunnableParallel
              ↓
             No  → Do you need branching?
                       ↓
                      Yes → RunnableBranch
                       ↓
                      No  → Just custom Python?
                                ↓
                               Yes → RunnableLambda
                                ↓
                               Preserve original input? → RunnablePassthrough
```

## Composing primitives into complex workflows

Every primitive is itself a Runnable, so you can nest and combine them freely. For example — take a question, retrieve documents in parallel with keeping the original, then build a prompt, call the model, and route the answer based on a classifier:

```text
                  Question
                      ↓
             RunnableParallel
             ┌────────┴────────┐
             ↓                 ↓
     RunnablePassthrough    Retriever
             │                 │
        original q       relevant docs
             └────────┬────────┘
                      ↓
                    Prompt
                      ↓
                     LLM
                      ↓
                Classifier
                      ↓
              RunnableBranch
                      ↓
             ┌────────┴────────┐
             ↓                 ↓
        Chain A (short)    Chain B (long)
             └────────┬────────┘
                      ↓
                   Answer
```

The same five primitives express arbitrarily complex flows.

## Runnable primitives and LCEL

LangChain Expression Language (LCEL) is essentially a syntax sugar over these primitives:

| LCEL syntax                 | Underlying primitive     |
| --------------------------- | ------------------------ |
| `A | B | C`                 | `RunnableSequence`       |
| `{"a": A, "b": B}`          | `RunnableParallel`       |
| `RunnableBranch(...)`       | `RunnableBranch`         |
| `RunnableLambda(f)`         | `RunnableLambda`         |
| `RunnablePassthrough()`     | `RunnablePassthrough`    |

Knowing the primitives lets you read LCEL fluently — and vice versa.

## Runnable primitives vs task Runnables — restated

| Category                | What it is                  | Examples                                    |
| ----------------------- | --------------------------- | ------------------------------------------- |
| Task-specific Runnable  | Does actual work            | Prompt, LLM, Retriever, Parser              |
| Runnable primitive      | Controls flow / composition | Sequence, Parallel, Branch, Lambda, Passthrough |

Both are Runnables. Both expose `.invoke()`. Both compose freely.

## Key takeaways

- Runnable primitives are the composition tools of LangChain.
- **Sequence** for order, **Parallel** for concurrency, **Branch** for conditions, **Lambda** for custom code, **Passthrough** for preservation.
- The primitives are themselves Runnables — they can be nested to any depth.
- LCEL is a compact syntax on top of these primitives.
- Together with task-specific Runnables, they can express any workflow LangChain supports today.

## Final mental model

```text
             Runnable Primitives
                     │
     ┌───────────────┼───────────────┐
     ↓               ↓               ↓
 Sequence        Parallel         Branch
     │               │               │
 A → B → C     A → {B,C,D}       A → B/C
                     │
     ┌───────────────┴───────────────┐
     ↓                               ↓
  Lambda                        Passthrough
  (wrap Python)                (preserve input)
```

> **The five primitives are the vocabulary of LangChain workflows. Everything else — chains, RAG pipelines, agents — is a composition of them.**

---

Next: [Document Loaders](./11-document-loaders.md) — the first step of the RAG pipeline, where external data enters LangChain.
