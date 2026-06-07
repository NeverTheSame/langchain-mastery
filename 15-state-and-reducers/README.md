# 15 · State & Reducers — The Shared Memory of a Graph

## The plain-English version

Every node in a LangGraph graph needs to share information. The model node produces a message; the tool node needs to see it; the final node needs the whole conversation. Where does that shared information live? In the graph's **state** — a single data structure that flows through every node, like a clipboard passed around a room where each person reads it and writes their part.

But sharing raises a subtle question: when two nodes both want to update the *same* field, what should happen? If the state has a `messages` list and a node produces a new message, should it **replace** the whole list or **append** to it? Almost always you want to append — you don't want each node to erase the conversation. A **reducer** is the rule that answers "how do updates to this field get combined with what's already there." State defines *what* the graph remembers; reducers define *how* that memory changes.

Getting state and reducers right is the single most important design decision in a LangGraph app. Everything — nodes, edges, persistence, memory — revolves around it.

## The mental model: a typed schema + per-field merge rules

You declare the state as a schema (usually a `TypedDict`, sometimes a Pydantic model or dataclass):

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]   # reducer: append, don't replace
    user_name: str                            # no reducer: replace
    step_count: int
```

- `messages` uses the `add_messages` **reducer** (via `Annotated[..., reducer]`), so when any node returns `{"messages": [new_msg]}`, that message is *appended* to the existing list, not overwritten.
- `user_name` has no reducer, so the **default behavior is "replace"** — a node returning `{"user_name": "Kirill"}` overwrites whatever was there.

**Nodes return partial updates, not the whole state.** A node returns a dict containing only the fields it changed; LangGraph merges each field using that field's reducer. This is why nodes stay simple and composable.

## Going deeper: how reducers actually work

A reducer is just a function `(current_value, update) -> new_value`. LangGraph calls it for each field whenever a node returns an update:

- **Default reducer (no annotation):** `new_value = update` — last write wins (replace).
- **`add_messages`:** intelligently appends messages. It's smarter than a plain `+`: it **deduplicates by message ID** (so re-emitting a message doesn't duplicate it) and supports **updating** an existing message (by ID) and **removing** messages (via `RemoveMessage`). This is what makes streaming, summarization, and message-editing clean.
- **`operator.add`:** plain concatenation/addition — append to a list or sum a number.
- **Custom reducers:** write your own to, say, merge dicts, keep a max, or maintain a set. Whatever combination logic your field needs.

**Why reducers matter for correctness.** Without the right reducer, parallel nodes that both write the same field would clobber each other or raise an error. The reducer is what lets two branches running in the same super-step *both* contribute to `messages` (their updates are merged) instead of one silently winning.

## The deepest layer: advanced state design

**The `MessagesState` shortcut.** Because nearly every graph has a `messages` field with `add_messages`, LangGraph provides a prebuilt `MessagesState` you can subclass and extend:

```python
from langgraph.graph import MessagesState

class State(MessagesState):     # already has messages + add_messages
    documents: list             # add your own fields
    needs_review: bool
```

**Multiple schemas: input, output, and private state.** A graph can distinguish:

- **Input schema** — what callers must provide.
- **Output schema** — what the graph returns (hide internal scratch fields from the caller).
- **Overall/internal state** — extra fields nodes use to pass scratch data among themselves but that aren't part of the public contract.

This lets you keep a clean public interface while nodes share private working data.

**State is what gets persisted.** The checkpointer (module 17) snapshots the *state* after each super-step. So your state schema *is* your durable memory and the thing you "time travel" through. Design it deliberately: put in it everything that must survive a pause/restart, and keep out giant blobs you don't need to persist.

**State vs. long-term memory.** State is *per-thread* (one conversation). Cross-thread, long-term memory (module 12) lives in a separate **store**, not in state. Don't try to cram cross-session facts into per-thread state.

**`Command`: update state *and* route in one move.** A node can return a `Command(update={...}, goto="next_node")` to both write state and decide the next node — useful for dynamic control flow and multi-agent handoffs (module 20). It blurs the node/edge line on purpose for expressiveness.

## Common gotchas

- **Forgetting the `add_messages` reducer** is the classic LangGraph bug: each node *replaces* `messages`, so the conversation keeps getting wiped and the model only ever sees the latest message. Always annotate `messages`.
- **Returning the whole state from a node** instead of a partial update — works, but fights the design and can clobber fields. Return only what changed.
- **Parallel writes without a reducer** to a non-list field cause conflicts. Give concurrently-written fields a reducer that merges.
- **Putting un-persistable or huge objects in state** bloats every checkpoint. Keep state lean; store large artifacts elsewhere and reference them.
- **Confusing state (per-thread) with the store (cross-thread).** They're different layers.

## Key takeaways

- **State** is the typed, shared data structure that flows through every node; **nodes return partial updates**, merged into state field-by-field.
- A **reducer** defines *how* a field's updates combine with its current value; the default is **replace**, while **`add_messages`** appends (with dedup/update/remove by ID).
- Use **`MessagesState`** as a base, and consider **separate input/output/internal schemas** to keep a clean public contract.
- State is **what the checkpointer persists** (per-thread durable memory); cross-session facts belong in a **store**, not state. **`Command`** can update state and route at once.

## Check your understanding

**Q1.** A LangGraph beginner finds that on every turn the model only sees the *latest* user message — the conversation never accumulates. What's the most likely cause?

- A) They forgot to call `.compile()`.
- B) Their checkpointer is misconfigured.
- **C) The `messages` field lacks the `add_messages` reducer, so each node's update *replaces* the list instead of appending — wiping the history every step.** ✅
- D) They used `MessagesState` instead of a `TypedDict`.

*Why:* Without a reducer the default is "replace." Annotating `messages` with `add_messages` makes updates append, preserving the conversation.

**Q2.** Why is `add_messages` described as smarter than a plain list concatenation (`+`)?

- A) It encrypts messages in transit.
- B) It sorts messages alphabetically.
- **C) It deduplicates by message ID and supports updating or removing messages by ID (e.g. via `RemoveMessage`), which makes streaming, summarization, and edits clean.** ✅
- D) It automatically trims to the context window.

*Why:* ID-aware merging (dedupe/update/remove) is what enables re-emitting chunks during streaming and rewriting history during summarization without duplicates.

**Q3.** Two branches run in the same super-step and both append to `messages`. Why does this work without one clobbering the other?

- A) LangGraph runs them sequentially, so there's no conflict.
- B) Only the first branch's update is kept; the second is discarded.
- **C) The field's reducer (`add_messages`) merges both updates into the existing list, so concurrent contributions combine instead of overwriting.** ✅
- D) Parallel writes are forbidden, so this never happens.

*Why:* Reducers are precisely what make concurrent writes to the same field safe — they define how multiple updates merge in a super-step.

> **Visualization:** `visualization.html` lets you fire node updates at a live state object and toggle the reducer between "replace" and "append" to watch the difference — including a parallel-write demo.