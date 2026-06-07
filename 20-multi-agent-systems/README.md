# 20 · Multi-Agent Systems — Many Specialists, One Goal

## The plain-English version

One agent with twenty tools and a giant prompt is like one employee asked to be the accountant, the lawyer, the copywriter, and the travel agent all at once. They get confused, pick the wrong tool, and the instructions become an unmanageable wall of text. The natural human solution is the same as the natural software solution: **divide the work among specialists and have someone coordinate them.**

**A multi-agent system is several focused agents working together on a task, each expert at one thing, coordinated so they hand work off to each other.** A "researcher" agent gathers facts, a "writer" agent drafts prose, a "critic" agent checks quality. Each has a tight prompt and a small, relevant tool set — which makes each more reliable than one overloaded generalist. The art is in *how they coordinate*.

This is the capstone of the LangGraph section: it combines everything — agents (13), graphs (14), state (15), edges (16), persistence (17), and handoffs — into systems that tackle complex, real-world work.

## The mental model: why split, and the coordination question

**Why multi-agent?** Three drivers: (1) **reliability** — a focused agent with few tools chooses correctly far more often than one with many; (2) **modularity** — you can build, test, and improve each agent independently; (3) **specialization** — different sub-tasks want different prompts, tools, even different models (a cheap model for routing, a strong one for reasoning).

**The core question is coordination:** who decides what runs next, and how does work pass between agents? In LangGraph, each agent is typically a **node or a subgraph** (module 16), and agents communicate by reading/writing **shared state** (module 15) and via **handoffs** (one agent explicitly transferring control to another, often using `Command(goto=...)`).

## Going deeper: the canonical architectures

**1. Supervisor (orchestrator-worker).** A central "supervisor" agent receives the task and routes it to the right specialist, collects the result, and decides what to do next — possibly looping through several specialists before answering. The workers don't talk to each other; everything goes through the supervisor. Predictable, easy to reason about, easy to add new workers. The most common production pattern.

```
              ┌─────────────┐
              │ supervisor  │  ← decides who works next
              └──────┬──────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │researcher│ │ writer  │  │ coder   │
   └─────────┘  └─────────┘  └─────────┘
```

**2. Swarm (network / peer-to-peer).** Agents hand off directly to one another based on their own judgment — the researcher decides "this needs the coder" and transfers control itself, no central boss. More flexible and emergent, but harder to predict and debug. Good when the right next step is best judged by the agent currently working.

**3. Hierarchical / teams.** Supervisors of supervisors: a top orchestrator manages team-leads, each of whom manages their own workers (each a subgraph). Scales to very complex systems by composing the supervisor pattern at multiple levels.

**4. Sequential pipeline.** Agents run in a fixed order — research → write → edit — each consuming the previous one's output. When the workflow is known, this is simple and reliable (and shades back toward a plain graph/chain).

## The deepest layer: handoffs, state sharing, and hard trade-offs

**Handoffs are the heart of it.** A handoff transfers control (and usually context) from one agent to another. In LangGraph the clean way is a node/tool returning `Command(goto="other_agent", update={...})` — routing to the next agent *and* passing data in one move. A common idiom is a **handoff tool**: the model "calls" a tool like `transfer_to_writer`, which the runtime turns into a `Command` that switches active agents. So agents decide handoffs the same way they decide any tool call (module 06).

**How much context to share?** A central design decision with real trade-offs:

- **Share full message history** — every agent sees everything. Maximum context, but token cost explodes and irrelevant chatter can confuse specialists.
- **Share only results / a summary** — each agent gets a clean, scoped input. Cheaper and more focused, but a downstream agent might miss nuance.
- **Separate vs. shared state schemas** — subgraph agents can keep private working state and expose only what the parent needs (module 15's input/output schemas). Often the right balance.

There's no universal answer; it depends on whether your sub-tasks need rich shared context or clean isolation.

**Subgraphs make it composable.** Because a compiled graph can be a node (module 16), each agent is a self-contained subgraph with its own nodes, tools, and state — wired together by a parent graph. Persistence (module 17) and streaming (module 19) work *through* subgraphs, so the whole system can be checkpointed, paused for human approval (module 18), and streamed coherently.

**The trade-offs are real — don't over-engineer.** Multi-agent systems add coordination overhead, more model calls (cost and latency), more places to fail, and harder debugging. The guidance mirrors module 13's "agency tax": **start with a single agent; reach for multi-agent only when one agent genuinely buckles under too many tools or too broad a task.** Many "multi-agent" problems are better solved by a single agent with better tools or a simpler router.

**Observability becomes essential.** With several agents handing off, *seeing* the flow — who did what, what was passed, where it went wrong — is the only way to debug. LangSmith tracing (module 21) across the whole system is non-negotiable; this is exactly the kind of complexity it was built to make legible.

**Prebuilt help.** Beyond hand-rolling graphs, the ecosystem offers prebuilt supervisor/swarm implementations and "deep agents" patterns (planning + sub-agents + shared file/scratchpad memory) for long, complex tasks — all built on the same LangGraph foundation.

## Common gotchas

- **Multi-agent when one agent would do.** The most common mistake — pay the coordination tax only when warranted.
- **Sharing too much context** balloons tokens and confuses specialists; sharing too little starves them. Tune deliberately.
- **Ambiguous handoff logic** in swarms → agents ping-pong or loop. Clear roles, clear handoff conditions, and a recursion cap.
- **No central trace** → an undebuggable tangle. Instrument with LangSmith from day one.
- **Forgetting subgraph state contracts** → agents clobber each other's state. Use scoped input/output schemas.

## Key takeaways

- A **multi-agent system** splits work among **focused specialist agents** (each with a tight prompt and few tools) coordinated to hand off work — improving reliability, modularity, and specialization.
- Canonical architectures: **supervisor** (central router — most common), **swarm** (peer-to-peer handoffs), **hierarchical teams**, and **sequential pipelines**; each agent is a **node/subgraph** sharing **state**.
- **Handoffs** transfer control + context, idiomatically via **`Command(goto=...)`** / handoff tools; deciding **how much context to share** is a key trade-off (full history vs. summaries vs. scoped schemas).
- Persistence, HITL, and streaming work **through subgraphs**; but multi-agent adds cost and complexity — **start single-agent**, escalate only when needed, and rely on **tracing** to stay debuggable.

## Check your understanding

**Q1.** A team's single agent has 25 tools and a sprawling prompt, and it frequently calls the wrong tool. Before jumping to a complex peer-to-peer swarm, what does the module recommend considering, and what's the most predictable multi-agent starting point if you do split?

- A) Add more tools so it has better options; then use a swarm.
- **B) First check whether a single agent with fewer/better tools or a router suffices; if splitting is truly warranted, start with a supervisor pattern, which is the most predictable and easiest to extend.** ✅
- C) Immediately build a hierarchical team of teams.
- D) Remove persistence to simplify the system.

*Why:* The module warns against over-engineering (the coordination tax) and names the supervisor as the predictable default when multi-agent is justified; swarms are more emergent and harder to debug.

**Q2.** In LangGraph, how is a handoff between agents idiomatically implemented, and how does the deciding agent "choose" to hand off?

- A) By writing to a global variable that other agents poll.
- **B) A node/handoff-tool returns `Command(goto="other_agent", update={...})` — routing to the next agent and passing context at once; the model triggers it by calling a handoff tool, just like any tool call.** ✅
- C) By raising an exception caught by the supervisor.
- D) Handoffs require restarting the graph with a new thread_id.

*Why:* `Command(goto=..., update=...)` fuses routing and state transfer, and handoff tools let the model decide transfers through the normal tool-calling mechanism.

**Q3.** Which statement about context sharing in multi-agent systems is most accurate?

- A) Always share the full message history with every agent for best results.
- B) Never share any context; agents must be fully isolated.
- **C) It's a trade-off: full history maximizes context but costs tokens and can confuse specialists, while sharing summaries/scoped schemas is cheaper and more focused but may drop nuance — the right choice depends on the task.** ✅
- D) Context sharing is automatic and not a design decision.

*Why:* The module frames context sharing as a deliberate trade-off with no universal answer, balancing richness against cost and focus (often via scoped subgraph schemas).

> **Visualization:** `visualization.html` lets you toggle between supervisor, swarm, and pipeline architectures and watch a task flow between specialist agents — with handoffs, shared state, and a token-cost meter that rises as more context is shared.