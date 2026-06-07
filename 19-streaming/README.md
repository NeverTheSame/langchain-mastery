# 19 · Streaming — Emitting Progress Live

## The plain-English version

Imagine asking an assistant a question and staring at a blank screen for 15 seconds while it works, then suddenly the full answer appears. It feels broken — you don't know if it's thinking, stuck, or dead. Now imagine instead that words appear one at a time as they're generated, and you can see "searching the web…", "found 3 results…", "writing answer…" as it goes. That second experience is **streaming**, and it's the difference between an app that feels responsive and one that feels frozen.

**Streaming is emitting output incrementally — as it's produced — instead of waiting for the whole thing to finish.** For a single model call, that means tokens appearing live (the "typewriter" effect). For an agent that runs many steps, it also means surfacing *what step it's on* so users see real-time progress through a long task. It's both a UX feature and a transparency feature.

## The mental model: stream instead of invoke

Every Runnable (module 07) and every LangGraph graph supports streaming. Instead of `invoke` (run, return once at the end), you call `stream` (run, yield pieces as they arrive):

```python
for chunk in graph.stream(inputs, config):
    print(chunk)        # arrives progressively, not all at once
```

For a plain chain, streaming gives you token chunks. For a graph, you choose *what* you want streamed via **stream modes** — that's the key concept, because an agent has several different things worth streaming.

## Going deeper: LangGraph's stream modes

A graph run produces multiple kinds of events. You pick which to receive (you can request several at once):

- **`values`** — the *full state* after each super-step. Good when you want the complete, updated snapshot at each step (e.g. the whole message list so far).
- **`updates`** — only the *delta* each node produced (just what changed). Lighter than `values`; ideal for "node X did Y" progress logs.
- **`messages`** — streams **LLM tokens** as they're generated, tagged with which node/message they belong to. This is what powers the live typewriter effect inside an agent — even though the agent is a graph, you still get token-by-token output from its model nodes.
- **`custom`** — data *you* emit from inside a node via a writer, for arbitrary progress signals ("downloaded 4/10 files"). Your own bespoke events.
- **`debug`** — maximally detailed events for tracing/diagnostics.

```python
for mode, chunk in graph.stream(inputs, config, stream_mode=["updates", "messages"]):
    ...
```

This multi-mode design is powerful: in one run you can simultaneously stream the assistant's tokens (`messages`) *and* high-level step updates (`updates`) to drive a rich UI — a chat bubble that types itself plus a status line showing tool activity.

## The deepest layer: events, async, and why streaming composes

**`astream_events` — the firehose.** Beyond `stream`, there's `astream_events`, which emits a fine-grained event stream for *every* component start/end in the run: model start, each token, tool start/end, retriever calls, chain steps. Each event carries a name, tags, and metadata, so you can filter to exactly the components you care about and build sophisticated, real-time UIs (e.g. show a spinner labeled with the specific tool currently running). It's the most granular view of execution.

**Streaming is mostly async-native.** Token streaming and event streaming shine with async (`astream`, `astream_events`) in real web servers, where you forward chunks to the browser over SSE/WebSockets as they arrive. The sync `stream` works too, but production streaming UIs are typically async.

**Why streaming "just works" everywhere (recap of module 07).** Because every component implements the Runnable streaming contract, streaming *composes* through a chain: tokens from a model flow through a `StrOutputParser`, and `JsonOutputParser` can even emit *partial* parsed objects as the JSON forms. You wrote no streaming code; the shared interface propagates it end to end. The same property lets a LangGraph node that internally calls a model surface those tokens up through the graph's `messages` stream.

**Streaming and the user's perception of speed.** Time-to-first-token matters more than total time for perceived responsiveness. Even if an answer takes 8 seconds total, showing the first words in 400ms makes the app feel fast. Streaming is one of the cheapest, highest-impact UX upgrades you can ship.

**Interplay with other features.** Streaming pairs naturally with: agents (module 13) — stream the reason→act→observe steps so users watch the agent work; human-in-the-loop (module 18) — stream up to the interrupt, pause, then stream the resumption; and structured output (module 05) — stream partial JSON for progressive form-filling.

## Common gotchas

- **Using `invoke` when users are waiting.** A multi-second silent call feels broken. Default to streaming for any user-facing latency.
- **Picking the wrong stream mode.** Want token-by-token text from an agent? You need `messages`, not `values`/`updates` (those give state/deltas, not tokens). Mixing these up is the most common confusion.
- **Forgetting partial structured output can't validate yet.** Streaming JSON gives *partial* dicts; a fully-validated Pydantic object only exists at the end (module 05).
- **Sync streaming in an async server** can block the event loop. Use the async variants (`astream`, `astream_events`) in async contexts.
- **Over-streaming `debug`/all events** to the client floods the UI; filter `astream_events` to what you actually render.

## Key takeaways

- **Streaming** emits output incrementally — live tokens for responsiveness and step-by-step progress for transparency — instead of one big wait.
- Use **`stream`** instead of `invoke`; for graphs, pick **stream modes**: **`values`** (full state), **`updates`** (deltas), **`messages`** (LLM tokens), **`custom`** (your events), **`debug`**. You can request several at once.
- **`astream_events`** is the fine-grained firehose for building rich real-time UIs; streaming is mostly **async-native** in production.
- Streaming **composes for free** through the Runnable interface (partial JSON, tokens through parsers), and it's a low-effort, high-impact UX win — optimize **time-to-first-token**.

## Check your understanding

**Q1.** You're building a chat UI on top of a LangGraph agent and want the assistant's text to appear token-by-token *and* a status line showing which tool is running. Which stream configuration fits?

- A) `stream_mode="values"` only.
- B) `invoke` with a callback.
- **C) Request multiple modes, e.g. `stream_mode=["messages", "updates"]` — `messages` gives LLM tokens for the typewriter effect; `updates` gives per-node deltas for the status line.** ✅
- D) `stream_mode="debug"` and parse everything client-side.

*Why:* `messages` streams tokens; `updates` streams node deltas. Requesting both drives the dual UI; `values` gives full state (not tokens), and `debug` floods the client.

**Q2.** A developer uses `stream_mode="updates"` and is confused that they don't get token-by-token text from the model. What's the explanation?

- A) `updates` is broken for models.
- **B) `updates` streams the state *delta each node returns*, not the model's tokens; token-level text comes from the `messages` mode.** ✅
- C) The model doesn't support streaming.
- D) They forgot to set a checkpointer.

*Why:* Stream modes deliver different things; tokens are specifically the `messages` mode, while `updates` carries node-level state changes.

**Q3.** Why does streaming work end-to-end through a chain (e.g. model → parser) without you writing streaming code, and what can `JsonOutputParser` do mid-stream?

- A) Because the parser buffers everything then replays it; it can only emit the final object.
- **B) Because every component implements the Runnable streaming contract so streaming propagates through stages; `JsonOutputParser` can emit *partial* parsed objects as the JSON forms.** ✅
- C) Because streaming only works on the model, and parsers are skipped.
- D) Because the provider streams directly to the browser, bypassing the chain.

*Why:* The shared Runnable interface composes streaming through stages, enabling progressive parsing such as partial-JSON emission for live form-filling.

> **Visualization:** `visualization.html` runs the same agent two ways — a frozen `invoke` spinner vs. a live multi-mode stream — and lets you watch tokens, node updates, and custom progress events arrive in real time.