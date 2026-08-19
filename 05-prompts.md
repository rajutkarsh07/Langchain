# Prompts

The **Prompt** component is one of the core parts of LangChain. A prompt is the **input/instructions sent to an LLM**, and prompt design strongly influences the model's output.

## What is a prompt?

At the simplest level:

```text
User Input
    ↓
  Prompt
    ↓
   LLM
    ↓
  Output
```

A prompt does not have to be just a question. It can contain:

- Instructions
- Context
- User input
- Formatting requirements
- Role/behavior definitions

This chapter focuses on **textual prompts**, though prompts can involve other modalities where the model supports them.

## Why prompt design matters

LLMs are highly sensitive to their input. Even a small change in wording can produce a different response:

```text
Prompt A → Output A
Prompt B → Output B
```

Therefore:

> **Good prompt design helps make LLM behavior more consistent, relevant, and controllable.**

This is the basic idea behind **prompt engineering**.

## Static vs dynamic prompts

This is one of the most important distinctions.

### Static prompt

A static prompt is a fixed string:

```text
"Summarize this document."
```

The user or application directly provides the content. The problem is that users may provide input in inconsistent formats:

```text
User A → Good input
User B → Poor input
User C → Very long input
User D → Ambiguous input
```

This can lead to inconsistent results.

### Dynamic prompt

A dynamic prompt uses a **template with variables/placeholders**:

```text
"Summarize the following {document}
in a {style} style."
```

At runtime:

```text
document = "..."
style = "simple"
```

The application constructs the final prompt:

```text
Template
   +
Variables
   ↓
Final Prompt
   ↓
LLM
```

This gives the application much more control over the interaction.

### Why dynamic prompts are better

Dynamic templates provide:

- Consistency
- Reusability
- Validation
- Better control
- Easier maintenance
- More predictable application behavior

Instead of every user creating their own prompt:

```text
User → arbitrary prompt → LLM
```

the application controls the structure:

```text
User Input
    ↓
Application Template
    ↓
Final Prompt
    ↓
LLM
```

## `PromptTemplate`

LangChain provides `PromptTemplate` for creating reusable dynamic text prompts.

Conceptually:

```python
template = """
Explain {topic} to a {audience}.
"""
```

With:

```text
topic    = "embeddings"
audience = "beginner"
```

produces:

```text
Explain embeddings to a beginner.
```

### Why `PromptTemplate` instead of basic string formatting?

The major benefit is **validation**. Suppose your template expects `{topic}` and `{audience}`, but you forget to provide `audience`:

```text
Template
 ├── topic ✓
 └── audience ✗
        ↓
   Validation Error
```

A prompt-template system can detect that required variables are missing before the prompt is sent to the model. This is safer than relying only on basic string formatting.

### Prompt templates should be reusable

Instead of scattering prompts throughout application code, templates can be stored separately:

```text
prompts/
├── summarization.json
├── classification.json
└── question_answering.json
```

The application loads the appropriate template when needed. Benefits:

- Easier maintenance
- Reusability
- Cleaner code
- Easier experimentation
- Better separation of prompt logic from application logic

## The chatbot problem — why single-string prompts aren't enough

Suppose we build a chatbot:

```text
User → LLM → Response
```

The first interaction works. But then:

```text
User:      "My favorite language is Python."
Assistant: "Nice!"

User:      "What is my favorite language?"
```

If the previous conversation isn't provided, the model doesn't automatically have that context. This is the **chat history problem**.

### Chat history

To maintain context, we keep the conversation history:

```text
History
├── User message
├── AI response
├── User message
├── AI response
└── ...
```

Then send the relevant history along with the current input:

```text
Chat History
     +
Current User Message
     ↓
   Prompt
     ↓
    LLM
     ↓
  Response
```

This allows the model to generate context-aware responses.

## Message types

LangChain structures chat conversations using different message types. The three most important are:

```text
SystemMessage
HumanMessage
AIMessage
```

### `SystemMessage`

Defines the assistant's behavior or instructions.

```text
System:
You are a helpful assistant.
```

> **System = rules/behavior**

### `HumanMessage`

Represents the user's message.

```text
Human:
Explain embeddings.
```

> **Human = user input**

### `AIMessage`

Represents the model's response.

```text
AI:
Embeddings are numerical representations...
```

> **AI = model response**

## Why message types matter

Instead of storing conversation as plain text:

```text
"Hello"
"Hi!"
"How are you?"
"Good!"
```

we can explicitly represent who said what:

```text
SystemMessage(...)
HumanMessage(...)
AIMessage(...)
HumanMessage(...)
AIMessage(...)
```

This gives the model a structured understanding of the conversation.

## Single-turn vs multi-turn

Two common invocation patterns.

### Single-turn

For an independent question:

```text
User
 ↓
LLM
 ↓
Answer
```

Only the current message is needed.

### Multi-turn

For a conversation:

```text
History
 +
Current Message
 ↓
LLM
 ↓
Answer
```

The relevant conversation context is provided each turn.

## `ChatPromptTemplate`

For ordinary text prompts:

```text
PromptTemplate
```

is enough.

For conversations containing multiple message types, LangChain provides:

```text
ChatPromptTemplate
```

Think:

```text
PromptTemplate
     ↓
Single text prompt

ChatPromptTemplate
     ↓
Structured list of messages
```

### ChatPromptTemplate structure

Conceptually:

```text
ChatPromptTemplate
        │
        ├── System Message
        ├── Human Message
        ├── AI Message
        ├── Human Message
        └── ...
```

It can dynamically construct the complete conversational prompt.

## Message placeholders

A very useful feature is `MessagesPlaceholder`. It allows an entire list of previous messages to be inserted into a chat prompt dynamically.

```text
Chat Template
     │
     ├── System Message
     │
     ├── MessagesPlaceholder
     │        ↑
     │     Chat History
     │
     └── Current Human Message
              ↓
        Complete Prompt
              ↓
             LLM
```

This is especially useful for multi-turn applications: your chat history grows in a list, and the placeholder splices it into the prompt on every turn.

## Complete chatbot architecture

Putting everything together:

```text
                    User
                     │
                     ▼
              Current Message
                     │
                     ▼
              ChatPromptTemplate
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     System       History       Human
     Message    Placeholder     Message
        │            │            │
        └────────────┼────────────┘
                     ↓
                Complete Prompt
                     ↓
                    LLM
                     ↓
                 AIMessage
                     ↓
                Chat History
```

The history is then available for the next interaction.

## `PromptTemplate` vs `ChatPromptTemplate`

| Component               | Purpose                             |
| ----------------------- | ----------------------------------- |
| **PromptTemplate**      | Dynamic single-text prompts         |
| **ChatPromptTemplate**  | Dynamic multi-message conversations |
| **SystemMessage**       | Defines behavior/instructions       |
| **HumanMessage**        | Represents user input               |
| **AIMessage**           | Represents model response           |
| **MessagesPlaceholder** | Inserts existing chat history       |

### Choosing between them

Use `PromptTemplate` when:

- The prompt is a single text block.
- There is no notion of roles.
- The target is a general LLM (not a chat model).

Use `ChatPromptTemplate` when:

- You want to separate a system instruction from user input.
- You are building a conversation.
- You are using a chat model (most modern providers).

## Prompts + chains

A prompt doesn't have to be sent to the model manually every time. LangChain can combine prompt creation and model invocation into a **Chain**:

```text
User Input
    ↓
PromptTemplate
    ↓
Formatted Prompt
    ↓
LLM
    ↓
Output
```

The chain encapsulates this workflow. Chains are covered in the [Chains](./08-chains.md) chapter.

## The three levels of prompts

Prompts are best remembered in three levels of sophistication.

### Level 1 — Simple prompt

```text
User Input
   ↓
Prompt
   ↓
LLM
```

### Level 2 — Dynamic prompt

```text
Template
   +
Variables
   ↓
Prompt
   ↓
LLM
```

### Level 3 — Conversational prompt

```text
System Instructions
        +
Chat History
        +
Current User Input
        ↓
ChatPromptTemplate
        ↓
       LLM
        ↓
    AI Response
```

## Key takeaways

### Prompt

> Input/instructions that guide an LLM.

### Prompt engineering

> Designing prompts to produce more useful, consistent, and controlled outputs.

### Static vs dynamic

```text
Static  → Fixed text
Dynamic → Template + Variables
```

### `PromptTemplate`

> Reusable, validated dynamic text prompts.

### `ChatPromptTemplate`

> Reusable templates for structured multi-message conversations.

### Message types

```text
SystemMessage → Behavior
HumanMessage  → User
AIMessage     → Model
```

### `MessagesPlaceholder`

> Dynamically inserts previous conversation history into a chat prompt.

## Final mental model

```text
                    PROMPTS
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
    PromptTemplate          ChatPromptTemplate
          │                         │
          ↓                 ┌───────┼────────┐
    Text Template            ↓       ↓        ↓
          │                System  History  Human
          │                Message Placeholder Message
          │                         │
          └────────────┬────────────┘
                       ↓
                  Complete Prompt
                       ↓
                      LLM
                       ↓
                    Response
```

The key idea: LangChain's prompt system moves you from sending arbitrary strings to an LLM toward **structured, reusable, validated, and context-aware prompts**.

---

Next: [Structured Outputs](./06-structured-outputs.md) — how to make the model return typed, schema-conformant data instead of free-form text.
