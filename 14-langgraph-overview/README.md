# 14 · LangGraph Overview — Why Graphs

## The plain-English version

The prebuilt agent (module 13) is great until you need something it doesn't do exactly your way: "after the model answers, run a fact-checker; if it fails, send it back; before any payment tool runs, pause and ask a human; and if the whole thing crashes, resume from where it stopped tomorrow." That's not a simple loop anymore — it's a *flowchart* with branches, cycles, pauses, and the ability to survive a restart.

**LangGraph is a framework for building exactly that flowchart.** You describe your application as a **graph**: boxes (steps) connected by arrows (transitions), with a shared notebook (state) that every step can read and write. LangGraph then runs your graph reliably — streaming progress, saving its place after every step, pausing for humans, and resuming later. It's the "low-level orchestration" layer beneath the friendly agent abstraction.

The mental shift: instead of writing one long function with tangled `if`s and `while`s, you declare *nodes* (what each step does) and *edges* (how to move between them), and hand the whole thing to LangGraph to execute.

## The mental model: nodes, edges, state

Three concepts, and everything else builds on them:

- **State** — a shared data structure (think: a dictionary) that flows through the graph. Every node receives the current state and returns updates to it. It's the graph's memory. (Module 15.)
- **Nodes** — Python functions that do the work. A node takes the state, does something (call a model, run a tool, transform data), and returns a partial update to the state. (Module 16.)
- **Edges** — the wiring that decides which node runs next. **Normal edges** always go A→B. **Conditional edges** run a function on the current state to *choose* the next node — this is how branching and looping happen. (Module 16.)

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    messages: list

builder = StateGraph(State)
builder.add_node("chatbot", call_model)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)
graph = builder.compile()
graph.invoke({"messages": [{"role": "user", "content": "hi"}]})
```

You build the graph, `compile()` it (which validates the wiring), and then it's a **Runnable** — `invoke`, `stream`, `batch` all work, just like everything else in the ecosystem.

## Going deeper: what LangGraph gives you that a plain loop doesn't

The reason to model your app as a graph isn't aesthetics — it's a bundle of production capabilities that come almost for free once your logic is a graph:

- **Durable execution / persistence (module 17).** LangGraph can save a snapshot of the state after *every* node via a **checkpointer**. If the process crashes, you resume from the last checkpoint instead of starting over. This also gives you per-conversation **threads** and **time-travel** debugging (rewind to any past state).
- **Human-in-the-loop (module 18).** Because state is persisted, the graph can *pause* mid-run (an `interrupt`), wait — for seconds or days — for a human to approve, edit, or reject, then *resume* exactly where it left off. Impossible to do cleanly with a normal loop.
- **Streaming (module 19).** Stream state updates, tokens, or custom events as the graph runs, so users see progress through a long task.
- **Controllability.** You decide the exact control flow. Loops, branches, parallel fan-out/fan-in, retries, sub-graphs — all explicit and inspectable, not buried in imperative code.
- **Multi-agent (module 20).** Each agent can be a node or a sub-graph; LangGraph orchestrates how they hand off work.

## The deepest layer: the design philosophy and when to use it

**LangGraph is intentionally low-level.** It does *not* hide the agent loop from you the way `create_agent` does. That's the point: when you need control, you want the machinery exposed. The relationship is layered — `create_agent` (module 13) is a *prebuilt graph* that LangGraph provides for the common case. You start high-level and drop to LangGraph when the prebuilt agent isn't enough. Same ecosystem, no rewrite.

**The Pregel heritage.** LangGraph's execution model is inspired by Google's **Pregel** (and dataflow systems): computation proceeds in discrete **super-steps**. In each step, all active nodes run (potentially in parallel), then their updates are applied to the shared state, then edges decide who's active next. This message-passing model is what makes parallelism, determinism, and checkpointing clean — and why "what runs next" is always a function of state, not hidden control flow.

**LCEL vs. LangGraph — the decisive line (recap of module 07).** Use **LCEL** for linear, stateless data pipelines (prompt → model → parser, RAG chains). Use **LangGraph** the moment you have **cycles**, **durable state**, **human pauses**, or **multiple agents**. A clean test: *"Does my app need to loop, remember across steps, or pause and resume?"* If yes → LangGraph.

**The Functional API alternative.** Besides the graph (`StateGraph`) API, LangGraph offers a **Functional API** (`@entrypoint`, `@task` decorators) that adds persistence and human-in-the-loop to ordinary-looking imperative code, for teams who prefer writing functions over declaring graphs. Same engine, different ergonomics.

**Deployment.** LangGraph apps can run anywhere Python runs, but the **LangGraph Platform** (module 24) adds managed, scalable hosting purpose-built for long-running, stateful agents — with persistence, task queues, and APIs handled for you.

## Common gotchas

- **Reaching for LangGraph too early.** If your flow is a straight line with no state or loops, LCEL is simpler. Don't pay the graph tax without a reason.
- **Forgetting to compile.** A `StateGraph` must be `.compile()`d before you can run it; compilation also validates the wiring.
- **Thinking the agent and LangGraph are different worlds.** The agent *is* a LangGraph graph; everything here applies to it.
- **Designing state poorly.** State design (module 15) is the foundation; a sloppy state schema makes nodes and edges awkward. Plan it first.

## Key takeaways

- **LangGraph models your app as a graph** of **nodes** (steps) and **edges** (transitions) sharing a common **state** — the low-level orchestration layer of the ecosystem.
- Modeling logic as a graph unlocks **durable persistence, human-in-the-loop pause/resume, streaming, explicit control flow, and multi-agent** orchestration.
- A compiled graph is a **Runnable**; `create_agent` is just a **prebuilt graph**, so you move from high-level to low-level without leaving the ecosystem.
- Use **LCEL for linear/stateless** flows and **LangGraph for cycles, durable state, human pauses, or multiple agents**.

## Check your understanding

**Q1.** Which requirement most clearly indicates you should move from an LCEL chain to LangGraph?

- A) You need to swap model providers.
- B) You want to parse the model's output into JSON.
- **C) The workflow must pause for human approval mid-run and resume — possibly days later — from exactly where it stopped.** ✅
- D) You want to template a prompt with variables.

*Why:* Durable pause/resume (and cycles, persistent state, multi-agent) is the LangGraph line. The others are well within LCEL's stateless, linear scope.

**Q2.** In LangGraph's Pregel-inspired execution model, how is "what runs next" determined?

- A) By the order functions were defined in the file.
- B) By hidden imperative control flow inside each node.
- **C) By edges that read the current shared state after each super-step — control flow is an explicit function of state, enabling clean parallelism and checkpointing.** ✅
- D) Randomly, to encourage exploration.

*Why:* Super-step message passing means transitions are computed from state via edges, which is what makes parallelism deterministic and checkpointable.

**Q3.** A developer says "LangGraph and the prebuilt agent are completely separate tools; choosing one rules out the other." Why is this wrong?

- A) Because the agent can't use tools.
- **B) Because `create_agent` is itself a prebuilt LangGraph graph; you can start with it and drop down to custom LangGraph control flow without leaving the ecosystem.** ✅
- C) Because LangGraph only works with LangSmith.
- D) Because the agent runs LCEL, not a graph.

*Why:* The layers are continuous: the agent is a graph, so all LangGraph capabilities apply and you can graduate to the low-level API seamlessly.

> **Visualization:** `visualization.html` lets you toggle between a "simple loop" and a "real graph" (with a conditional branch, a human-pause node, and a checkpoint marker) to see what modeling your app as a graph unlocks.