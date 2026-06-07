# 07 · Runnables & LCEL — The Wiring of LangChain

## The plain-English version

You've now met several building blocks: prompt templates, chat models, output parsers, retrievers. Each does one job. Real applications **chain** them: take the user's question → format it into a prompt → send it to the model → parse the reply. The question is *how do these pieces connect?*

LangChain's answer is elegant: **every building block speaks the same language.** They all implement one shared interface called **Runnable**. Because they're all Runnables, you can snap them together with a pipe `|`, exactly like Unix shell pipes (`cat file | grep x | sort`). This composition style is nicknamed **LCEL** — the LangChain Expression Language.

The plain-English payoff: you describe *what* should happen (`prompt | model | parser`) and you get streaming, parallelism, batching, async, and automatic tracing **for free**, without writing any of that infrastructure yourself.

## The mental model: one interface to rule them all

A **Runnable** is any object with a standard set of methods:

- `invoke(input)` — run once, return output.
- `stream(input)` — run and yield output incrementally.
- `batch(inputs)` — run many inputs in parallel.
- `ainvoke` / `astream` / `abatch` — async versions.

Prompts, models, parsers, retrievers, tools, and *entire chains* all implement this. Because the interface is uniform, the output of one can feed the input of the next:

```python
chain = prompt | model | StrOutputParser()
chain.invoke({"question": "What is RAG?"})     # string out
chain.stream({"question": "What is RAG?"})     # tokens stream out — free!
chain.batch([{...}, {...}, {...}])             # parallelized — free!
```

The `|` operator literally builds a `RunnableSequence`: the left side's output becomes the right side's input. That's the whole trick.

## Going deeper: the composition primitives

LCEL gives you a small vocabulary of Runnables for control flow:

**`RunnableSequence` (the `|` pipe)** — run steps in order, passing output forward.

**`RunnableParallel` (a dict)** — run several Runnables on the *same* input simultaneously and collect results into a dict. This is the canonical RAG move:

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

setup = RunnableParallel(
    context=retriever,                 # fetch docs
    question=RunnablePassthrough(),    # pass the question through unchanged
)
rag_chain = setup | prompt | model | StrOutputParser()
```

Here `context` and `question` are computed in parallel, then both flow into the prompt.

**`RunnablePassthrough`** — forwards input unchanged (optionally adding fields with `.assign(...)`). The glue that keeps the original input available downstream.

**`RunnableLambda`** — wraps any plain Python function so it becomes a Runnable and joins the pipe. Your custom logic, first-class in a chain.

**`RunnableBranch`** — conditional routing: pick a sub-chain based on the input (a simple if/elif/else over Runnables).

**`.with_fallbacks([...])`** — if a Runnable fails (e.g. a model is down), automatically try a backup. Production resilience in one call.

**`.with_retry(...)`** — retry on transient failures with backoff.

**`.with_config(...)` / `RunnableConfig`** — attach run-time configuration: tags, metadata, callbacks, concurrency limits, and `configurable` fields that let you swap parts of a chain at call time.

## The deepest layer: why this design is powerful

**Streaming composes automatically.** When you `stream()` a sequence, LCEL streams through every stage that supports it. Tokens from the model flow through a `StrOutputParser` live. Even `JsonOutputParser` can emit partial objects as they form. You didn't write any streaming code — the interface made it work end to end.

**Optimal parallelism for free.** `RunnableParallel` and `batch` run independent work concurrently. LangChain figures out what can run at once. A RAG chain that fetches context while echoing the question does so in parallel without you managing threads.

**Async without rewrites.** Every Runnable has async variants, so the same chain runs efficiently in an async web server. You write the chain once.

**Universal observability.** Because every step is a Runnable, LangSmith (module 21) can trace each one automatically — inputs, outputs, latency, token counts — usually with zero code changes. The shared interface is *what makes tracing possible*.

**Configurable, swappable internals.** With `configurable_fields`/`configurable_alternatives`, you can expose knobs (model name, temperature, which retriever) that callers set at invoke time — so one chain definition serves many configurations.

## LCEL vs. LangGraph — when to use which

This is a crucial judgment call:

- **LCEL** is perfect for **linear or lightly-branching data pipelines** — "do A, then B, then C." It's declarative and concise. RAG chains, extraction pipelines, and prompt→model→parser flows are textbook LCEL.
- **LangGraph** is for **cyclical, stateful, long-running** flows — agents that loop, branch dynamically, pause for humans, and need persistence. The moment you need a *loop* or durable state, you've outgrown LCEL and should move to LangGraph (modules 14+).

A clean heuristic: **no cycles, no durable state → LCEL. Cycles or state or human pauses → LangGraph.** They share the same Runnable foundation, so a LCEL chain can be a node *inside* a LangGraph graph.

## Common gotchas

- **Type mismatches between stages.** Each step's output must be a valid input to the next. A model outputs an `AIMessage`; if the next step expects a string, add `StrOutputParser()`. Reading the trace in LangSmith makes mismatches obvious.
- **`RunnableParallel` keys define the output dict shape** — downstream steps must reference those exact keys.
- **Forgetting `RunnablePassthrough`** drops the original input, so later stages can't see it.
- **Over-engineering with LCEL.** If your chain is sprouting branches, retries, and loops, stop forcing it into pipes — switch to LangGraph.

## Key takeaways

- **Every** LangChain building block is a **Runnable** with the same `invoke`/`stream`/`batch`/async interface — that uniformity is the foundation of the whole library.
- **LCEL** composes Runnables with `|` (sequence) and dicts (parallel), giving you streaming, batching, async, and tracing **for free**.
- Core primitives: `RunnableSequence`, `RunnableParallel`, `RunnablePassthrough`, `RunnableLambda`, `RunnableBranch`, plus `.with_fallbacks`/`.with_retry`/`.with_config`.
- Use **LCEL for linear pipelines, LangGraph for loops and durable state** — they're built on the same interface and interoperate.

> **Visualization:** `visualization.html` lets you build a chain by toggling stages and watch data — and a live token stream — flow through the pipe, including a parallel RAG-style fan-out.

---

## Check your understanding

**Q1.** A developer's flow is sprouting cycles, retries that re-enter earlier steps, and a pause for human approval that resumes hours later. They keep forcing it into longer `|` pipes. What's the correct call?

- A) Keep using LCEL but add more `RunnableLambda`s.
- B) Use `RunnableBranch` to create the loop.
- **C) Move to LangGraph — cycles, durable state, and human pauses are exactly where LCEL ends and LangGraph begins (they share the Runnable foundation, so it's not a rewrite from scratch).** ✅
- D) Replace the model with a reasoning model so loops aren't needed.

*Why:* The heuristic is "no cycles/no durable state → LCEL; cycles or state or human pauses → LangGraph." `RunnableBranch` does conditional routing, not durable loops.

**Q2.** In a RAG setup you write `{"context": retriever, "question": RunnablePassthrough()} | prompt | ...`. What does the dict accomplish?

- A) It runs the retriever and passthrough sequentially, retriever first.
- **B) It's a `RunnableParallel`: retriever and passthrough run on the same input simultaneously, producing a dict consumed by the prompt's `{context}` and `{question}` slots.** ✅
- C) It caches the retriever output for later batches.
- D) It converts the question into an embedding.

*Why:* A dict of Runnables is `RunnableParallel`, executing branches concurrently on the same input and collecting results by key.

**Q3.** Why does `chain.stream(...)` yield tokens live even though *you* never wrote streaming code?

- A) Because `StrOutputParser` buffers the full output then replays it quickly.
- B) Because streaming only works on single-stage chains.
- **C) Because every stage implements the Runnable interface, so `stream` propagates through each stage that supports streaming — end-to-end streaming emerges from the shared interface.** ✅
- D) Because the provider pushes tokens directly to your terminal.

*Why:* The uniform Runnable streaming contract composes; tokens flow through compatible stages (even partial-JSON parsers) without bespoke code.
