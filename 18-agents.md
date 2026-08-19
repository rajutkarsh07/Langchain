# Agents

An **Agent** is an LLM that decides — at runtime — which tools to call and in what order to accomplish a task. Agents are where chains stop being fixed pipelines and start behaving like autonomous workers.

## What is an agent?

A regular chain follows a workflow you defined:

```text
A → B → C → D
```

An agent, in contrast, decides what to do next:

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

The developer defines **what tools are available**. The **LLM inside the agent** decides how and when to use them.

## The agent loop

At its core, an agent is a loop:

```text
loop:
    LLM reasons about the request and current state
        ↓
    LLM decides:
       ├─ produce final answer  → stop
       └─ call a tool           → continue
        ↓
    Application executes the tool
        ↓
    Tool result is added to the state
        ↓
    Return to top of loop
```

The loop terminates when the model produces a final answer instead of another tool call.

## Chain vs Agent — the definitive comparison

This is one of the most important distinctions in LangChain.

| Aspect         | Chain                                    | Agent                                        |
| -------------- | ---------------------------------------- | -------------------------------------------- |
| Workflow       | Predefined by the developer              | Chosen dynamically by the LLM at runtime      |
| Predictability | High                                     | Lower — depends on model reasoning            |
| Debuggability  | Straightforward                          | Harder — behavior varies per input            |
| Tool use       | Optional and hardcoded                   | Central — agent picks tools per step          |
| Best for       | Known, repeatable flows                  | Open-ended tasks needing decisions            |

Conceptually:

```text
Chain:
             User Request
                   ↓
              A → B → C → D
                   ↓
                 Answer

Agent:
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

> **Chain = predefined workflow.**
>
> **Agent = dynamically selected workflow/actions.**

Use a chain when you know the steps. Use an agent when you don't.

## Anatomy of an agent

An agent combines four things:

```text
LLM
 +
Tools
 +
Decision Making
 +
Execution Loop
```

- **LLM** — provides reasoning.
- **Tools** — provide capabilities. See [Tools and Tool Calling](./17-tools-and-tool-calling.md).
- **Decision making** — driven by the LLM using the tools' descriptions.
- **Execution loop** — repeatedly asks the LLM for the next action until it produces a final answer.

## Why agents need tool calling

An agent's loop requires the LLM to emit **structured tool calls**. Without them, the agent can't reliably know which tool to execute or with what arguments. This is why the chapter order is:

```text
Structured Outputs
        ↓
Tool Calling
        ↓
Agents
```

An agent is essentially a **loop around tool calling**.

## Example — an agent that solves a task

Suppose the user asks:

> "What is 458 × 923, and what's the weather in the city with the largest population?"

A single LLM call cannot answer this well. An agent can:

```text
User Request
     ↓
Agent
     ↓
LLM: "I need to calculate 458 × 923"
     ↓
Tool call: calculator("458 * 923")
     ↓
Result: 422,734
     ↓
LLM: "Now I need to know the largest city"
     ↓
Tool call: search("world's largest city by population")
     ↓
Result: Tokyo (~37M)
     ↓
LLM: "Now I need Tokyo's weather"
     ↓
Tool call: weather("Tokyo")
     ↓
Result: 22°C, cloudy
     ↓
LLM: produces final answer combining all three
     ↓
Answer
```

Each step is chosen by the LLM based on the current state. No hardcoded flow.

## Where retrievers fit — RAG as a tool

An agent can use a **retriever as a tool**:

```text
Agent
  │
  ├── tool: search_docs (wraps a retriever)
  ├── tool: calculator
  └── tool: send_email
```

When the LLM needs internal knowledge, it calls `search_docs`; it doesn't have to be given the context upfront. This blurs the line between a "RAG app" and an "agent that can retrieve":

- **RAG** — always retrieves, always uses the results.
- **Agent with a retriever tool** — retrieves *when it decides retrieval is needed*.

The second is more flexible; the first is more predictable.

## Design considerations for agents

Agents are powerful and dangerous. Some practical guidance:

- **Give tools clear, precise descriptions.** The model chooses tools based on their descriptions. Ambiguous ones cause bad choices.
- **Constrain the tool set.** Fewer, better-scoped tools produce more reliable agents than a giant catalog.
- **Validate every tool argument.** Never trust the model's arguments blindly, especially for tools with side effects.
- **Bound the loop.** Set a maximum number of iterations; otherwise a confused agent can spin indefinitely.
- **Log everything.** Traceability is critical when behavior varies per input.
- **Watch for prompt injection.** Any tool that returns arbitrary text (search results, file contents) can smuggle instructions to the model.
- **Handle failures.** Feed errors back to the model so it can adapt rather than crashing.
- **Start small.** A two- or three-tool agent that works reliably is worth more than a ten-tool one that mostly doesn't.

## When to use an agent (and when not to)

Agents are the right choice when:

- The path to the answer varies with the input.
- Multiple tools may be needed in different orders.
- You cannot enumerate all possible workflows in advance.
- The model needs to react to intermediate results.

Agents are the wrong choice when:

- The workflow is fixed and known.
- You need strong guarantees on latency, cost, or behavior.
- The task is simple enough for a chain.

If a chain would do — use a chain. Agents are strictly more expensive and less predictable.

## Agents vs Chains vs RAG — one final table

| Pattern | Structure | Best For | Predictability |
| ------- | --------- | -------- | -------------- |
| **Chain**  | Fixed pipeline | Known, repeatable flows | High |
| **RAG**    | Retrieve → prompt → generate | Grounded Q&A over documents | High |
| **Agent**  | Loop around tool calls | Open-ended, multi-step tasks | Lower |

All three are built on the same primitives: models, prompts, output parsers, runnables. They differ in how much decision-making they delegate to the LLM.

## Where agents sit in the mental model

If a chain is a script an engineer wrote, and RAG is a script that first fetches the right paragraph, an **agent is an employee**. You tell them the goal, give them a set of tools, and trust them to decide the steps. That autonomy is powerful — and it is exactly why agents also require more care.

## Key takeaways

- An agent is an LLM that loops around tool calls to accomplish a task.
- Agents combine **LLM + Tools + Decision Making + Execution Loop**.
- Tool calling (built on structured output) is the prerequisite mechanism.
- **Chains** follow fixed paths; **agents** decide paths at runtime.
- Agents are the most flexible LangChain pattern — and the most expensive to run and hardest to debug.
- Use a chain when you can; use an agent when you can't.

### Final mental model

```text
                       AGENT
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
      LLM             Tools             Loop
   reasoning         act on            call → observe
                     the world         → decide → repeat
       └─────────────────┼─────────────────┘
                         ↓
                Autonomous LLM Application
```

### The one sentence to remember

> **An agent is an LLM given a set of tools and permission to decide, at each step, which one to call next.**

---

You've reached the end of this documentation. From here, the recommended next step is to build something — a small RAG app, a simple tool-using agent, or a chained pipeline that uses everything from Chapter II. The concepts here compose in more ways than any documentation can enumerate.

Back to the [README](./README.md) for the full table of contents.
