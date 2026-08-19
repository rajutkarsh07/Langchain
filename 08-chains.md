# Chains

A **Chain** in LangChain is a way to connect multiple components into an automated workflow. Instead of manually calling a prompt, extracting a response, calling a model, extracting again, and so on, you connect the operations into a single pipeline.

> **Chain = multiple LangChain components connected together with automatic data flow.**

## Why do we need chains?

Without chains, a multi-step LLM application requires manually managing every step:

```text
Input
 ↓
Prompt 1
 ↓
LLM 1
 ↓
Extract Output
 ↓
Prompt 2
 ↓
LLM 2
 ↓
Extract Output
```

The developer has to manually pass the output of one operation into the next.

Chains automate this:

```text
Input
 ↓
Step 1
 ↓
Step 2
 ↓
Step 3
 ↓
Final Output
```

This makes applications:

- Easier to build
- More modular
- Easier to maintain
- Easier to extend
- Less repetitive

## Basic chain architecture

A simple chain can contain:

```text
Prompt
   ↓
LLM
   ↓
Output Parser
```

For example:

```text
User Input
    ↓
PromptTemplate
    ↓
LLM
    ↓
PydanticOutputParser
    ↓
Structured Result
```

The output of each component automatically becomes the input to the next.

## The three important chain types

```text
                    Chains
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
   Sequential      Parallel      Conditional
```

They solve different workflow problems.

## 1. Sequential chains

A **Sequential Chain** executes operations one after another:

```text
Step 1 → Step 2 → Step 3 → Step 4
```

The output of one step becomes the input to the next.

### Example

```text
Topic
 ↓
Generate Detailed Report
 ↓
Extract Important Points
 ↓
Summarize
 ↓
Final Output
```

The second operation cannot start until the first has produced its result.

### When to use sequential chains

> **Use sequential chains when step B needs the result of step A.**

## 2. Parallel chains

Sometimes multiple tasks can be performed independently. Instead of:

```text
Input
 ↓
Task A
 ↓
Task B
```

you can do:

```text
             ┌→ Task A ─┐
Input ───────┤          ├→ Combined Result
             └→ Task B ─┘
```

Both tasks execute at the same time.

### Why parallel chains?

Suppose you need two independent outputs from the same input:

```text
Input
 ├──→ Generate Notes
 │
 └──→ Generate Questions
```

Running them sequentially:

```text
Notes → Questions
```

takes longer.

Running them in parallel:

```text
        ┌→ Notes
Input ──┤
        └→ Questions
```

reduces the overall latency.

### `RunnableParallel`

LangChain provides a runnable abstraction for parallel execution:

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

This is covered further in [Runnable Primitives](./10-runnable-primitives.md).

### Sequential vs Parallel

|            | Sequential           | Parallel          |
| ---------- | -------------------- | ----------------- |
| Execution  | One after another    | Simultaneously    |
| Dependency | Usually dependent    | Independent       |
| Latency    | Higher               | Lower             |
| Pattern    | A → B → C            | A → {B, C}        |
| Best for   | Multi-step workflows | Independent tasks |

```text
Dependent tasks   → Sequential
Independent tasks → Parallel
```

## 3. Conditional chains

Real applications often need decisions:

```text
If condition A:
    Run Chain A
else:
    Run Chain B
```

LangChain represents this as a **Conditional Chain**:

```text
                Input
                  ↓
             Classification
                  ↓
             ┌────┴────┐
             ↓         ↓
         Condition A  Condition B
             ↓         ↓
          Chain A    Chain B
             └────┬────┘
                  ↓
               Output
```

### Example

Suppose an LLM classifies an input as `positive` or `negative`:

```text
User Input
     ↓
Sentiment Classifier
     ↓
 ┌───┴────┐
 ↓        ↓
Positive Negative
 ↓        ↓
Chain A   Chain B
```

This is essentially the LLM equivalent of:

```python
if sentiment == "positive":
    ...
else:
    ...
```

### Why structured output is important here

Conditional chains depend on **reliable outputs**. Imagine the classifier sometimes returns:

```text
"positive"
"Positive sentiment"
"I think this is probably positive."
```

Branching logic becomes unreliable.

Instead, enforce a schema:

```text
sentiment: "positive" | "negative"
```

Then the chain can reliably make its decision. This is exactly what **Pydantic output parsing** (see [Output Parsers](./07-output-parsers.md)) or **structured outputs** (see [Structured Outputs](./06-structured-outputs.md)) are for.

## Chains + output parsers

The pieces fit together naturally:

```text
Prompt
 ↓
LLM
 ↓
Pydantic Parser
 ↓
Structured Result
 ↓
Condition
 ↓
Next Chain
```

The parser makes the output predictable enough for the next stage to use.

> **Structured outputs make chains more reliable, especially when later steps depend on specific fields.**

## LangChain Expression Language (LCEL)

LangChain provides a declarative syntax for constructing chains called **LangChain Expression Language (LCEL)**. Instead of manually writing all the plumbing between components, you compose them using pipeline-style syntax:

```text
Prompt | Model | Parser
```

This represents:

```text
Prompt
  ↓
Model
  ↓
Parser
```

The main benefit is that the workflow becomes easier to read.

### Imperative vs declarative

**Imperative approach:**

```text
Call prompt
 ↓
Get result
 ↓
Pass result manually
 ↓
Call model
 ↓
Get result
 ↓
Parse result
```

**Declarative approach:**

```text
Prompt | Model | Parser
```

You describe **what connects to what**, rather than manually managing every intermediate operation. This reduces boilerplate and improves readability.

## Chains are built on Runnables

Under the hood, chains are made of **Runnables** — the shared interface every LangChain component implements.

```text
                    Runnable
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Sequence      Parallel     Branching
```

> **Chains = workflows built by composing runnable components.**

The [Runnables](./09-runnables.md) and [Runnable Primitives](./10-runnable-primitives.md) chapters cover this abstraction in depth.

## Visualizing chains

As chains become more complex, understanding the execution flow becomes difficult. LangChain provides graph-visualization capabilities such as:

```python
chain.get_graph().print_ascii()
```

Conceptually, a simple chain prints as:

```text
Input
  │
  ▼
Prompt
  │
  ▼
Model
  │
  ▼
Parser
  │
  ▼
Output
```

For branching workflows:

```text
          Model
            │
       ┌────┴────┐
       ↓         ↓
    Chain A    Chain B
       │         │
       └────┬────┘
            ↓
          Output
```

This is useful for **debugging and understanding complex workflows**.

## Combining all three patterns

Real applications combine sequential, parallel, and conditional workflows:

```text
                    Input
                      ↓
                  Step 1
                      ↓
               ┌──────┴──────┐
               ↓             ↓
            Task A         Task B
               ↓             ↓
               └──────┬──────┘
                      ↓
                  Condition
                 ┌────┴────┐
                 ↓         ↓
              Chain C    Chain D
                 └────┬────┘
                      ↓
                   Output
```

This is where chains become much more powerful than a simple `Prompt → LLM → Response`.

## Chain vs simple LLM call

### Simple LLM application

```text
User → Prompt → LLM → Response
```

### Chain-based application

```text
User
 ↓
Prompt
 ↓
LLM
 ↓
Parser
 ↓
Condition
 ↓
 ┌───────────┐
 ↓           ↓
LLM A       LLM B
 ↓           ↓
 └─────┬─────┘
       ↓
    Final Result
```

Chains let the application implement **actual workflows**.

## Chain design patterns

| Chain Type      | Pattern    | Main Purpose                     |
| --------------- | ---------- | -------------------------------- |
| **Sequential**  | A → B → C  | Multi-step dependent workflow    |
| **Parallel**    | A → {B, C} | Independent tasks simultaneously |
| **Conditional** | A → B/C    | Dynamic branching                |

Memorize this triple. Almost every LangChain workflow is a composition of these three patterns.

## Chain vs Agent — the critical distinction

Chains follow a **predefined** workflow. Even conditional chains branch on paths **you** wrote in advance.

Agents, in contrast, let the **LLM** decide what to do next at runtime. See [Agents](./18-agents.md).

| Aspect         | Chain                        | Agent                                |
| -------------- | ---------------------------- | ------------------------------------ |
| Workflow       | Defined by the developer     | Chosen dynamically by the LLM        |
| Predictability | High                         | Lower                                |
| Best for       | Known, repeatable flows      | Open-ended tasks needing decisions   |
| Debugging      | Straightforward              | Harder — behavior varies per input   |

Use a chain when you know the steps. Use an agent when you don't.

## Why chains matter

Chains solve a fundamental problem:

> **How do you turn individual LLM calls into a complete application workflow?**

Instead of treating every LLM call independently:

```text
LLM
LLM
LLM
LLM
```

you connect them into a system:

```text
Input
 ↓
Processing
 ↓
LLM
 ↓
Parsing
 ↓
Decision
 ↓
Additional Processing
 ↓
Output
```

This provides modularity, reusability, automatic data flow, better readability, easier debugging, and more scalable workflows.

## Connection to previous topics

You have now seen:

```text
Models
   ↓
Prompts
   ↓
Structured Outputs
   ↓
Output Parsers
   ↓
Chains
```

A typical application composes these into:

```text
Prompt
   ↓
Model
   ↓
Output Parser
   ↓
Chain
   ↓
Another Prompt
   ↓
Another Model
   ↓
Final Output
```

Chains provide the **orchestration layer** connecting everything.

## Final mental model

```text
                         CHAIN
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
     Sequential         Parallel        Conditional
          │                │                │
      A → B → C       A → {B,C}         A → B/C
          │                │                │
          └────────────────┼────────────────┘
                           ↓
                       RUNNABLES
                           ↓
                  Component Execution
                           ↓
              Prompts + Models + Parsers
                           ↓
                     Final Output
```

### The one thing to remember

> **A chain is a workflow that connects LangChain components and automatically manages how data flows between them.**

And the three core patterns are:

```text
Sequential  → dependency
Parallel    → independence
Conditional → decision
```

---

Next: [Runnables](./09-runnables.md) — the shared interface that makes chains possible in the first place.
