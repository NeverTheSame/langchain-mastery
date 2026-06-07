# 12 · Memory & Chat History — Making Applications Remember

## The plain-English version

By default, a language model has amnesia. Each API call is independent — it doesn't remember the previous message, your name, or what you decided two turns ago. That's fine for a one-shot question, but useless for a conversation. If you tell a chatbot "my name is Kirill" and then ask "what's my name?", a memoryless model has no idea.

**Memory is how you give an application the ability to remember across turns** (and even across sessions, days later). The core trick is almost embarrassingly simple: since the model only sees the list of messages you send it, "remembering" just means *including the relevant past messages in the next request.* Memory is the discipline of deciding **what** to keep, **how** to store it, and **how much** to send back — because you can't just send the entire history forever.

## The mental model: short-term vs. long-term

Two fundamentally different kinds of memory, often confused:

**Short-term (within-conversation) memory** = the message history of the *current* thread. "What did we say earlier in this chat?" It lives in the conversation's state and is bounded by the context window. In modern LangChain/LangGraph this is just the running list of messages held in the graph's **state** (module 15), persisted per-thread by a **checkpointer** (module 17).

**Long-term (cross-conversation) memory** = durable facts the assistant should recall *across* separate sessions. "The user prefers metric units." "Last week they were debugging a payment bug." This isn't kept in the message list; it's saved to a **store** (a database, often a vector store) and *retrieved* when relevant — essentially RAG over the user's own history.

```
Short-term:  this chat's messages  → held in state, persisted per thread
Long-term:   facts about the user  → saved to a store, retrieved on demand
```

## Going deeper: managing short-term memory (the context window problem)

You can't send an ever-growing history to the model forever — you'll blow the context window and your budget. So short-term memory is really a *summarization/compression* problem. The main strategies:

**1. Keep the full history** (do nothing). Simple, works until the conversation gets long, then breaks or gets expensive.

**2. Windowing / trimming.** Keep only the last N messages or last N tokens. Use the message-aware `trim_messages` (module 03) so you always keep the system message and never sever a tool-call/result pair. Cheap and robust; loses old context.

**3. Summarization.** When history grows past a threshold, ask a model to *summarize* the older turns into a compact paragraph, replace those messages with the summary, and keep recent turns verbatim. Preserves the gist of long conversations at a fraction of the tokens. In LangGraph you implement this as a node that rewrites the message list (often using `RemoveMessage` to delete the compacted turns).

**4. Hybrid.** Summarize the distant past + keep a verbatim window of the recent past. The common production choice.

## The deepest layer: long-term memory and the modern approach

**The historical note.** Older LangChain had explicit `Memory` classes (`ConversationBufferMemory`, `ConversationSummaryMemory`, etc.) you attached to a chain. These are now largely **superseded**: in the LangGraph era, short-term memory is handled by **state + checkpointers**, which is more flexible and durable. If you see old `ConversationBufferMemory` tutorials, mentally translate them to "message history in graph state."

**Long-term memory in practice.** To remember across sessions you need three operations: **write** (decide a fact is worth saving and store it), **read/retrieve** (pull relevant facts into the current context), and **manage** (update or forget stale facts). LangGraph provides a **`Store`** abstraction (e.g. `InMemoryStore`, or a persistent/vector-backed store) keyed by user/namespace, so an agent can save and look up memories. Patterns:

- **Semantic memory** — facts about the user/world ("works in fintech," "prefers terse answers"), often stored as embeddings and retrieved by similarity.
- **Episodic memory** — records of past interactions/events ("on June 2 we fixed the login bug").
- **Procedural memory** — learned how-to/behavioral instructions the agent should follow.

**Who decides what to remember?** Two styles: *hot-path* (the agent explicitly calls a "save_memory" tool during the conversation) or *background* (a separate process reflects on the conversation afterward and extracts durable facts). The background approach keeps the live conversation fast.

**Memory is not free or risk-free.** Every remembered token costs money and context budget, and stale memories cause confidently-wrong answers ("you said you use Postgres" — three projects ago). Good memory systems prune, version, and timestamp facts, and prefer retrieving a *relevant subset* over dumping everything.

## Common gotchas

- **Confusing short-term and long-term memory.** Trimming/summarization solves the context-window problem *within* a chat; it does nothing for cross-session recall, which needs a store.
- **Sending the whole history forever.** Works in demos, explodes in production on cost and latency, and eventually exceeds the context window.
- **Naive trimming breaks tool pairs.** Use message-aware trimming so you don't orphan a `ToolMessage` (module 03).
- **Hoarding long-term memories.** Unbounded, un-pruned memory degrades into noise and contradictions. Curate and expire.
- **Reaching for deprecated `Memory` classes.** Prefer state + checkpointer (short-term) and a store (long-term).

## Key takeaways

- Models are stateless; "memory" means **including the right past information in the next request.**
- **Short-term** memory = current-thread messages held in **state** and persisted per-thread by a **checkpointer**; manage the context window with **trimming**, **summarization**, or a **hybrid**.
- **Long-term** memory = durable facts saved to a **store** and **retrieved** when relevant (RAG over the user's own history), in semantic/episodic/procedural flavors.
- The old `Memory` classes are largely superseded by **state + checkpointers + stores**; curate memories to control cost and avoid stale, confidently-wrong recall.

## Check your understanding

**Q1.** A user returns a week later and expects the assistant to recall a preference they stated in a *previous* session. Trimming and summarization of the current thread won't help. Why, and what does?

- A) They would help if the window were larger; just raise the token limit.
- **B) Trimming/summarization only manage the *current* conversation's context window; cross-session recall requires saving facts to a long-term *store* and retrieving them — distinct from short-term memory.** ✅
- C) The checkpointer automatically copies all messages into every future thread.
- D) Long-term recall is impossible without fine-tuning the model.

*Why:* Short-term techniques operate within a thread; durable cross-session facts must live in a store and be retrieved, conceptually RAG over the user's history.

**Q2.** In the LangGraph era, how is *short-term* conversational memory primarily implemented, and what does this say about the old `ConversationBufferMemory` classes?

- A) Via a global cache; the old classes are still the recommended default.
- **B) As the running message list in the graph's *state*, persisted per-thread by a *checkpointer*; the old `Memory` classes are largely superseded by this approach.** ✅
- C) By embedding every message into a vector store on each turn.
- D) By the provider, which stores history server-side automatically.

*Why:* State + checkpointers handle durable per-thread history more flexibly than the legacy `Memory` classes, which you should mentally translate to "messages in state."

**Q3.** A team enables aggressive long-term memory and starts getting confidently-wrong answers referencing outdated facts ("you use Postgres" — from a project two quarters ago). What's the lesson?

- A) Long-term memory should never be used in production.
- B) The embedding model is too small.
- **C) Memory isn't free or risk-free: stale, un-pruned facts cause wrong answers, so good systems timestamp, version, prune/expire, and retrieve a relevant subset rather than dumping everything.** ✅
- D) They should switch from semantic to procedural memory to fix accuracy.

*Why:* Unbounded memory degrades into noise/contradictions; curation (expiry, timestamps, selective retrieval) is essential to keep recall accurate and costs bounded.

> **Visualization:** `visualization.html` lets you watch a conversation grow and apply trimming vs. summarization live, while separately writing a fact to long-term memory and retrieving it in a brand-new session.