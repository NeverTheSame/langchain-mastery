# LangChain Mastery — A Concept-by-Concept Learning Collection

A structured, self-paced curriculum that inventories the core ideas across the **LangChain ecosystem** — the three products documented at [docs.langchain.com](https://docs.langchain.com/):

- **LangChain** — the open-source building blocks for agents (models, messages, prompts, tools, retrieval, composition).
- **LangGraph** — low-level orchestration: durable, stateful graphs with memory and human-in-the-loop.
- **LangSmith** — the production platform: observability/tracing, evaluation, prompt engineering, and deployment.

Every module is a self-contained folder with **two files**:

1. `README.md` — a long, layered explainer. It always opens with a **plain-English, non-technical "why"**, then descends through progressively deeper layers: the mental model, the mechanics, the API surface, advanced patterns, gotchas, and how the piece connects to the rest of the stack.
2. `visualization.html` — a self-contained, styled visual (diagram, animation, or interactive widget) that makes the concept tangible. Open it in any browser; no internet or build step required.

## How to use this collection

Work top to bottom. The numbering is the **recommended learning order**, from the easiest foundations to the most advanced production topics. Each module assumes you've absorbed the ones before it. If you only care about one pillar, the table below shows which product each module belongs to.

## The learning path

### Part I — Foundations (the building blocks of any LLM app)

| # | Module | What you'll understand |
|---|--------|------------------------|
| 01 | [LangChain Overview](01-langchain-overview/) | What the ecosystem is, why it exists, how the three products fit together |
| 02 | [Chat Models](02-chat-models/) | The standard model interface and why a uniform abstraction matters |
| 03 | [Messages](03-messages/) | How conversations are represented: roles, content blocks, tool messages |
| 04 | [Prompt Templates](04-prompt-templates/) | Turning variables into reliable, reusable prompts |
| 05 | [Structured Output](05-structured-output/) | Forcing models to return typed, validated data |
| 06 | [Tools & Tool Calling](06-tools-and-tool-calling/) | Letting models take actions in the world |
| 07 | [Runnables & LCEL](07-runnables-and-lcel/) | Composition, streaming, batching — the "wiring" of LangChain |

### Part II — Retrieval, Knowledge & Memory (giving models context)

| # | Module | What you'll understand |
|---|--------|------------------------|
| 08 | [Document Loaders & Splitters](08-document-loaders-and-splitters/) | Getting raw data in and chunking it well |
| 09 | [Embeddings](09-embeddings/) | Turning meaning into vectors |
| 10 | [Vector Stores](10-vector-stores/) | Storing and searching by semantic similarity |
| 11 | [Retrievers & RAG](11-retrievers-and-rag/) | The full retrieval-augmented generation pattern |
| 12 | [Memory & Chat History](12-memory-and-chat-history/) | Making applications remember across turns |

### Part III — Agents & Orchestration (LangGraph)

| # | Module | What you'll understand |
|---|--------|------------------------|
| 13 | [Agents](13-agents/) | The agent loop: reasoning + acting with tools |
| 14 | [LangGraph Overview](14-langgraph-overview/) | Why graphs, and when to drop below the agent abstraction |
| 15 | [State & Reducers](15-state-and-reducers/) | The shared memory that flows through a graph |
| 16 | [Nodes, Edges & Control Flow](16-nodes-edges-and-control-flow/) | Wiring deterministic and conditional paths |
| 17 | [Persistence & Checkpointers](17-persistence-and-checkpointers/) | Durable execution, threads, time travel |
| 18 | [Human-in-the-Loop](18-human-in-the-loop/) | Pausing for approval, editing, and resuming |
| 19 | [Streaming](19-streaming/) | Emitting tokens, state updates, and events live |
| 20 | [Multi-Agent Systems](20-multi-agent-systems/) | Supervisors, swarms, handoffs, and subgraphs |

### Part IV — Production (LangSmith)

| # | Module | What you'll understand |
|---|--------|------------------------|
| 21 | [Observability & Tracing](21-observability-and-tracing/) | Seeing exactly what your agent did and why |
| 22 | [Evaluation](22-evaluation/) | Measuring quality with datasets and judges |
| 23 | [Prompt Engineering & Hub](23-prompt-engineering-and-hub/) | Versioning, optimizing, and collaborating on prompts |
| 24 | [Deployment & LLM Gateway](24-deployment-and-gateway/) | Shipping long-running agents and governing model calls |

## A note on currency

The LangChain ecosystem moves quickly. These modules teach the **durable concepts and mental models** — the things that stay true across versions — and use current (LangChain v1 / LangGraph) terminology where APIs are named. Always cross-check exact function signatures against [docs.langchain.com](https://docs.langchain.com/), which is the authoritative, live source.

---

*Built as a teaching collection. Read the Markdown for depth; open the HTML for intuition.*
