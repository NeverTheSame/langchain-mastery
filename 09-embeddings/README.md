# 09 · Embeddings — Turning Meaning into Numbers

## The plain-English version

Computers can't compare *meanings* — they compare numbers. So how do you build a search that understands "How do I reset my password?" matches a document titled "Account recovery steps," even though they share almost no words?

The trick is **embeddings**: a way to convert any piece of text into a list of numbers (a *vector*) such that **texts with similar meaning end up with similar numbers**. "Cat" and "kitten" land close together; "cat" and "tax law" land far apart. Once meaning is expressed as coordinates, "find related text" becomes "find nearby points" — a math problem computers are extremely good at.

Embeddings are the quiet engine behind semantic search, recommendation, clustering, and — crucially — RAG (module 11). If you understand embeddings, the rest of retrieval clicks into place.

## The mental model: meaning as a location in space

An embedding model reads text and outputs a fixed-length vector — say 1,536 numbers. Think of each text as a point in a high-dimensional space. The *direction* and *position* of that point encode meaning. The whole point of training these models is that **distance in this space corresponds to difference in meaning.**

```python
from langchain.embeddings import init_embeddings

emb = init_embeddings("openai:text-embedding-3-small")
vec  = emb.embed_query("How do I reset my password?")   # -> [0.013, -0.21, ...] length 1536
docs = emb.embed_documents(["Account recovery steps", "Pizza recipes"])
```

To compare two texts, you embed both and measure how close their vectors are. The closer, the more semantically related.

## Going deeper: the two methods and why they differ

The `Embeddings` interface has exactly two methods, and the split matters:

- **`embed_query(text)`** — embeds a *single* search query.
- **`embed_documents(list)`** — embeds a *batch* of documents to be stored.

Why two methods for "the same" operation? Some embedding models are **asymmetric** — they're trained to embed questions and answers slightly differently so a short question lands near the long document that answers it. Even when symmetric, separating them lets LangChain batch document embedding efficiently (you embed thousands of docs once at indexing time, but queries one at a time at search time).

**Measuring similarity.** The standard metric is **cosine similarity** — it measures the *angle* between two vectors, ignoring their length, so it captures "pointing in the same semantic direction." Values run from -1 (opposite) to 1 (identical meaning). Other metrics (dot product, Euclidean distance) are used too; the right choice depends on how the model was trained, and vector stores let you pick.

## The deepest layer: dimensions, models, and practical economics

**Dimensionality is a trade-off.** More dimensions can capture finer distinctions but cost more to store and compare. Modern models (e.g. Matryoshka-style embeddings) let you *truncate* the vector to fewer dimensions with graceful quality loss — so you can dial the storage/accuracy trade-off. Typical sizes range from a few hundred to a few thousand dimensions.

**Choosing a model — the levers:**

- **Quality** on your domain (general web text vs. code vs. legal vs. medical). Domain-specific models can beat bigger general ones.
- **Cost & latency** — API models (OpenAI, Cohere, Voyage) are easy but metered; local/open models (e.g. via Hugging Face or `sentence-transformers`) are free to run and keep data private.
- **Context length** — how much text fits in one embedding. Long-document models embed bigger chunks.
- **Multilingual** support if you serve multiple languages.

**The golden rule: never mix embedding models in one index.** Vectors from model A and model B live in *different, incompatible spaces*. If you embed your documents with one model and your queries with another, similarity is meaningless. If you change embedding models, you must **re-embed your entire corpus**. This is a real operational cost — choose deliberately.

**Caching saves real money.** Embedding the same text repeatedly is wasteful. `CacheBackedEmbeddings` stores computed vectors keyed by text, so re-indexing unchanged documents is free. Important at scale.

**Embeddings power more than search.** The same vectors enable **clustering** (group similar docs), **classification** (label by nearest examples), **deduplication** (find near-identical content), **recommendations** (find similar items), and **outlier detection**. Retrieval is the headline use, not the only one.

**Where embeddings sit in RAG.** Pipeline: load → split (module 08) → **embed each chunk** → store vectors in a vector store (module 10) → at query time, embed the question and find nearest chunks (module 11). Embeddings are the translation layer that makes semantic search possible.

## Common gotchas

- **Mismatched models = broken retrieval.** Query and corpus must use the *same* embedding model. The most common silent RAG bug.
- **Bigger isn't always better.** A 3,072-dim model can underperform a smaller domain-tuned one on your data — and costs more. Evaluate on *your* queries.
- **Embeddings have no notion of recency or authority.** "Similar" ≠ "correct" or "current." Combine with metadata filters and re-ranking for production quality.
- **Chunk quality caps embedding quality.** A chunk that mixes three topics produces a muddy "average" vector that matches nothing well (ties back to module 08).
- **Re-embedding cost is easy to forget** when planning a model upgrade. Budget for it.

## Key takeaways

- An **embedding** maps text to a vector so that **semantic similarity becomes geometric closeness**; **cosine similarity** is the usual measure.
- The interface has **`embed_query`** (one query) and **`embed_documents`** (batch) — split because models can be asymmetric and for indexing efficiency.
- Choose a model on **quality, cost, context length, and language**; **never mix models** in one index, and **cache** to avoid re-paying for unchanged text.
- Embeddings are the foundation of **semantic search and RAG**, plus clustering, classification, dedup, and recommendations.

> **Visualization:** `visualization.html` is an interactive 2-D "meaning map" — type a query and watch it land near semantically related documents, with live similarity scores.

---

## Check your understanding

**Q1.** After a model upgrade, your team re-embeds only *new* documents added since the switch, leaving old ones embedded with the previous model. Retrieval quality collapses. Why?

- A) The new model produces longer vectors that overflow the store.
- **B) Vectors from different embedding models live in incompatible spaces; mixing them makes similarity meaningless — the entire corpus must be re-embedded with one model.** ✅
- C) The store's HNSW index can't hold two models.
- D) Cosine similarity only works within a single day of indexing.

*Why:* Query and corpus must share one embedding model; changing models requires re-embedding everything, which is a real, easily-forgotten operational cost.

**Q2.** Why does the `Embeddings` interface separate `embed_query` from `embed_documents` instead of one method?

- A) Purely for naming convenience; they're identical.
- B) Queries are encrypted but documents are not.
- **C) Some models are asymmetric (trained to embed questions and answers differently), and the split also enables efficient batch embedding of documents at index time.** ✅
- D) `embed_query` returns text while `embed_documents` returns vectors.

*Why:* Asymmetric models place a short question near its long answer; separating the methods supports that and lets you batch-embed corpora efficiently.

**Q3.** A general 3,072-dimension model performs *worse* on your legal-search task than a smaller domain-tuned model, while also costing more. What does this illustrate?

- A) Higher dimensionality always reduces accuracy.
- B) Cosine similarity is broken above 1,000 dimensions.
- **C) "Bigger isn't always better" — domain fit can beat raw size/dimensionality, and you should evaluate models on *your* queries, weighing cost and latency.** ✅
- D) Legal text cannot be embedded.

*Why:* Model choice is a trade-off across quality-on-your-domain, cost, latency, context length, and language — evaluate on your data rather than assuming bigger wins.
