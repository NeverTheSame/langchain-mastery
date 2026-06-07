# 03 · Messages — How Conversations Are Represented

## The plain-English version

When you chat with an AI, it feels like a back-and-forth conversation. Under the hood, that conversation is just a **list**. Each entry says *who* spoke and *what* they said. "Messages" is LangChain's standard way of writing down that list so every part of your app — and every model provider — agrees on what a conversation looks like.

Why bother formalizing something so simple? Because the *roles* matter enormously. A model treats text differently depending on whether it came from the **system** (the instructions that set the rules), the **human** (the user), or the **AI** (its own past replies). Get the roles wrong and the model misbehaves. Messages make the roles explicit and tamper-proof.

## The mental model: a typed list of turns

A conversation is a Python list of message objects. The main types:

- **`SystemMessage`** — the standing instructions. "You are a helpful tutor. Always answer in French." Sets behavior for the whole conversation. Usually first, usually one.
- **`HumanMessage`** — input from the user.
- **`AIMessage`** — output from the model. May contain plain text *and/or* `tool_calls` (requests to run tools).
- **`ToolMessage`** — the *result* of running a tool, fed back to the model so it can continue. Carries a `tool_call_id` linking it to the request.

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

conversation = [
    SystemMessage("You are a concise assistant."),
    HumanMessage("What's the capital of France?"),
    AIMessage("Paris."),
    HumanMessage("And its population?"),
]
model.invoke(conversation)
```

The model only ever sees this list. Memory, retrieval, and prompt templates are all ultimately just *ways of building the right list of messages* before you call the model.

## Going deeper: content blocks and multimodality

Early on, a message's `content` was just a string. Modern models handle images, audio, files, PDFs, and "thinking" — so `content` can also be a **list of content blocks**, each a typed dict.

```python
HumanMessage(content=[
    {"type": "text", "text": "What's in this image?"},
    {"type": "image", "source_type": "url", "url": "https://.../cat.png"},
])
```

This is how multimodality works uniformly: a text block, an image block, an audio block — all live inside one message. The same applies to **output**: reasoning models return separate `reasoning` content blocks alongside `text` blocks, so you can display or suppress the chain of thought without parsing provider-specific fields. LangChain's standardized content-block format means you read these the same way no matter the provider.

## The deepest layer: tool calls, the loop, and message management

**The tool-calling round-trip lives entirely in messages.** This is the single most important thing to internalize, because it's the heartbeat of every agent:

1. You send `[SystemMessage, HumanMessage]`.
2. The model returns an `AIMessage` whose `.tool_calls` is `[{name: "get_weather", args: {city: "Paris"}, id: "call_1"}]` — it's *asking* you to run a tool.
3. Your code runs the tool and appends a `ToolMessage(content="18°C", tool_call_id="call_1")`.
4. You send the whole list back. Now the model sees the result and produces a final `AIMessage` with the answer.

That request → result → continue cycle, expressed purely as appended messages, *is* the agent loop (module 13). Agents and LangGraph are largely machinery for managing this growing message list.

**Other practical pieces:**

- **`AIMessageChunk`** — what you get while streaming. Chunks support `+` so you can accumulate them into a full `AIMessage`.
- **`.text`** — a convenience accessor that pulls just the text out, even when content is a list of blocks.
- **Trimming and filtering.** Conversations grow past the context window. Helpers like `trim_messages` (keep the last N tokens, but always preserve the system message and keep tool call/result pairs intact) and `filter_messages` let you manage history without corrupting it.
- **`RemoveMessage`** — a special instruction used with LangGraph state to *delete* messages from the running history (e.g. to summarize and compact).
- **Merging consecutive messages.** Some providers dislike two human messages in a row; `merge_message_runs` fixes that.

## Common gotchas

- **Orphaned tool calls break the model.** If an `AIMessage` has `tool_calls`, the next turn *must* include a matching `ToolMessage` for each `tool_call_id`. Trimming history naively can sever this pair and cause provider errors — use the message-aware trimmers.
- **System message placement.** Most providers expect the system message first; some (e.g. certain Anthropic configs) handle it specially. LangChain normalizes this, but don't scatter multiple system messages randomly.
- **String vs. blocks.** Code that assumes `message.content` is always a string will break on multimodal/reasoning outputs. Prefer `.text` when you only want text.

## Key takeaways

- A conversation is a **list of typed messages**; roles (`System`/`Human`/`AI`/`Tool`) carry meaning the model relies on.
- `content` is a string **or** a list of **content blocks**, which is how images, audio, files, and reasoning are represented uniformly.
- The **tool-call round-trip** (`AIMessage.tool_calls` → run tool → `ToolMessage` → call again) is the foundation of agents.
- Managing the message list (trim, filter, merge, remove) is core to keeping long conversations within the context window without corrupting tool pairs.

> **Visualization:** `visualization.html` lets you step through a tool-calling conversation one message at a time, watching the list grow and the roles light up.

---

## Check your understanding

**Q1.** While trimming a long conversation to fit the context window, your naive trimmer drops an `AIMessage` that contained `tool_calls` but keeps the following `ToolMessage`. What happens and why?

- A) Nothing — `ToolMessage`s are self-contained.
- B) The model summarizes the missing message automatically.
- **C) The provider likely errors, because a `ToolMessage` references a `tool_call_id` whose originating tool call no longer exists in the list.** ✅
- D) The `ToolMessage` is silently promoted to a `HumanMessage`.

*Why:* Tool calls and their results form a linked pair via `tool_call_id`. Severing that pair corrupts the conversation; message-aware trimmers exist precisely to avoid this.

**Q2.** A multimodal request includes an image. How is that represented, and what code assumption will break?

- A) The image is sent as a separate `SystemMessage`; code assuming one system message breaks.
- **B) `content` becomes a list of typed content blocks (text + image); code assuming `message.content` is always a string breaks.** ✅
- C) The image is base64-encoded into the `tool_call_id`; code parsing IDs breaks.
- D) Images require a different model class, so `init_chat_model` breaks.

*Why:* Multimodality uses content blocks inside one message, so `.content` may be a list. Prefer `.text` when you only need text.

**Q3.** Why is the request→result→continue cycle (AIMessage with `tool_calls` → run tool → `ToolMessage` → call again) described as "the heartbeat of every agent"?

- A) Because it's the only way to set a system prompt.
- B) Because it replaces the need for embeddings.
- **C) Because agents are largely machinery for managing this growing, appended message list — the loop expressed purely through messages *is* the agent loop.** ✅
- D) Because it guarantees deterministic output.

*Why:* The agent loop is exactly this message round-trip repeated; agents/LangGraph mostly orchestrate the growing message list.
