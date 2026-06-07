# 11 · Retrievers & RAG — Grounding Models in Your Data

## The plain-English version

A language model only knows what it learned during training. It doesn't know your company's internal docs, last week's news, or the contents of the PDF you uploaded five minutes ago. Ask it about those and it will either say "I don't know" or, worse, confidently make something up (hallucinate).

**RAG — Retrieval-Augmented Generation — fixes this with a beautifully simple idea: before the model answers, go *fetch* the relevant information and paste it into the prompt.** Instead of "answer from memory," it's "here are the relevant documents; answer using them." The model becomes an open-book test-taker instead of a closed-book one.

This single pattern is the backbone of most production LLM applications — chatbots over documentation, customer-support assistants, research tools, "chat with your PDF." If you learn one composite skill in this whole collection, make it this one. It ties together everything from modules 08–10.

## The mental model: retrieve, then generate

RAG has two phases.

**Indexing (done once, ahead of time):** load your documents (08) → split into chunks (08) → embed each chunk (09) → store the vectors (10). Now you have a searchable knowledge base.

**Retrieval + generation (at query time):**

```
user question
   │
   ▼
[ retriever ] ── finds the most relevant chunks ──┐
   │                                              │
   ▼                                              ▼
[ prompt: "Answer using this context: {chunks} \n Question: {q}" ]
   │
   ▼
[ model ] ──► grounded answer (often with citations)
```

A **retriever** is the standardized component that does the "fetch relevant chunks" step. Its interface is dead simple: question in, list of `Document`s out.

```python
retriever = vector_store.as_retriever(search_kwargs={"k": 4})
docs = retriever.invoke("How do I reset my password?")   # -> 4 relevant chunks
```

Because a retriever is a **Runnable** (module 07), it drops straight into an LCEL chain:

```python
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt | model | StrOutputParser()
)
rag_chain.invoke("How do I reset my password?")
```

## Going deeper: retrievers are more than vector search

A vector store is *a* retriever, but the `Retriever` interface is broader and more powerful. Important built-in retrievers:

- **`MultiQueryRetriever`** — uses an LLM to rephrase the user's question into several variants, retrieves for each, and unions the results. Catches relevant docs the original phrasing would have missed.
- **`ContextualCompressionRetriever`** — retrieves, then *compresses*: a step (often an LLM or a re-ranker) trims each chunk to only the sentences relevant to the query, so the prompt isn't padded with noise.
- **Re-ranking retrievers** — fetch a larger candidate set with cheap vector search, then re-score them with a more accurate (but slower) **cross-encoder** model, keeping only the true top results. This two-stage "retrieve wide, re-rank narrow" pattern is one of the highest-ROI quality upgrades in RAG.
- **`ParentDocumentRetriever`** — embeds small chunks for precise *matching* but returns the larger *parent* passage for richer *context*. Best of both worlds: precise retrieval, complete context.
- **`SelfQueryRetriever`** — an LLM translates a natural-language query into a structured metadata filter *plus* a semantic query ("red shoes under $50" → filter `color=red, price<50` + search "shoes"). Marries semantic and structured search automatically.
- **`EnsembleRetriever`** — combines several retrievers (e.g. BM25 keyword + vector) and fuses their rankings — the hybrid-search pattern from module 10, as a retriever.

## The deepest layer: why naive RAG underperforms, and how to fix it

A first-pass RAG demo is easy; a *good* RAG system is an engineering discipline. The failure modes and their fixes:

**Retrieval failures (the model never saw the answer).** Causes: bad chunking (08), wrong/weak embeddings (09), `k` too small, or the user's wording not matching the docs. Fixes: better chunking, hybrid search, query expansion (`MultiQueryRetriever`), and re-ranking.

**Context dilution ("lost in the middle").** Stuffing many chunks degrades answers because models attend less to the middle of long contexts. Fixes: retrieve fewer but better chunks, re-rank, compress, and order the strongest chunks at the edges.

**No grounding / no citations.** A good RAG prompt instructs the model to answer *only* from the provided context, to say "I don't know" if the context lacks the answer, and to cite which chunk each claim came from (using the metadata from module 08). This is your main defense against hallucination — and it's mostly prompt engineering.

**Agentic RAG.** The modern evolution: instead of always retrieving once, an *agent* (modules 13+) decides **whether** to retrieve, **what** query to use, and **whether to retrieve again** after seeing initial results. Retrieval becomes a *tool* the agent calls when it judges it needs information, enabling multi-hop questions ("compare the 2023 and 2024 figures") that single-shot RAG can't handle. This is where retrieval and agents converge — and where LangGraph (modules 14+) shines, because the retrieve→reason→retrieve loop is naturally a graph.

**Evaluation is non-negotiable.** Because RAG has many moving parts, you must *measure* it — retrieval metrics (did we fetch the right chunks?) and answer metrics (faithfulness to context, relevance to the question). LangSmith (module 22) is built for exactly this. You cannot tune what you don't measure.

## Common gotchas

- **Treating RAG as plug-and-play.** Quality comes from tuning chunking, embeddings, `k`, re-ranking, and the prompt — not from wiring the pieces together.
- **Over-retrieving.** More chunks ≠ better answers. Precision beats volume.
- **No "I don't know" instruction.** Without it, the model fills gaps with plausible fiction even when the context is empty.
- **Ignoring metadata.** Filtering and citations both depend on the metadata you preserved upstream.
- **Skipping evaluation.** "It looks right on my three test questions" is not a quality bar. Build a dataset and measure.

## Key takeaways

- **RAG = fetch relevant context, then generate** — turning the model into an open-book answerer grounded in *your* data.
- A **retriever** (question → `Document`s) is a Runnable that plugs into LCEL chains and agents; vector search is just one kind.
- Quality comes from advanced retrievers (**multi-query, compression/re-ranking, parent-document, self-query, ensemble/hybrid**) and a disciplined **grounding prompt** with citations and "I don't know."
- The frontier is **agentic RAG**, where an agent decides when and how to retrieve — naturally expressed as a LangGraph loop — and where **evaluation** (LangSmith) is essential.

> **Visualization:** `visualization.html` is an end-to-end RAG animation — type a question and watch indexing, retrieval, re-ranking, prompt assembly, and a grounded, cited answer come together.

---

## Check your understanding

**Q1.** A user asks, "Compare our 2023 and 2024 churn and explain the change." Single-shot RAG retrieves once and answers poorly. What approach is designed for this, and what enables it?

- A) Increase `chunk_overlap` until both years fit one chunk.
- **B) Agentic RAG — an agent decides to retrieve multiple times (one query per year, then synthesize), naturally expressed as a LangGraph retrieve→reason→retrieve loop.** ✅
- C) Switch to a larger embedding model.
- D) Use `StrOutputParser` to merge the answers.

*Why:* Multi-hop questions need iterative, agent-driven retrieval where the agent chooses whether/what/when to retrieve — a loop that fits LangGraph.

**Q2.** Which description of the "retrieve wide, re-rank narrow" pattern is correct?

- A) Retrieve a few chunks, then ask the model to invent more.
- **B) Fetch a large candidate set cheaply with vector search, then re-score with a slower, more accurate cross-encoder and keep only the true top results.** ✅
- C) Retrieve from two vector stores and average their vectors.
- D) Retrieve, then randomly drop half the chunks to save tokens.

*Why:* Two-stage retrieval uses cheap recall first and expensive precision second — one of the highest-ROI RAG upgrades.

**Q3.** Your RAG system confidently answers even when the retrieved context contains nothing relevant. What is the primary defense the module recommends?

- A) Lower the temperature to 0.
- B) Increase `k` so something relevant is always included.
- **C) A grounding prompt instructing the model to answer only from the provided context, to say "I don't know" when it's absent, and to cite sources.** ✅
- D) Remove metadata so the model can't be misled.

*Why:* Hallucination on empty context is mainly a prompt-engineering problem; explicit grounding + "I don't know" + citations is the main mitigation.
