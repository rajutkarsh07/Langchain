# Tools and Tool Calling

**Tools** are external capabilities an LLM can invoke — a calculator, a search API, a database query, a file operation. **Tool calling** is the mechanism by which the model requests one of those capabilities at runtime. Together, they are the foundation of **agents** (see the next chapter).

## Why do we need tools?

An LLM on its own has two abilities: **read text** and **generate text**. That is powerful but limited. It cannot:

- Look up live information.
- Do arithmetic reliably at scale.
- Query a database.
- Call an API.
- Interact with the filesystem or the world.

Everything that requires **doing** rather than **generating** needs something outside the model. That something is a **tool**.

```text
LLM
 │
 ├── can read text
 ├── can generate text
 └── cannot act on the world

To act on the world, it needs:

Tools
 │
 ├── calculator
 ├── search
 ├── database
 ├── APIs
 └── ...
```

## What is a tool?

Conceptually, a tool is:

- A **name** (e.g., `search`).
- A **description** telling the model what it does and when to use it.
- A **schema** describing its input arguments.
- A **function** that runs when the tool is invoked.

```text
Tool
 ├── name
 ├── description
 ├── input schema
 └── function
```

In LangChain, tools are Python functions or classes registered with the framework so a model can be told about them.

## How tool calling works

The model doesn't call your Python function directly. It cannot — it is a text generator. Instead:

1. You tell the model which tools exist and what each one expects.
2. The model, given a user question, decides whether to call a tool.
3. If it decides to, it emits a **structured tool call** — a name plus arguments matching the tool's schema.
4. Your application executes the actual function with those arguments.
5. The result is fed back into the model so it can continue reasoning or produce an answer.

Conceptually:

```text
User Question
     ↓
Model (knows tool list)
     ↓
Decides: "I need to call search(query='...')"
     ↓
Application executes search
     ↓
Result returned to model
     ↓
Model produces final answer
```

The critical part is step 3: the model must produce a **typed, schema-conformant** request. This is exactly the **function-calling mode** of structured output covered in [Structured Outputs](./06-structured-outputs.md).

> **Tool calling is structured output in action. The model's "call" is a structured object; your code interprets and executes it.**

## Tool vs Tool Calling

These two terms are often confused because they show up together.

| Term            | Meaning                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| **Tool**        | The external capability (name + description + schema + function)         |
| **Tool Calling**| The **mechanism** by which the model requests a tool at runtime          |

A tool is a **thing**. Tool calling is a **process**.

- You define tools.
- The model uses tool calling to request them.

## Example — a calculator tool

Conceptually, you define a tool:

```text
Name:        calculator
Description: Compute an arithmetic expression.
Input:
   expression: string
Function:    eval(expression)  # or a safer impl
```

You attach the tool to the model, and the model becomes able to emit calls like:

```text
{ "tool": "calculator", "args": { "expression": "458 * 923" } }
```

The application executes it:

```text
User: "What is 458 × 923?"
        ↓
      Model
        ↓
tool call: calculator("458 * 923")
        ↓
     Calculator
        ↓
      Result
        ↓
       Model
        ↓
     Answer
```

Same pattern applies to a search tool, a database query, or a call to an external API.

## Defining tools in LangChain

LangChain provides several ways to declare a tool. All three produce something that can be attached to a model and invoked from a tool call.

### 1. The `@tool` decorator (simplest)

Wrap a plain Python function. The **docstring becomes the description** the model sees, and the **type hints become the schema**.

```python
from langchain_core.tools import tool

@tool
def calculator(expression: str) -> str:
    """Compute an arithmetic expression like '458 * 923'."""
    return str(eval(expression))
```

Attributes automatically set:

- `calculator.name` → `"calculator"`
- `calculator.description` → the docstring
- `calculator.args_schema` → inferred from type hints

That is enough for the model to know when and how to call it.

### 2. `StructuredTool` with an explicit Pydantic schema

When arguments need validation, defaults, or descriptions, define the schema explicitly:

```python
from pydantic import BaseModel, Field
from langchain_core.tools import StructuredTool

class SearchInput(BaseModel):
    query: str = Field(description="What to search for")
    top_k: int = Field(default=5, description="How many results to return")

def search(query: str, top_k: int = 5) -> list[str]:
    ...

search_tool = StructuredTool.from_function(
    func=search,
    name="search",
    description="Search the knowledge base for a query.",
    args_schema=SearchInput,
)
```

Pydantic here does the same job it does in [Structured Outputs](./06-structured-outputs.md) — validate that the model produced arguments matching the contract.

### 3. `BaseTool` for full custom control

For tools with async execution, custom error handling, or complex lifecycles, subclass `BaseTool` directly:

```python
from langchain_core.tools import BaseTool

class DatabaseTool(BaseTool):
    name = "database"
    description = "Run a read-only SQL query."
    args_schema = SqlInput  # a Pydantic model

    def _run(self, query: str) -> str:
        return db.execute(query).text()

    async def _arun(self, query: str) -> str:
        return await db.execute_async(query)
```

`BaseTool` is a proper Runnable, so it slots into any chain.

### Which one to use

| Situation                                     | Use                     |
| --------------------------------------------- | ----------------------- |
| Straight function, simple args                | `@tool` decorator       |
| Args need validation / defaults / descriptions | `StructuredTool`        |
| Async, custom lifecycle, complex behavior     | Subclass `BaseTool`     |

## Binding tools to a model

A tool by itself is inert. The model has to be **told** about it. That is `bind_tools()`:

```python
model_with_tools = model.bind_tools([calculator, search_tool])
```

`model_with_tools` is a new Runnable. When invoked, it:

- Tells the underlying provider about the tools (using the provider's native tool/function-calling API).
- Returns an `AIMessage` that may contain a normal text `content` **and/or** a list of `tool_calls`.

```text
model.invoke("What is 458 × 923?")
     ↓
AIMessage(
    content="",
    tool_calls=[
        { "name": "calculator",
          "args": { "expression": "458 * 923" },
          "id":   "call_abc123" }
    ]
)
```

The tool call itself is a **typed, structured object** — this is the function-calling mode of structured output in action.

## Executing tool calls

Binding tools does not execute them. The model only *requests* execution. Your application (or an agent) has to actually run the tool and send the result back.

The canonical loop:

```python
from langchain_core.messages import HumanMessage, ToolMessage

messages = [HumanMessage("What is 458 × 923?")]

# 1. Model produces tool_calls
ai_msg = model_with_tools.invoke(messages)
messages.append(ai_msg)

# 2. For each requested call, run the tool and record the result
tools_by_name = {"calculator": calculator, "search": search_tool}
for call in ai_msg.tool_calls:
    tool = tools_by_name[call["name"]]
    result = tool.invoke(call["args"])
    messages.append(
        ToolMessage(content=str(result), tool_call_id=call["id"])
    )

# 3. Feed the results back and let the model produce a final answer
final = model_with_tools.invoke(messages)
```

The four message types together — `SystemMessage`, `HumanMessage`, `AIMessage`, `ToolMessage` — form the complete conversation the model sees.

```text
HumanMessage   → the user's request
AIMessage      → the model's response (text and/or tool_calls)
ToolMessage    → the result of executing a tool_call
```

`ToolMessage.tool_call_id` links the result back to the specific request. Providers use this to reconcile multiple calls issued in the same turn.

## Parallel tool calls

Modern chat models can emit **multiple tool calls in a single response**:

```text
User: "What is 458 × 923, and what's the weather in Tokyo?"
        ↓
AIMessage(
    tool_calls=[
        { "name": "calculator", "args": {...}, "id": "c1" },
        { "name": "weather",    "args": {...}, "id": "c2" },
    ]
)
```

Both should be executed (ideally in parallel) and returned as two separate `ToolMessage`s before the model produces a final answer:

```text
        AIMessage(tool_calls=[c1, c2])
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
     execute c1          execute c2
          │                   │
          ↓                   ↓
    ToolMessage(c1)     ToolMessage(c2)
          │                   │
          └─────────┬─────────┘
                    ↓
                AIMessage(final answer)
```

Because tools are Runnables, `RunnableParallel` (from [Runnable Primitives](./10-runnable-primitives.md)) is a natural fit for executing them concurrently.

## Prebuilt tools in LangChain

LangChain ships with many ready-made tools you don't need to write yourself:

- **Search** — web search integrations (various providers).
- **Wikipedia / arXiv** — knowledge lookups.
- **Requests** — arbitrary HTTP calls.
- **Shell** — command execution (use with care).
- **Python REPL** — evaluate Python (use with extreme care).
- **SQL / DataFrame** — query structured data.
- **File-system** — read/write local files.
- **Retriever tool** — wrap a retriever so an agent can call it (via `create_retriever_tool`).

Wrapping a retriever as a tool is especially important: it turns your RAG pipeline into a capability an agent can choose to invoke, rather than a fixed prefix on every prompt.

```python
from langchain.tools.retriever import create_retriever_tool

docs_tool = create_retriever_tool(
    retriever,
    name="search_docs",
    description="Search the internal knowledge base for a query.",
)
```

## Error handling in tool execution

Tool calls will fail — malformed arguments, network errors, invalid queries. The right pattern is to **catch the error, return it as the tool result, and let the model decide what to do next**:

```python
for call in ai_msg.tool_calls:
    try:
        result = tools_by_name[call["name"]].invoke(call["args"])
    except Exception as e:
        result = f"Error: {e}"
    messages.append(ToolMessage(content=str(result),
                                tool_call_id=call["id"]))
```

The model, on the next turn, will see the error and can retry with different arguments or fall back to another approach. Silently swallowing errors — or letting them propagate — both break the loop.

## Categories of tools

Any external capability can be a tool. Common categories:

- **Search** — web search, internal search, RAG retrieval.
- **Math and computation** — calculator, unit conversion.
- **Data access** — SQL queries, key/value lookups, file reads.
- **APIs** — weather, translation, geocoding, custom services.
- **Actions** — send an email, create a ticket, book something.

Every one of these follows the same shape: a schema-defined function the model can request.

## Tools + LLM = the foundation of agents

An agent is an LLM that decides which tools to call and in what order. A simple loop looks like:

```text
User Request
     ↓
     Agent
     ↓
LLM decides:
   - answer directly, or
   - call a tool
     ↓
if tool call:
     ↓
   execute tool
     ↓
   feed result back to LLM
     ↓
   (loop until final answer)
```

Without tool calling, an agent would have nothing to do beyond talking. With it, the LLM can plan, act, observe, and continue. See the [Agents](./18-agents.md) chapter for the full picture.

## Why the model doesn't just "do it itself"

A common early misconception: "why doesn't the LLM just do arithmetic / search / etc. itself?"

- LLMs generate text token-by-token. They are unreliable at exact arithmetic, especially with many digits.
- LLMs don't have live data. Their training cut off at some point in the past.
- LLMs can't touch external systems. They generate text; they don't run code or make HTTP requests.

Tools delegate the parts that require **precision, freshness, or side effects** to systems built to handle those things.

## Structured output is a prerequisite

To make tool calling work, the model must produce a **structured, typed request**. That is exactly what [Structured Outputs](./06-structured-outputs.md) enable. In fact, most provider APIs implement tool calling *as* structured output under a specific mode.

```text
Structured Output
        │
        ▼
Function-calling mode
        │
        ▼
Tool Calling
        │
        ▼
    Agents
```

So the chapter order matters: structured output → tool calling → agents.

## Tool calling vs plain output parsing

Both let you get structured data out of a model. The difference:

| Aspect                 | Output Parsing                       | Tool Calling                            |
| ---------------------- | ------------------------------------ | --------------------------------------- |
| Where structure lives  | In text the model writes             | In a native "tool call" the model emits |
| Purpose                | Extract data from a response         | Request execution of a capability       |
| Enforced by the model? | No (parser cleans up afterwards)     | Yes (native provider mechanism)         |
| Downstream effect      | Data is used                         | A function is executed                  |

You can parse text to look like a tool call, but native tool calling is more reliable and is what agents rely on.

## Practical considerations

- **Descriptions matter.** The model chooses a tool primarily based on its description. Be precise about **when** the tool should be used and **what** it does.
- **Keep schemas minimal.** Only ask for arguments the tool actually needs. Ambiguous schemas produce bad calls.
- **Validate inputs.** Never trust the arguments blindly — validate before executing anything with side effects.
- **Handle errors gracefully.** Tools can fail; feed the error back to the model so it can decide what to do next.
- **Watch for prompt-injection.** Tools that read arbitrary text (search results, user files) may contain content that tries to manipulate the model. Sanitize and constrain accordingly.
- **Cost.** Every tool call is another round-trip through the model. Design tools that do meaningful work per call, not tiny operations.

## Where tools sit in the mental model

If a plain LLM is a very knowledgeable person answering only from memory, a tool-using LLM is that same person with a computer, a search engine, and a phone. The tools don't make the model smarter — they make the model **capable of acting**.

```text
LLM (thinks)
   +
Tools (act)
   =
An assistant that can do things
```

## Key takeaways

- A **tool** is an external capability with a name, description, schema, and function.
- **Tool calling** is the mechanism by which a model requests a tool at runtime.
- Tool calling is structured output applied to function invocations.
- Tool calling is the prerequisite that makes **agents** possible.
- Define tools with `@tool` (simple), `StructuredTool` (validated), or `BaseTool` (custom).
- `bind_tools()` attaches tools to a model so it can request them.
- A tool call is not executed by the model — your code runs the tool and returns the result as a `ToolMessage`.
- The full turn uses four message types: `SystemMessage`, `HumanMessage`, `AIMessage` (may carry `tool_calls`), and `ToolMessage`.
- Modern models can emit multiple tool calls per turn — execute them (ideally in parallel) and return one `ToolMessage` per call.
- Wrap a retriever as a tool with `create_retriever_tool` to let agents choose when to do RAG.
- Descriptions and schemas are the primary way the model decides which tool to call — invest in them.
- Return errors as tool results rather than swallowing them; let the model react.

### Final mental model

```text
                     LLM
                      │
              ┌───────┴────────┐
              ↓                ↓
        Generate text     Tool call
                                │
                        ┌───────┴───────┐
                        ↓               ↓
                     Tool A          Tool B
                        │               │
                     Execute         Execute
                        └───────┬───────┘
                                ↓
                          Feed result back
                                ↓
                              LLM
                                ↓
                            Answer
```

> **Tools let an LLM act. Tool calling is how it asks.**

---

Next: [Agents](./18-agents.md) — LLMs that use tool calling in a loop to accomplish open-ended tasks.
