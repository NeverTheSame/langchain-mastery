# 16 · Nodes, Edges & Control Flow — Wiring the Graph

## The plain-English version

If state (module 15) is the *what* a graph remembers, nodes and edges are the *what happens* and the *what happens next*. A **node** is a single step — a box on the flowchart that does one job. An **edge** is an arrow connecting boxes — it decides where to go after a step finishes. Wire nodes together with edges and you've described the entire behavior of your application as a diagram you can read.

The power comes from the two flavors of arrow. A **normal edge** is a fixed arrow: "after A, always go to B." A **conditional edge** is a smart arrow: "after A, look at the current state and *decide* whether to go to B, C, or stop." Conditional edges are how a graph branches, loops, and reacts — they're what separate a real application from a straight pipeline.

## The mental model: functions as nodes, routing as edges

**A node is a function.** It takes the current state and returns a partial update (module 15):

```python
def call_model(state: State) -> dict:
    response = model.invoke(state["messages"])
    return {"messages": [response]}     # partial update, merged via reducer

builder.add_node("model", call_model)
```

**Edges connect nodes.** Two special markers, `START` and `END`, mark where execution begins and where it stops.

```python
builder.add_edge(START, "model")     # entry point
builder.add_edge("model", END)       # exit
```

**Conditional edges route based on state.** You give a *routing function* that inspects the state and returns the name of the next node:

```python
def should_continue(state: State) -> str:
    last = state["messages"][-1]
    return "tools" if last.tool_calls else END    # branch on whether a tool was requested

builder.add_conditional_edges("model", should_continue, ["tools", END])
```

This three-piece pattern — a **model node**, a **tools node**, and a **conditional edge** that loops back to the model after tools run — *is* the agent (module 13). You've just seen what `create_agent` builds for you.

## Going deeper: the control-flow toolkit

**Loops.** A cycle is just an edge pointing "backward": `tools → model`. Combined with a conditional edge `model → (tools or END)`, you get the agent loop. LangGraph allows cycles (unlike a plain LCEL chain), which is the whole reason it exists. A **recursion limit** caps total super-steps so a runaway loop can't run forever.

**Branching / parallelism (fan-out and fan-in).** If a node has edges to *several* nodes, those nodes run **in parallel** in the next super-step. Their state updates merge via reducers (module 15). A later node with edges *from* all of them runs once they've all finished (fan-in). This is how you run independent work concurrently — e.g. query three retrievers at once, then combine.

**Entry and finish points.** `add_edge(START, "x")` sets where input enters; routing to `END` stops that branch. A graph can have multiple paths to `END`.

**`Command` — combine update + routing.** Instead of a separate node and conditional edge, a node can return `Command(update={...}, goto="next")` to write state *and* choose the next node in one move. Handy for dynamic routing and multi-agent handoffs (module 20).

**`Send` — dynamic fan-out (map-reduce).** When you don't know *how many* parallel branches you need until runtime — e.g. "process each of these N retrieved documents in parallel" — a conditional edge can return a list of `Send("node", payload)` objects, spawning one instance of a node per item with its own input. This is LangGraph's map-reduce primitive.

## The deepest layer: design patterns and subtleties

**Common topologies you'll build:**

- **Linear pipeline:** A → B → C → END. (When it's this simple and stateless, LCEL is often enough — module 07.)
- **Branch/router:** START → classifier → (one of several handlers) → END. The classifier node sets a state field; a conditional edge routes on it.
- **Agent loop:** START → model ⇄ tools → END. The canonical cyclic graph.
- **Map-reduce:** START → splitter → (Send fan-out to N workers) → aggregator → END.
- **Evaluator-optimizer:** generator → critic → (loop back if critic rejects, else END). Self-correction.

**Determinism and super-steps.** Execution proceeds in discrete super-steps (the Pregel model, module 14): all currently-active nodes run, their updates apply, then edges decide the next active set. Within a super-step, parallel nodes don't see each other's updates — they only land afterward. Understanding this prevents "why didn't node B see node A's change?" confusion (they ran in the same step).

**Nodes should be focused.** Keep each node doing one clear thing (call a model, run tools, transform data, decide). Fat nodes with hidden branching defeat the purpose — push branching into *edges* where it's visible and inspectable. This is what makes the graph debuggable and traceable (module 21).

**Subgraphs.** A whole compiled graph can be used as a *node* inside a bigger graph. This is how you compose complex systems and build multi-agent architectures (module 20): each agent is a subgraph, wired together by edges in a parent graph.

## Common gotchas

- **No recursion limit on a loop** → infinite execution and runaway cost. Set one.
- **Expecting parallel siblings to see each other's updates** within the same super-step — they don't; updates merge *after* the step.
- **Branching logic hidden inside a node** instead of expressed as a conditional edge — makes the flow opaque and hard to trace. Route in edges.
- **Returning the next node name that isn't in the allowed list** of a conditional edge → routing error. Keep the routing function's outputs and the edge's target list in sync.
- **Forgetting fan-in.** If three parallel branches must all complete before a summary node, make sure the summary node has edges from all three (or they'll race).

## Key takeaways

- **Nodes are functions** (state → partial update); **edges** decide what runs next. `START`/`END` mark entry and exit.
- **Normal edges** are fixed (A→B); **conditional edges** route on the current state — enabling **branching and loops**, the things LCEL can't do.
- Multiple outgoing edges run nodes **in parallel** (fan-out), merging via reducers; **`Send`** does dynamic, runtime-sized fan-out (map-reduce); **`Command`** fuses update + routing.
- Keep nodes **focused** and push branching into **edges**; compose complexity with **subgraphs**. Always cap loops with a **recursion limit**.

## Check your understanding

**Q1.** You need to process an unknown number of retrieved documents — determined at runtime — each in its own parallel branch, then aggregate. Which primitive fits?

- A) A normal edge to a single "process" node.
- B) A `Command(goto=...)` from each document.
- **C) A conditional edge returning a list of `Send("worker", doc)` objects — LangGraph's dynamic fan-out (map-reduce) primitive that spawns one worker per item.** ✅
- D) Increasing the recursion limit.

*Why:* `Send` is built for runtime-sized parallel fan-out where the number of branches isn't known until you inspect state.

**Q2.** Two sibling nodes run in parallel in the same super-step. Node B's code reads a field that node A updates. B doesn't see A's change. Why is this *expected*?

- A) Because B has a bug.
- B) Because parallel nodes are actually run sequentially.
- **C) Because within a super-step nodes run on the same input snapshot; their updates are merged into state only *after* the step completes, so siblings don't see each other's writes mid-step.** ✅
- D) Because reducers delete A's update.

*Why:* The Pregel super-step model applies updates between steps; same-step parallel nodes operate on the pre-step state.

**Q3.** Why does the module recommend pushing branching logic into conditional *edges* rather than hiding `if/else` inside a node?

- A) Edges run faster than nodes.
- B) Nodes cannot contain `if` statements.
- **C) Routing expressed in edges is explicit and inspectable — it keeps the control flow visible and traceable, which is the point of modeling the app as a graph.** ✅
- D) Conditional edges are required for `.compile()` to succeed.

*Why:* Visible, edge-level routing makes the graph debuggable and traceable; fat nodes with hidden branching defeat the graph model's main benefit.

> **Visualization:** `visualization.html` is a graph builder — toggle between linear, branch, agent-loop, and map-reduce topologies and watch tokens of execution flow through nodes and conditional edges super-step by super-step.