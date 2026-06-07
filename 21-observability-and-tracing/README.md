# 21 · Observability & Tracing — Seeing What Your Agent Did

## The plain-English version

When a normal program misbehaves, you read the logs or step through a debugger. But an AI agent is different: its "logic" is a chain of model decisions, tool calls, and retrieved documents, much of it non-deterministic. When it gives a wrong answer, there's no stack trace pointing at line 42. You need to *see the whole thought process* — what prompt the model actually got, what it decided, which tool it called with which arguments, what came back, and how that led to the final answer.

**Observability is the ability to see exactly what your AI application did, step by step. A trace is the detailed recording of one run** — every model call, tool invocation, retrieval, and decision, with inputs, outputs, timing, and token counts. LangSmith is the platform that captures and visualizes these traces. Without it, debugging an agent is guesswork; with it, failures become legible.

This is the first of the production modules. Everything you've built (foundations, RAG, agents) becomes operable once you can *see* it running.

## The mental model: runs and traces as a tree

A single user request produces a **trace** — a tree of **runs**. Each run is one unit of work:

```
Trace: "What's the weather in Paris?"          (root run)
├─ Agent                                        (chain run)
│  ├─ ChatModel call                            (llm run) — saw prompt X, returned tool_call
│  ├─ Tool: get_weather(city="Paris")           (tool run) — returned "18°C"
│  └─ ChatModel call                            (llm run) — produced final answer
```

For each run you can inspect: exact inputs and outputs, latency, token usage and cost, errors, and metadata/tags. The nested tree mirrors your application's structure (which is why the Runnable interface and LangGraph nodes map so cleanly onto traces).

## Going deeper: how tracing gets captured (almost for free)

The headline feature is how *little* you do to get this. Because every component implements the shared Runnable interface (module 07) and LangGraph nodes are instrumented, LangSmith can capture traces automatically. Typically you just set environment variables:

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=...
export LANGSMITH_PROJECT=my-app
```

…and your existing LangChain/LangGraph code is traced with **no code changes**. For non-LangChain code, a `@traceable` decorator (or the OpenTelemetry integration) instruments any function. LangSmith is **framework-agnostic** — you can trace a plain OpenAI-SDK app too — but it's effectively zero-effort if you're already on LangChain.

Traces are organized into **projects** (e.g. one per app or environment), so you can separate dev from prod and keep runs grouped.

## The deepest layer: what observability unlocks beyond debugging

**1. Debugging the non-deterministic.** The core use: when an answer is wrong, open the trace and find the exact step that went sideways — a malformed prompt, a tool called with bad arguments, an empty retrieval, a model that ignored an instruction. You see the *actual* prompt after all templating and history assembly, which is usually where the bug hides.

**2. Latency and cost analysis.** Each run records duration and tokens, so you can find the slow step (often a serial chain of model calls or a fat retrieval) and the expensive step. Aggregate metrics across many traces reveal p50/p99 latency and cost-per-request trends — essential for optimization and budgeting.

**3. Production monitoring.** Beyond single traces, LangSmith aggregates **trends**: error rates, latency, token spend, feedback scores over time. You set up dashboards and alerts so you learn about a regression from a chart, not an angry customer.

**4. Capturing real-world data for evaluation.** Production traces are gold: the real inputs your users send. You can turn interesting or failing traces directly into **datasets** for evaluation (module 22) — closing the loop between "what happened in prod" and "what we test against." This is the heart of the LangSmith philosophy: use live production data for continuous testing and improvement.

**5. Feedback and annotation.** You can attach **feedback** to runs — thumbs up/down from users, scores from automated evaluators, or human annotations in **annotation queues** where reviewers label outputs. This feedback both monitors quality and builds labeled data for improvement.

**6. Human review and collaboration.** Traces are shareable. A teammate (even non-engineer) can open a trace, see exactly what the agent did, and comment. For multi-agent systems (module 20), this cross-system visibility is the only practical way to debug handoffs.

**The deeper point:** observability isn't a debugging convenience bolted on at the end — in LLM apps it's *foundational infrastructure*. Because behavior is probabilistic and emergent, you cannot reliably improve what you cannot see. Teams that ship serious agents instrument tracing from day one.

## Common gotchas

- **Flying blind in production.** Running agents without tracing makes failures nearly impossible to diagnose. Turn it on first, not after the incident.
- **Logging secrets into traces.** Inputs/outputs may contain PII or credentials. Use LangSmith's data-masking/redaction and be mindful of what enters traces.
- **One giant project for everything.** Mixing dev and prod runs muddies metrics. Separate by project/environment.
- **Tracing but never looking.** Capturing traces you don't review, dashboard, or turn into datasets wastes the signal. The value is in *using* them.
- **Assuming the bug is the model.** Traces frequently reveal the real culprit is the *assembled prompt*, an empty retrieval, or a bad tool argument — not the model itself.

## Key takeaways

- **Observability** = seeing exactly what your app did; a **trace** is the step-by-step recording of one run, structured as a **tree of runs** (chain → llm → tool → …) with inputs, outputs, latency, tokens, and errors.
- Tracing is **near-zero-effort** for LangChain/LangGraph (often just env vars) thanks to the shared Runnable interface, and **framework-agnostic** via `@traceable`/OpenTelemetry for other code.
- It unlocks far more than debugging: **latency/cost analysis, production monitoring, dataset capture, feedback/annotation, and team collaboration.**
- In probabilistic LLM systems, observability is **foundational infrastructure** — you can't improve what you can't see, so instrument from day one and actually *use* the traces.

## Check your understanding

**Q1.** An agent returns a wrong answer in production. Opening the trace, the team should look first for which kind of root cause that a normal stack trace would never reveal?

- A) A syntax error on line 42.
- **B) The *actual* assembled prompt the model received (after templating + history), an empty/irrelevant retrieval, or a tool called with bad arguments — the real culprit is often not the model itself.** ✅
- C) A compiler optimization bug.
- D) The thread_id being a string.

*Why:* LLM failures are usually in the assembled inputs (prompt, retrieval, tool args), which traces expose precisely and a conventional stack trace cannot.

**Q2.** Why can LangSmith trace an existing LangChain/LangGraph app with essentially no code changes, yet still trace a plain OpenAI-SDK app?

- A) It only works with LangChain; the OpenAI claim is false.
- **B) The shared Runnable interface and instrumented LangGraph nodes let it auto-capture traces (often via env vars); for non-LangChain code, a `@traceable` decorator / OpenTelemetry makes it framework-agnostic.** ✅
- C) It rewrites your bytecode at import time.
- D) Providers push traces to LangSmith automatically.

*Why:* Auto-tracing rides on the Runnable interface for LangChain code, while `@traceable`/OTel instruments arbitrary functions — so it's both zero-effort and framework-agnostic.

**Q3.** Which statement best captures the LangSmith philosophy connecting observability to improvement?

- A) Traces are only for post-incident forensics and should be deleted after a fix.
- B) Observability replaces the need for evaluation.
- **C) Production traces are real user data that can be turned into datasets and feedback, closing the loop between what happened in prod and what you continuously test and improve.** ✅
- D) Monitoring should be added only once the app is already perfect.

*Why:* The platform's core idea is using live production data for continuous testing/improvement — traces feed datasets (module 22) and feedback, not just one-off debugging.

> **Visualization:** `visualization.html` is an interactive trace explorer — expand a run tree, click any step to inspect its inputs/outputs/latency/tokens, and spot the step where a failing run went wrong.