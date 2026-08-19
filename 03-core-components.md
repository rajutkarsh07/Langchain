# Core Components Overview

This chapter is a **conceptual overview** of the major LangChain components. The goal is to understand **what each component does and why it exists** before diving into each one individually.

The six major components are:

```text
Models
Prompts
Chains
Memory
Indexes
Agents
```

Later chapters expand each of these into a full deep dive.

## 1. Models

The **Model component** provides a common interface for interacting with different AI models.

```text
                Models
                   │
          ┌────────┴────────┐
          ↓                 ↓
    Language Models    Embedding Models
```

The main benefit is **abstraction**. Instead of your application being tightly coupled to one provider, LangChain sits between the application and the provider:

```text
Application
     ↓
LangChain Model Interface
     ↓
Provider A / Provider B / Provider C
```

This makes switching models easier and reduces code changes.

> **Models = interface to AI models.**

The [Models](./04-models.md) chapter covers this component in depth.

## 2. Prompts

A **Prompt** is the input/instruction given to an LLM. The prompt has an enormous influence on the model's output.

```text
Prompt
   +
User Input
   ↓
  LLM
   ↓
Output
```

LangChain allows prompts to be dynamic, reusable, parameterized, and role-based. Instead of hardcoding a string, you can create a reusable template:

```text
"Explain {topic} for a {audience} audience."
```

At runtime, `topic = "embeddings"` and `audience = "beginner"` produces a dynamically constructed prompt. The [Prompts](./05-prompts.md) chapter covers this in full.

## 3. Prompt engineering

Because LLM output is highly dependent on the input, **prompt design is a critical part of LLM application development**.

```text
Better Prompt
     ↓
Better-controlled Model Behavior
```

Prompts can control the model's role, context, instructions, tone, output requirements, and user-specific information.

> **Prompts = instructions and context that guide the model.**

## 4. Chains

A **Chain** connects multiple operations into a workflow. The output of one component automatically becomes the input of another.

Chains come in three important flavors:

```text
                    Chains
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
   Sequential      Parallel      Conditional
```

### Sequential chains

```text
Text → Translate → Summarize → Summary
```

Useful when one step depends on the output of the previous step.

### Parallel chains

Independent tasks can run at the same time:

```text
                 Input
          ┌────────┼────────┐
          ↓        ↓        ↓
       Task A    Task B    Task C
          └────────┼────────┘
                   ↓
              Combined Result
```

### Conditional chains

The next step depends on an intermediate result:

```text
                Input
                  ↓
               Decision
               /      \
              ↓        ↓
           Path A    Path B
              \      /
               ↓    ↓
                Output
```

> **Chains = connecting components into workflows.**

The [Chains](./08-chains.md) chapter covers all three chain types.

## 5. Memory

Basic LLM APIs are generally **stateless**. Consider:

```text
User:      "My name is Alex."
Assistant: "Nice to meet you!"

User:      "What is my name?"
```

Without providing the previous conversation as context, the model doesn't inherently know the earlier message. **Memory** addresses this problem:

```text
Conversation
     ↓
   Memory
     ↓
Relevant History
     ↓
    LLM
     ↓
Response
```

### Types of memory

**Conversation buffer** — stores full conversation history.

```text
Message 1
Message 2
Message 3
Message 4
...
```

**Sliding window** — keeps only the most recent portion. Useful when the complete conversation becomes too large.

```text
Old messages → Removed
Recent messages → Memory
```

**Summary memory** — previous conversation is represented as a summary.

```text
Long Conversation
       ↓
    Summary
       ↓
    Memory
```

**Custom memory** — designed according to the application's requirements.

> **Memory = maintaining relevant state/context across interactions.**

## 6. Indexes

The **Indexes component connects LLM applications with external knowledge sources**.

An LLM's built-in knowledge may not contain:

- Private information
- Internal documents
- Newly created information
- Domain-specific information

The indexing pipeline is:

```text
External Data
     ↓
Documents
     ↓
Chunking
     ↓
Embeddings
     ↓
Vector Database
     ↓
Semantic Search
     ↓
Relevant Information
     ↓
LLM
```

Later, at query time:

```text
User Query
 ↓
Query Embedding
 ↓
Similarity Search
 ↓
Relevant Chunks
 ↓
LLM
 ↓
Answer
```

This is essentially the foundation of **RAG**.

### Why indexes matter

Without external retrieval:

```text
User
 ↓
LLM
 ↓
Answer based on model knowledge
```

With indexing/retrieval:

```text
User
 ↓
Search External Knowledge
 ↓
Relevant Context
 ↓
LLM
 ↓
Grounded Answer
```

> **Indexes = connecting the LLM application to external knowledge for retrieval.**

The [RAG](./16-rag.md) chapter (and everything in Chapter IV/V) covers this in detail.

## 7. Agents

Agents are different from normal chains. A chain generally follows a predefined workflow:

```text
A → B → C → D
```

An agent can **reason about what it needs to do**:

```text
User Request
      ↓
    Agent
      ↓
What should I do?
      ↓
Choose Tool
      ↓
Execute Tool
      ↓
Observe Result
      ↓
What next?
      ↓
Another Tool / Final Answer
```

### Agents + tools

An agent becomes powerful when it has access to tools. Examples of tools:

```text
Calculator
Weather API
Search API
Database
Booking API
External Services
```

The agent decides which tool is appropriate. For example:

```text
User: "What is 458 × 923?"
        ↓
       Agent
        ↓
Choose Calculator
        ↓
Calculator
        ↓
Result
        ↓
Agent
        ↓
Answer
```

> **The agent decides what actions/tools are required rather than following one fixed sequence.**

## Chain vs Agent — a critical distinction

This distinction is one of the most important in LangChain.

### Chain

The developer defines the workflow:

```text
A → B → C → D
```

### Agent

The LLM can determine the workflow dynamically:

```text
             User Request
                   ↓
                 Agent
                   ↓
              Decide Action
              /           \
             ↓             ↓
          Tool A         Tool B
             ↓             ↓
             └─────┬─────┘
                   ↓
                Continue
                   ↓
                Answer
```

Comparison:

| Aspect          | Chain                     | Agent                                    |
| --------------- | ------------------------- | ---------------------------------------- |
| Workflow        | Predefined by developer   | Chosen dynamically by the LLM            |
| Predictability  | High                      | Lower — depends on model reasoning       |
| Best for        | Known, repeatable flows   | Open-ended tasks needing decisions       |
| Debugging       | Straightforward           | Harder — behavior varies per input       |
| Tool use        | Optional and hardcoded    | Central; agent picks tools at runtime    |

> **Chain = predefined workflow**
>
> **Agent = dynamically selected workflow/actions**

The [Agents](./18-agents.md) chapter covers agents in depth.

## The six components together

```text
                         LangChain
                             │
        ┌────────────┬───────┼────────┬──────────┬─────────┐
        ↓            ↓       ↓        ↓          ↓         ↓
      Models      Prompts  Chains   Memory    Indexes    Agents
        │            │       │        │          │         │
        ↓            ↓       ↓        ↓          ↓         ↓
     AI Models  Instructions Workflows Context Knowledge Tools
```

These components can be combined. For example, a sophisticated application:

```text
                    User
                      ↓
                    Agent
                      ↓
               ┌──────┴──────┐
               ↓             ↓
            Retriever      Tool
               ↓             ↓
          External Data   API
               │             │
               └──────┬──────┘
                      ↓
                    Model
                      ↑
                      │
                    Prompt
                      ↑
                      │
                    Memory
                      │
                      ↓
                   Answer
```

## Quick mental model reference

| Component   | Think of it as                   |
| ----------- | -------------------------------- |
| **Models**  | The AI model interface           |
| **Prompts** | Instructions for the model       |
| **Chains**  | Predefined workflows             |
| **Memory**  | Conversation/state storage       |
| **Indexes** | Connection to external knowledge |
| **Agents**  | AI that reasons and uses tools   |

Or even simpler:

```text
Models  → WHO thinks
Prompts → WHAT you tell it
Chains  → HOW steps are connected
Memory  → WHAT it remembers
Indexes → WHAT external knowledge it can access
Agents  → WHAT actions it decides to take
```

## Overall LangChain architecture

```text
                         User
                           ↓
                     ┌───────────┐
                     │   Agent   │
                     └─────┬─────┘
                           ↓
                 ┌──────────────────┐
                 │     Workflow      │
                 │     / Chain       │
                 └────────┬─────────┘
                          ↓
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
     Prompt            Memory            Index
        ↓                 ↓                 ↓
        └─────────────────┼─────────────────┘
                          ↓
                        Model
                          ↓
                       Output
```

## Which components matter most for RAG?

From the six components, the ones most directly connected to RAG are:

```text
Indexes
   ↓
Document Loading
   ↓
Chunking
   ↓
Embeddings
   ↓
Vector Store
   ↓
Retrieval
   ↓
Model
```

And for controlling the LLM workflow:

```text
Prompt
   ↓
Chain
   ↓
Model
```

For more autonomous applications:

```text
Model + Prompt + Tools + Reasoning
              ↓
            Agent
```

## Key takeaways

- LangChain is a collection of abstractions for combining models, prompts, workflows, memory, external knowledge, and tools into complete LLM applications.
- The six major components are Models, Prompts, Chains, Memory, Indexes, and Agents.
- Chains and Agents differ fundamentally: chains follow a predefined path; agents decide dynamically.
- Indexes are the foundation of RAG.
- These components are designed to be **composed**, not used in isolation.

### Final takeaway

> **LangChain is essentially a collection of abstractions for combining models, prompts, workflows, memory, external knowledge, and tools into complete LLM applications.**

---

Next: [Models](./04-models.md) — the interface between your application and any LLM or embedding model.
