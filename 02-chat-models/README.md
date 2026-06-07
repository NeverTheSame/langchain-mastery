# 02 · Chat Models — The Standard Model Interface

## The plain-English version

A "chat model" is the actual AI brain — GPT, Claude, Gemini, Llama. You hand it a conversation and it produces the next message. That's it. Everything else in LangChain exists to feed this thing good input and do something useful with its output.

So why not just call the provider's SDK directly? Because every provider speaks a slightly different dialect. OpenAI wants your messages shaped one way, Anthropic another, Google a third. Their streaming works differently. Their tool-calling formats differ. If you write your app against OpenAI's exact SDK and later want to try Claude — maybe it's cheaper, maybe it's better at your task — you'd be rewriting plumbing all over your codebase.

**LangChain gives you one universal "remote control" that works with every brand of TV.** You learn one set of buttons (`invoke`, `stream`, `bind_tools`) and they work the same whether the model behind them is from OpenAI, Anthropic, Google, AWS, or a model running on your own laptop. Switching providers becomes a one-line change.

## The mental model

A chat model in LangChain is an object that obeys a single contract called `BaseChatModel`. You give it a list of **messages** (the conversation so far) and it returns an **AIMessage** (the model's reply). Because it's a `Runnable` (module 07), it automatically supports four ways of being called:

- `invoke(messages)` — one call, wait for the full reply.
- `stream(messages)` — get the reply token-by-token as it's generated.
- `batch([...])` — run many inputs in parallel efficiently.
- async versions (`ainvoke`, `astream`, `abatch`) for concurrency.

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-4o-mini", model_provider="openai")
# swap the next line and nothing else changes:
# model = init_chat_model("claude-3-5-sonnet-latest", model_provider="anthropic")

reply = model.invoke("Explain embeddings in one sentence.")
print(reply.content)
```

`init_chat_model` is the modern, provider-agnostic entry point. The two arguments — model name and provider — are the *only* things that change when you switch vendors.

## Going deeper: what the interface standardizes

The genius isn't just "one function name." LangChain normalizes the messy parts so they look identical everywhere:

**Input.** You can pass a plain string, a list of `(role, content)` tuples, or a list of message objects (`SystemMessage`, `HumanMessage`, `AIMessage`). All three are accepted and normalized internally.

**Output.** Always an `AIMessage` with a consistent shape: `.content` (the text, or a list of content blocks for multimodal/reasoning), `.tool_calls` (structured requests to call tools), `.usage_metadata` (token counts), and `.response_metadata` (provider-specific extras like finish reason). You read token usage the same way regardless of provider.

**Capabilities as methods, not rewrites.** Want structured output? `model.with_structured_output(Schema)`. Want tools? `model.bind_tools([...])`. Want to pin parameters like temperature? `model.bind(temperature=0)`. Each returns a *new* configured model; the call site stays the same.

**Configuration parameters** are standardized too: `temperature` (randomness), `max_tokens` (length cap), `timeout`, `max_retries`, and `model_kwargs` for provider-specific knobs.

## The deepest layer: standard params, rate limits, and caching

**Standard vs. provider-specific parameters.** LangChain defines a set of *standard* parameters (`temperature`, `max_tokens`, `stop`, `timeout`, `max_retries`) that map onto each provider's equivalent. Anything genuinely unique to a provider goes through `model_kwargs`, keeping the common path portable while still giving you an escape hatch.

**Rate limiting.** Production apps hit provider rate limits. LangChain offers an `InMemoryRateLimiter` you can attach to a model so it self-throttles rather than erroring out under load.

**Caching.** Identical prompts can be cached so you don't pay for or wait on duplicate calls — useful in tests and for deterministic, repeated queries. You can set a global cache or a per-model one.

**Multimodality.** Modern chat models accept more than text — images, audio, files, and PDFs — expressed as *content blocks* inside a message (covered in module 03). The same `invoke` call carries them.

**Reasoning models.** Newer "thinking" models emit a separate reasoning stream. LangChain surfaces this as distinct content blocks so you can show or hide the model's chain of thought without parsing provider-specific fields.

## Common gotchas

- **Temperature 0 is not fully deterministic.** It reduces randomness but providers don't guarantee identical bytes every time. Don't build exact-string assertions on top of it.
- **`init_chat_model` needs the integration package installed** for the provider you name (e.g. `langchain-openai`, `langchain-anthropic`), plus the API key in the environment.
- **Token limits are real.** `.usage_metadata` is your friend for cost control and for catching prompts that are quietly being truncated.
- **Not every model supports every feature.** Tool calling and structured output depend on the underlying model; LangChain exposes the method uniformly but the provider must support it.

## Key takeaways

- A chat model maps **messages in → AIMessage out**, behind the uniform `BaseChatModel` / `Runnable` interface.
- `init_chat_model("name", model_provider="...")` is the portable way to instantiate one; switching vendors is a one-line change.
- Capabilities (tools, structured output, fixed params) are added with `bind_tools`, `with_structured_output`, and `bind` — the call site never changes.
- Standard parameters keep you portable; `model_kwargs` is the escape hatch for provider-specific features.

> **Visualization:** `visualization.html` shows the same request being routed through four different providers behind one interface — and lets you inspect the standardized `AIMessage` that comes back.

---

## Check your understanding

**Q1.** You write `model.with_structured_output(Schema)` and `model.bind_tools([...])`. What is true about the *original* `model` object and the call site?

- A) Both methods mutate `model` in place, so later plain `.invoke` calls also become structured.
- **B) Each returns a new configured Runnable; the original `model` is unchanged and the `invoke` call signature stays the same.** ✅
- C) `bind_tools` changes the provider, so you must re-import the integration package.
- D) They only work after you set `temperature=0`.

*Why:* These methods return new configured models without mutating the original, and the uniform Runnable interface keeps the call site identical — that's the whole point of the abstraction.

**Q2.** A teammate sets `temperature=0` and writes a test asserting the model returns a byte-for-byte identical string every run. Why is this fragile?

- A) Temperature 0 is ignored by `init_chat_model`.
- B) Temperature only affects streaming, not `invoke`.
- **C) Temperature 0 reduces but does not guarantee determinism — providers don't promise identical output, so exact-string assertions can flake.** ✅
- D) Temperature 0 forces the model into JSON mode, changing the output shape.

*Why:* Low temperature lowers randomness but providers make no bit-exactness guarantee; build assertions on semantics, not exact bytes.

**Q3.** Your app must run mostly on a provider's standard knobs but occasionally needs a feature unique to one vendor. What's the idiomatic way to stay portable yet access that feature?

- A) Fork the integration package and hard-code the feature.
- B) Abandon `init_chat_model` and call the raw SDK everywhere.
- C) Put everything, standard and unique, into `model_kwargs`.
- **D) Use the standard parameters for the common path and pass the vendor-specific feature through `model_kwargs` as an escape hatch.** ✅

*Why:* Standard params keep you portable; `model_kwargs` is the deliberate escape hatch for provider-specific extras without coupling your whole app to one vendor.
