# 24 · Deployment & LLM Gateway — Shipping and Governing Agents

## The plain-English version

You've built an agent that works on your laptop. Now real users need to reach it, around the clock, possibly thousands at once, with tasks that run for minutes and conversations that span days. That's a different problem from "it runs in my notebook." You need a server that's always on, scales with load, keeps each conversation's state durably, survives crashes, and exposes an API your app can call. **Deployment is the work of turning your agent into a reliable, hosted service.**

Separately, once many parts of your company are calling language models, a new problem appears: *governance*. Who's spending how much? Are API keys scattered across a dozen codebases? Is sensitive data leaking into prompts? An **LLM Gateway** is a single checkpoint that all model calls pass through, so you can enforce budgets, redact data, and manage credentials centrally.

This is the final module: it's how everything you've learned actually reaches users and stays under control in production.

## The mental model: why agents are hard to deploy

A stateless web API is easy to host. Agents break the usual assumptions in ways that make naive deployment painful:

- **Long-running.** A research agent might run for minutes; a normal HTTP request times out. You need background execution and a way to poll or stream results.
- **Stateful.** Each conversation has durable state (module 17) that must persist across requests and survive restarts — so you need managed persistence (a real database for checkpoints), not in-memory state.
- **Bursty and parallel.** Load spikes; many agents run at once. You need queuing and horizontal scaling.
- **Pausable.** Human-in-the-loop (module 18) means runs can sit paused for hours/days, then resume — the infrastructure must hold and reactivate them.
- **Streaming.** Users expect live progress (module 19), so the server must stream over the network.

The **LangGraph Platform** exists to handle exactly these concerns so you don't rebuild them.

## Going deeper: the LangGraph Platform

LangGraph apps are just Python, so you *can* deploy them anywhere (a container, your own server). But the LangGraph Platform is **purpose-built hosting for stateful, long-running agents**, providing:

- **Managed persistence.** A production-grade checkpointer/store backend (e.g. Postgres) handled for you — durable threads, memory, and time travel without you operating a database.
- **APIs out of the box.** REST endpoints to create threads, run the graph, stream output, list/inspect state, and submit human-in-the-loop resumptions — the server-side surface real UIs are built on.
- **Task queues & horizontal scaling.** Handles bursts, background long-running tasks, retries, and concurrency, scaling workers as load grows.
- **Streaming & double-texting handling.** Built-in support for streaming responses and for the messy realities of chat (e.g. a user sending a second message before the first finishes).
- **Deployment options.** Cloud (fully managed), hybrid, or self-hosted — to meet data-residency and compliance needs.
- **Studio.** A visual IDE to run, debug, and inspect your deployed graph — see the state at each step, edit it, and replay, tightly integrated with LangSmith tracing (module 21).

The "one-click deploy" promise is about taking a compiled graph and getting all of the above without assembling it yourself.

## The deepest layer: the LLM Gateway and production governance

**The LLM Gateway** is a proxy that sits between your applications and the model providers. Every model call routes through it, which centralizes control that's otherwise scattered:

- **Spend limits & cost control.** Enforce per-team/per-app budgets and rate limits; stop runaway costs before they happen, and get unified spend visibility across providers.
- **Credential management.** Provider API keys live in *one* governed place, not copy-pasted across repos. Rotate keys centrally; apps never hold raw provider secrets.
- **Data protection.** Redact or block sensitive data (PII, secrets) from leaving in prompts — a compliance and security safeguard.
- **Provider abstraction & routing.** Route to different providers/models behind one endpoint, add fallbacks, and switch providers without touching app code — the gateway-level cousin of LangChain's `init_chat_model` portability (module 02).

The gateway is the *operational* governance layer; LangSmith tracing/eval is the *quality* layer. Together they're what "running LLMs responsibly at scale" means.

**Production concerns that tie it all together:**

- **Security & compliance.** Enterprise deployments need data privacy, access controls, and certifications (e.g. SOC 2, HIPAA, GDPR). This drives the self-hosted/hybrid options.
- **Observability in prod (module 21).** You don't deploy and walk away — tracing, dashboards, and alerts run continuously so you catch regressions and drift.
- **Online evaluation (module 22).** Score live traffic to keep quality from silently degrading.
- **The full lifecycle.** This collection's arc: build with LangChain (foundations + RAG) → orchestrate with LangGraph (agents, state, HITL, streaming) → operate with LangSmith (observe, evaluate, manage prompts) → deploy and govern (Platform + Gateway). Deployment isn't the end; it feeds back into observation and evaluation, and the loop continues.

## Common gotchas

- **Treating an agent like a stateless API.** Long-running, stateful, pausable workloads need purpose-built infra; naive request/response hosting breaks on timeouts and lost state.
- **In-memory persistence in production.** `InMemorySaver` loses everything on restart (module 17). Use a durable backend — the Platform manages this.
- **Scattered API keys & no budget caps.** Without a gateway, costs and credentials sprawl; one bug can run up a huge bill. Centralize spend limits and keys.
- **No data redaction.** Sensitive data flowing into third-party model calls is a compliance risk; gate it.
- **Deploy-and-forget.** Without continuous tracing and online eval, quality drifts and you find out from users. Operate, don't just ship.

## Key takeaways

- **Deployment** turns an agent into a reliable hosted service; agents are hard to deploy because they're **long-running, stateful, bursty, pausable, and streaming** — assumptions a plain API doesn't meet.
- The **LangGraph Platform** provides purpose-built hosting: **managed persistence, ready-made APIs, task queues/scaling, streaming, deployment options, and Studio** for debugging.
- An **LLM Gateway** centralizes governance for all model calls — **spend limits, credential management, data redaction, and provider routing** — the operational counterpart to LangSmith's quality tooling.
- Production is a **lifecycle**: build (LangChain) → orchestrate (LangGraph) → operate (LangSmith) → **deploy & govern** (Platform + Gateway) → observe/evaluate → repeat.

## Check your understanding

**Q1.** Which property of agents most directly breaks a *standard stateless HTTP API* deployment, and what does the Platform provide to address it?

- A) Agents use too many tokens; the Platform compresses them.
- **B) Agents are long-running and stateful (and can pause for human input for hours/days); the Platform provides background execution, managed durable persistence, and APIs to stream/resume — things a request/response API can't handle.** ✅
- C) Agents require a GPU; the Platform rents one.
- D) Agents can't be written in Python; the Platform transpiles them.

*Why:* Long-running, stateful, pausable workloads exceed normal request timeouts and lose in-memory state; the Platform is purpose-built for exactly these traits.

**Q2.** A company has model API keys copy-pasted across many repos, no per-team budgets, and worries about PII leaking into prompts. Which component addresses all three, and how does it relate to LangChain's `init_chat_model` portability?

- A) The checkpointer; it stores keys in state.
- B) The Prompt Hub; it versions the keys.
- **C) An LLM Gateway — it centralizes credentials, enforces spend limits, and redacts sensitive data; it's the gateway-level cousin of `init_chat_model`'s provider portability, routing/switching providers behind one endpoint.** ✅
- D) Studio; it debugs the keys visually.

*Why:* The gateway is the operational governance layer for credentials, cost, data protection, and provider routing — distinct from per-app code portability but conceptually related.

**Q3.** Why does the module insist deployment is "not the end" of the lifecycle?

- A) Because deployed agents never need changes.
- B) Because the Platform automatically improves the agent.
- **C) Because production feeds back into continuous observability (tracing) and online evaluation, which surface regressions/drift and harvest data to improve the app — the build→orchestrate→operate→deploy loop repeats.** ✅
- D) Because deployment must be redone daily.

*Why:* Shipping starts the operate/observe/evaluate loop; production traces and online evals drive ongoing improvement rather than ending the work.

> **Visualization:** `visualization.html` is a production-architecture diagram — trace a user request from the app, through the LLM Gateway (budget/redaction/key checks), into a scaled LangGraph deployment with durable persistence, and back, with a live "without vs. with" governance toggle.