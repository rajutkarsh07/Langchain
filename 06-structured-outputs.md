# Structured Outputs

Traditionally, LLMs return **unstructured text**. This is fine for a chat UI, but it makes automated processing extremely fragile. **Structured outputs** let you ask the model to return typed, schema-conformant data — the kind of data your application, database, or downstream API can actually consume.

This chapter is about **native structured output** — the model itself producing the shape you want. [Output Parsers](./07-output-parsers.md) cover the alternative: extracting structure from a text response after the fact.

## The problem: unstructured output

A normal LLM response might look like:

```text
"The product is good overall.
The sentiment is positive.
Its main advantages are performance and reliability."
```

For a human that's fine. For a program, it is nearly useless. What you often want is:

```json
{
  "summary": "The product is good overall.",
  "sentiment": "positive",
  "pros": ["performance", "reliability"],
  "cons": []
}
```

Without a structured contract, downstream systems cannot reliably extract data.

## Why do we need structured outputs?

Structured output turns an LLM from a conversational agent into a **programmable component**:

- It can write directly into a database.
- It can be fed into an API without brittle regex parsing.
- It can drive tool calling and agent workflows.
- It reduces post-processing bugs.

The core idea:

> **Structured output = a contract between the LLM and the rest of your system.**

## Typical use cases

- Extracting fields from a résumé or invoice.
- Summarizing and classifying customer reviews.
- Populating a database row from unstructured text.
- Building agents that need to call tools with typed arguments.
- Any workflow where the next step is code, not a human reader.

## How LangChain requests structured output

LangChain provides a helper on chat models:

```python
structured_model = model.with_structured_output(schema)
```

You pass a **schema** describing the fields you want; LangChain instructs the model to conform to it and validates the result.

```text
Schema
  ↓
with_structured_output()
  ↓
Model
  ↓
Structured Response
```

Under the hood, LangChain generates the system prompt and (where available) uses provider-native structured-output or function-calling APIs to enforce the shape.

## Three ways to define a schema

LangChain accepts a schema in three common forms:

```text
                Schema
                   │
     ┌─────────────┼─────────────┐
     ↓             ↓             ↓
  TypedDict     Pydantic     JSON Schema
```

Each is appropriate in different situations.

### 1. TypedDict (Python)

A lightweight typing hint for a dictionary.

```python
from typing_extensions import TypedDict

class Review(TypedDict):
    summary: str
    sentiment: str
```

- Great IDE support.
- **No runtime validation.**
- Python-only.

### 2. Pydantic (Python)

A `BaseModel`-based schema with real runtime validation.

```python
from pydantic import BaseModel, Field

class Review(BaseModel):
    summary: str
    sentiment: str = Field(description="positive, negative, or neutral")
    pros: list[str] = []
    cons: list[str] = []
```

- Runtime validation.
- Default values.
- Optional fields.
- Automatic type coercion (e.g. numeric string → int).
- Python-only.

### 3. JSON Schema

A universal, language-agnostic schema.

```json
{
  "type": "object",
  "properties": {
    "summary":   { "type": "string" },
    "sentiment": { "type": "string", "enum": ["positive", "negative", "neutral"] }
  },
  "required": ["summary", "sentiment"]
}
```

- Cross-language interoperability (front-end and back-end can share it).
- Validation depends on the validator you plug in.

## Comparing the three approaches

| Schema type  | Language          | Runtime validation | Default values | Cross-language | Best for                                             |
| ------------ | ----------------- | ------------------ | -------------- | -------------- | ---------------------------------------------------- |
| **TypedDict**| Python only       | No                 | No             | No             | Lightweight typing in a Python-only prototype        |
| **Pydantic** | Python only       | Yes                | Yes            | No             | Python projects needing strong validation            |
| **JSON Schema** | Any            | Depends on tooling | Depends        | Yes            | Multi-language projects, shared front/back contracts |

### Decision guide

- **Pure Python, quick and simple** → `TypedDict`.
- **Pure Python, needs validation** → `Pydantic`.
- **Cross-stack contract** → `JSON Schema`.

## The workflow

The end-to-end shape is the same for any schema type:

```text
1. Define a schema
      ↓
2. model.with_structured_output(schema)
      ↓
3. model.invoke(input)
      ↓
4. Model returns data conforming to the schema
      ↓
5. Application consumes it directly (DB, API, tool, ...)
```

## JSON mode vs function-calling mode

LangChain's `with_structured_output` typically uses one of two provider mechanisms under the hood:

```text
                with_structured_output
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
        JSON mode              Function-calling mode
             │                         │
   Model returns JSON        Model returns a "tool call"
   in the response body      whose arguments match the
                             schema
```

- **JSON mode** returns a JSON-formatted response.
- **Function-calling mode** allows the LLM to invoke a defined "function" whose arguments match your schema — the same mechanism that powers tool calling and agents.

Which mode is used depends on the provider and configuration. LangChain hides most of this from you, but knowing both exist is important when debugging.

## Not all models support structured output natively

Leading commercial models (e.g. from major API providers) generally support structured output modes. Many smaller open-source models do **not**.

```text
Model supports it natively?
        │
        ├── Yes → use with_structured_output
        │
        └── No  → use an Output Parser
```

This is exactly where [Output Parsers](./07-output-parsers.md) come in — they extract structure from a text response when the model can't produce it directly.

## Structured Output vs Output Parser

This is one of the most-confused topics in LangChain. The two are **complementary**, not alternatives.

| Aspect                    | Structured Output                        | Output Parser                                    |
| ------------------------- | ---------------------------------------- | ------------------------------------------------ |
| Where the structure comes from | The **model** produces structured data | The **parser** extracts structure after the fact |
| Requires model support    | Yes                                      | No                                               |
| Reliability               | Higher (enforced at generation)          | Lower (depends on the model following instructions) |
| Works with any model      | No                                       | Yes                                              |
| Typical use               | Modern API models                        | Smaller / open-source / older models             |

### Use structured output when

- The model exposes a native JSON/function-calling mode.
- You want the strongest guarantees.

### Use an output parser when

- The model returns only free-form text.
- You want to add a validation layer on top of structured output.
- You need a fallback when structured mode is unavailable.

## Limitations to keep in mind

Even with structured output, the model can produce partial or incorrect data if the schema or prompt is ambiguous. Production systems should still:

- Validate incoming data (Pydantic does this for you).
- Retry on failure.
- Provide fallback parsers.
- Include clear instructions in the prompt.

## Example — extracting review data

Schema (Pydantic):

```python
from pydantic import BaseModel, Field
from typing import Literal

class Review(BaseModel):
    summary: str = Field(description="1-2 sentence summary")
    sentiment: Literal["positive", "negative", "neutral"]
    pros: list[str] = []
    cons: list[str] = []
```

Call:

```python
structured_model = model.with_structured_output(Review)
result = structured_model.invoke(
    "The device is fast but the battery drains quickly."
)
```

Result:

```python
Review(
    summary="Fast device, but poor battery life.",
    sentiment="negative",
    pros=["fast"],
    cons=["battery drains quickly"],
)
```

The result is a validated Pydantic object, ready to store in a database or pass to another component.

## Structured outputs enable agents

Agents are LLMs that call tools. Tools require **precise, typed arguments** — free-form text is not enough. Structured output (specifically, function-calling mode) is the mechanism that lets the model produce those arguments in the first place.

```text
Agent
  ↓
LLM
  ↓
Structured tool call: { tool: "search", args: {...} }
  ↓
Tool executes
  ↓
Result
```

This is why structured output is a **prerequisite** for the [Tools and Tool Calling](./17-tools-and-tool-calling.md) and [Agents](./18-agents.md) chapters.

## Key takeaways

- Structured output turns free-text LLMs into programmable components.
- LangChain provides `with_structured_output(schema)` to request typed data from a model.
- Schemas can be defined as `TypedDict`, `Pydantic`, or `JSON Schema` — pick based on validation needs and language reach.
- **JSON mode** returns JSON; **function-calling mode** returns a typed tool call.
- Not every model supports structured output natively — use **Output Parsers** when it doesn't.
- Structured output is the foundation of tool calling and agents.

### Final mental model

```text
                LLM
                 ↓
   Wants: free-form text?           → Use raw invoke()
   Wants: structured, typed data?   → Use with_structured_output(schema)
   Model can't do it natively?      → Use an Output Parser
```

---

Next: [Output Parsers](./07-output-parsers.md) — the tools for extracting structure from a free-form response.
