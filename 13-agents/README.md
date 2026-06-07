# 13 · Agents — Reasoning + Acting in a Loop

## The plain-English version

So far the model answers in one shot: question in, answer out. But many real tasks can't be done in one shot. "Book me the cheapest flight to Berlin next Friday and add it to my calendar" requires *several* steps, and which steps are needed isn't known in advance — the model has to figure it out as it goes: search flights, compare prices, maybe search again with different dates, then create a calendar event.

**An agent is a language model that runs in a loop, deciding for itself which tools to use and when, until the task is done.** Instead of you hard-coding the steps, the model *plans* them. It thinks ("I need flight prices"), acts (calls the flight tool), observes the result, thinks again ("now I'll add the calendar event"), acts, and finally answers. The model is the brain; the tools are the hands (module 06); the loop is what makes it an *agent* rather than a single call.

This is the payoff of everything you've learned. Messages (03) carry the conversation, tools (06) provide actions, and the agent stitches them into autonomous, multi-step behavior.

## The mental model: the ReAct loop

The dominant pattern is **ReAct** — *Reason + Act*. One iteration:

```
            ┌─────────────────────────────────┐
            ▼                                 │
   ┌─────────────────┐   tool call    ┌──────────────┐
   │   LLM (reason)  │ ─────────────► │  run tool(s) │
   │ "what next?"    │ ◄───────────── │ (act/observe)│
   └─────────────────┘  tool result   └──────────────┘
            │
            │ no tool call → final answer
            ▼
        ✅ done
```

The model looks at the conversation and decides: *do I have enough to answer, or do I need a tool?* If it needs a tool, it emits a tool call (module 06); your runtime executes it and appends a `ToolMessage`; then the model runs again with that new information. The loop ends when the model responds **without** requesting a tool — that's its signal that it's finished.

## Creating an agent

LangChain ships a prebuilt agent so you don't write the loop yourself:

```python
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-4o",
    tools=[search_flights, create_calendar_event],
    prompt="You are a travel assistant. Be efficient.",
)
result = agent.invoke({"messages": [{"role": "user", "content": "Book a flight to Berlin Friday and add it to my calendar"}]})
```

That's it. `create_agent` wires up the reason→act→observe loop, runs tools (including in parallel), manages the message list, and returns when the model is done. Crucially, **this prebuilt agent is built on LangGraph** — under the hood it's a graph with an "LLM" node and a "tools" node and an edge that loops between them. That's why everything in modules 14–20 (state, persistence, human-in-the-loop, streaming) applies directly to agents.

## Going deeper: what a real agent needs beyond the loop

A bare loop is a demo. Production agents add:

- **A system prompt** that defines the agent's role, constraints, and when to stop. This shapes behavior more than anything else.
- **Structured/controlled output** when the final answer must be machine-readable (module 05).
- **Persistence** (module 17) so a long task can survive restarts and the conversation continues across turns — pass a `checkpointer` and a `thread_id`.
- **Human-in-the-loop** (module 18) to approve risky tool calls before they execute.
- **Memory** (module 12) for cross-session recall.
- **Streaming** (module 19) so users see progress instead of staring at a spinner during a multi-step task.
- **Guardrails**: a maximum iteration / recursion limit so a confused agent can't loop forever, plus error handling so a failing tool becomes an observation the agent can recover from rather than a crash.

## The deepest layer: agency, control, and when *not* to use an agent

**Agency is a spectrum.** More autonomy = more capability but less predictability. The art is giving the model exactly as much freedom as the task needs and no more:

- **Fixed chain (no agency)** — you hard-code the steps (LCEL, module 07). Predictable, cheap, fast. Use when the steps are always the same.
- **Router (a little agency)** — the model picks one of a few predefined branches, then a fixed path runs. Good when there are a handful of known cases.
- **Tool-calling agent (full agency)** — the model freely decides the sequence of tool calls. Use when the path genuinely can't be predicted.

A key engineering judgment: **don't reach for a full agent when a chain or router will do.** Agents are more expensive (many model calls), slower, and harder to make reliable and to debug. "Agentic" is not automatically better.

**Cost and latency compound.** Each loop iteration is at least one model call. A 6-step task is 6+ calls — costly and slow. Minimize tools, write tight prompts, and cap iterations.

**Reliability is the hard part.** Agents fail in characteristic ways: looping forever, calling the wrong tool, hallucinating tool arguments, or getting stuck. Mitigations: clear tool descriptions (module 06), few well-chosen tools, recursion limits, strong prompts, evaluation (module 22), and tracing every run (module 21) so you can *see* where reasoning went wrong. You cannot operate agents in production without observability.

**Deep agents and beyond.** For long, complex tasks, patterns like planning (the agent drafts a multi-step plan first), sub-agents (delegating subtasks, module 20), and scratchpad/file-system memory let agents tackle work far beyond a simple loop. These build directly on the same foundation: a model, tools, state, and a loop.

## Common gotchas

- **Using an agent where a chain would do** — pay the agency tax only when the path is genuinely unpredictable.
- **No iteration cap** — a confused agent loops until it burns your budget. Always set a recursion/step limit.
- **Too many tools** — selection accuracy drops past ~10–20 tools; group, route, or retrieve over tools.
- **No persistence on multi-turn agents** — without a checkpointer + `thread_id`, the agent forgets between turns.
- **Flying blind** — running agents without tracing makes failures nearly impossible to diagnose.

## Key takeaways

- An **agent** is an LLM in a **reason→act→observe loop** that chooses its own tools and stops when it answers without a tool call (the **ReAct** pattern).
- `create_agent(model, tools, prompt)` gives you the loop prebuilt — and it's **built on LangGraph**, so state, persistence, HITL, and streaming all apply.
- **Agency is a spectrum** (fixed chain → router → full agent); use the *least* autonomy the task requires, because agents cost more, are slower, and are harder to make reliable.
- Production agents need **system prompts, iteration caps, persistence, human-in-the-loop, evaluation, and tracing** — observability is non-negotiable.

## Check your understanding

**Q1.** What event signals that an agent's ReAct loop should terminate?

- A) The recursion limit is always reached.
- B) A `ToolMessage` is appended.
- **C) The model produces a response that contains *no* tool call — indicating it judges it has enough information to give the final answer.** ✅
- D) The system prompt is exhausted.

*Why:* The loop continues as long as the model requests tools; an assistant message without a tool call is the natural stop condition.

**Q2.** A team builds a full tool-calling agent for a workflow whose steps are actually always the same three operations in the same order. What does the module suggest, and why?

- A) Keep the agent but raise the temperature for creativity.
- **B) Use a fixed chain (or at most a router): full agency adds cost, latency, and unpredictability with no benefit when the path is known — use the least autonomy the task needs.** ✅
- C) Add more tools so the agent has options.
- D) Replace the model with a larger one to stabilize the agent.

*Why:* Agency is a spectrum; paying the "agency tax" is only justified when the sequence of steps genuinely can't be predicted in advance.

**Q3.** Why does the fact that `create_agent` is "built on LangGraph" matter practically?

- A) It means agents can't be customized.
- B) It forces you to use LangSmith.
- **C) Because the agent is really a graph (LLM node ↔ tools node with a loop edge), LangGraph capabilities — state, persistence/checkpointers, human-in-the-loop, streaming — apply directly to it.** ✅
- D) It means agents run only locally.

*Why:* Sitting on LangGraph means the agent inherits durable state, pause/resume, HITL, and streaming — and you can drop to the graph level for custom control flow.

> **Visualization:** `visualization.html` is an interactive ReAct-loop simulator — step the agent through reasoning, tool calls, and observations on a multi-step task, with an iteration counter and a "stuck in a loop" demo.