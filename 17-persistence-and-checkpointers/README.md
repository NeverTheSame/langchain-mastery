# 17 · Persistence & Checkpointers — Durable, Resumable Agents

## The plain-English version

Run a normal program, and when it ends — or crashes — its memory vanishes. For a quick question that's fine. But agents do *long* things: a research task that takes ten minutes, a conversation that spans days, a workflow that pauses to wait for your approval. If the server restarts in the middle, you don't want to lose all that progress and start over.

**Persistence is LangGraph's ability to save its progress so a run can survive interruptions, span multiple turns, and be resumed later.** The mechanism is a **checkpointer**: after every step, LangGraph snapshots the entire state and writes it to storage. Think of it as a video game's autosave — the game saves after each level, so if the power goes out you reload from the last save instead of the beginning.

This one capability is what turns a fragile script into a production-grade, stateful application. It's the foundation that makes memory (module 12), human-in-the-loop (module 18), and reliable long-running agents possible.

## The mental model: checkpoints + threads

Two ideas work together:

**A checkpointer** saves a snapshot of the graph's **state** (module 15) after each super-step. You attach one at compile time:

```python
from langgraph.checkpoint.memory import InMemorySaver

graph = builder.compile(checkpointer=InMemorySaver())
```

**A thread** is one conversation/session, identified by a `thread_id`. All the checkpoints for a thread form its history. You pass the thread in a config:

```python
config = {"configurable": {"thread_id": "user-42"}}
graph.invoke({"messages": [HumanMessage("hi")]}, config)
graph.invoke({"messages": [HumanMessage("what did I just say?")]}, config)  # remembers!
```

Because the second call uses the same `thread_id`, LangGraph **loads the saved state** for that thread before running — so the graph already "remembers" the first message. Change the `thread_id` and you get a fresh, independent conversation. **This is how short-term memory actually works** (module 12): it's persistence keyed by thread.

## Going deeper: what persistence enables

- **Multi-turn conversations.** Each user turn is a separate `invoke`, but the shared `thread_id` makes them continuous. No need to manually replay history — the checkpointer restores it.
- **Fault tolerance / durable execution.** If the process dies mid-run, resuming with the same thread continues from the last checkpoint, not from scratch. Long, expensive tasks become robust.
- **Human-in-the-loop (module 18).** To pause for human input, the graph must *persist* its state while it waits — possibly for days. Checkpointing is the precondition for `interrupt`/resume.
- **Time travel.** Because every step is saved, you can list a thread's checkpoint history, inspect the state at any past point, and even *resume from* an earlier checkpoint — to debug, or to explore an alternate path ("what if the model had chosen differently here?").

## The deepest layer: checkpointer backends, threads, and stores

**Checkpointer backends (pick by durability needs):**

- **`InMemorySaver`** — keeps checkpoints in RAM. Zero setup, perfect for development, tests, and notebooks — but everything is lost on restart. Never use for production.
- **`SqliteSaver`** — a local file. Persists across restarts; great for single-machine apps and prototypes that need durability.
- **`PostgresSaver`** (and other DB-backed savers) — production-grade: durable, concurrent, scalable. What you deploy with. The LangGraph Platform (module 24) manages this for you.

The interface is uniform, so — like everything in the ecosystem — you develop on `InMemorySaver` and switch to Postgres by changing one line.

**Inspecting and steering state:**

- `graph.get_state(config)` — read the current state (and which node is next) for a thread.
- `graph.get_state_history(config)` — list all checkpoints (the time-travel log).
- `graph.update_state(config, values)` — *manually* write to a thread's state, e.g. to inject a correction before resuming. This is how a human edit (module 18) lands.

**Checkpoints vs. the store (the two persistence layers).** Don't confuse them:

- **Checkpointer** = *per-thread* short-term state (this conversation's messages and working data). Restored automatically each turn.
- **Store** (`BaseStore`, module 12) = *cross-thread* long-term memory (facts about the user that should be recalled in *other* conversations). Queried explicitly.

A production agent typically uses **both**: a checkpointer for "remember this conversation" and a store for "remember this user across all conversations."

**Sub-graphs and nested state.** Checkpointing works through subgraphs (module 20), so even a multi-agent system can be paused and resumed coherently — each level's state is captured.

**Cost and hygiene.** Every checkpoint is written to storage, so very high-frequency graphs accumulate data. Production setups manage retention (TTLs, cleanup) and keep state lean (module 15) so checkpoints stay small.

## Common gotchas

- **No checkpointer = no memory.** Without one, every `invoke` starts blank — the agent forgets between turns. The #1 "why doesn't my agent remember?" cause.
- **Reusing `thread_id` across unrelated users** mixes their conversations. Make thread IDs unique per session/user.
- **`InMemorySaver` in production** silently loses everything on restart. Use a durable backend.
- **Confusing checkpointer with store.** Per-thread state ≠ cross-thread memory; you usually need both.
- **Bloated state** makes every checkpoint heavy. Keep large artifacts out of state and reference them.

## Key takeaways

- **Persistence** = a **checkpointer** snapshots the graph's **state after every step**, so runs survive crashes, span turns, and can be resumed.
- A **`thread_id`** identifies a conversation; reusing it restores that thread's saved state — which is exactly **how short-term memory works**.
- Persistence unlocks **multi-turn chat, fault tolerance, human-in-the-loop pause/resume, and time-travel** debugging; inspect/steer via `get_state`, `get_state_history`, `update_state`.
- Choose a backend by durability (**InMemory → SQLite → Postgres**); and remember the **two layers**: checkpointer (per-thread) vs. store (cross-thread long-term memory).

## Check your understanding

**Q1.** An agent answers the first question fine, but on the second turn it acts as if the first never happened. The graph compiles and runs without error. Most likely cause?

- A) The reducer on `messages` is wrong.
- B) The model provider rate-limited the request.
- **C) No checkpointer was attached (or a different `thread_id` was used), so the second `invoke` started from blank state instead of restoring the conversation.** ✅
- D) The recursion limit was hit.

*Why:* Cross-turn memory comes from persistence keyed by `thread_id`; without a checkpointer or with a changed thread ID, state isn't restored.

**Q2.** Why is a checkpointer the *precondition* for human-in-the-loop pause/resume?

- A) Because humans can only approve in-memory data.
- **B) Because pausing means waiting (possibly days); the graph must persist its state while idle so it can be faithfully resumed from exactly where it stopped.** ✅
- C) Because interrupts disable the reducer.
- D) Because the store handles all pausing automatically.

*Why:* You can't pause and later resume a run unless its state was durably saved; checkpointing is what makes the wait survivable.

**Q3.** Which statement correctly distinguishes the checkpointer from the store?

- A) They're the same thing with different names.
- B) The checkpointer holds cross-user facts; the store holds the current conversation.
- **C) The checkpointer persists *per-thread* short-term state (restored each turn); the store holds *cross-thread* long-term memory (queried explicitly) — production agents often use both.** ✅
- D) The store replaces the need for a checkpointer.

*Why:* They are two distinct persistence layers: per-conversation state vs. cross-conversation memory.

> **Visualization:** `visualization.html` simulates an autosaving agent — run it, "crash" it mid-task, and resume from the last checkpoint; switch threads to see independent memories; and scrub the time-travel slider through past checkpoints.