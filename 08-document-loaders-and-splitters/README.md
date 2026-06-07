# 08 · Document Loaders & Text Splitters — Getting Data In, Cut Well

## The plain-English version

You want your AI to answer questions about *your* stuff — your PDFs, your Notion pages, your company wiki, a folder of contracts. Before the model can use any of it, two boring-but-critical things must happen:

1. **Load** the data: read it out of whatever format it lives in (PDF, Word, HTML, a database, a website) and turn it into clean text the system can work with.
2. **Split** it: chop that text into bite-sized chunks.

Why chop it up? Two reasons. First, models have a limited context window — you can't paste a 300-page manual into a prompt. Second, and more importantly, when the user asks a question you only want to retrieve the *relevant* paragraph, not the whole book. Small, well-formed chunks make retrieval precise. **Loaders get the data in; splitters make it usable.** Together they're the unglamorous foundation of every RAG system (module 11) — and the place where most RAG quality problems are actually born.

## The mental model: Documents in, chunked Documents out

The unit of currency here is the **`Document`** — a simple object with two fields:

- `page_content` — the text.
- `metadata` — a dict of everything *about* the text: source filename, page number, URL, author, timestamps, tags.

A **document loader** reads a source and produces a list of `Document`s. A **text splitter** takes large `Document`s and returns more, smaller `Document`s — carrying the metadata along so you always know where each chunk came from.

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

docs = PyPDFLoader("manual.pdf").load()          # one Document per page
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
chunks = splitter.split_documents(docs)          # many small Documents
```

## Going deeper: loaders

There are **hundreds** of loaders, because data lives everywhere. Categories you'll meet:

- **Files:** PDF, Word, PowerPoint, CSV, JSON, Markdown, plain text.
- **Web:** a single URL, a recursive crawl, sitemaps.
- **SaaS/apps:** Notion, Google Drive, Slack, Confluence, GitHub, Gmail.
- **Databases:** SQL, MongoDB, etc.

Two loading styles matter for big sources:

- `.load()` — read everything into memory at once. Fine for small sources.
- `.lazy_load()` — yields `Document`s one at a time, so you can process huge corpora without exhausting RAM.

**Parsing quality is everything.** A PDF loader that mangles tables, drops headers, or scrambles multi-column layouts will poison everything downstream — garbage in, garbage out. For messy real-world documents, specialized parsers (OCR for scans, layout-aware PDF parsers) often matter more than any later tuning. Many loaders also let you control whether a PDF becomes one Document per page or one per document.

## The deepest layer: splitting strategy is a quality lever

Naïvely cutting text every N characters is a recipe for bad retrieval — you'll slice sentences in half and separate a claim from its context. Better strategies:

**`RecursiveCharacterTextSplitter` (the default workhorse).** It tries to split on a *hierarchy* of separators — first paragraphs (`\n\n`), then lines (`\n`), then sentences, then words — only descending to a finer level when a chunk is still too big. This keeps semantically related text together far better than a blind character cut. Sensible defaults for prose.

**Structure-aware splitters.** Markdown, HTML, and code have structure worth respecting. `MarkdownHeaderTextSplitter` splits on headers and stores the header hierarchy in metadata, so a chunk "knows" which section it belongs to. Language-aware code splitters break on function/class boundaries rather than mid-function.

**Token-based splitting.** Because models think in *tokens*, not characters, you can split by token count to align chunks precisely with context limits and cost.

**Semantic splitting.** A more advanced approach embeds sentences and cuts where meaning *shifts* (where adjacent sentences become dissimilar), producing topically-coherent chunks. More expensive, sometimes worth it.

**The two key dials:**

- **`chunk_size`** — how big each chunk is. Too large → retrieval returns lots of irrelevant text and dilutes the signal; too small → chunks lose context and you fragment ideas. Typical starting range: 500–1500 characters/tokens, tuned to your content.
- **`chunk_overlap`** — how much consecutive chunks share. A small overlap (e.g. 10–20%) prevents a key sentence from being orphaned exactly at a boundary, preserving continuity across the cut.

**Metadata is a first-class retrieval tool.** Good chunks carry rich metadata (source, page, section, date). Later you can *filter* retrieval by metadata ("only search the 2024 contracts") and you can *cite* sources in answers ("per page 14 of manual.pdf"). Invest in metadata at load/split time; you can't easily add it back later.

## Common gotchas

- **Chunking is where RAG silently fails.** If answers are vague or miss obvious facts, suspect chunk size/overlap and parsing quality *before* blaming the model or the embeddings.
- **Bad PDF parsing is invisible until it isn't.** Always eyeball a few parsed chunks. Tables and multi-column PDFs are notorious.
- **One-size chunking across mixed content** (code + prose + tables) underperforms. Match the splitter to the content type.
- **Lost metadata = no citations and no filtering.** Preserve source/page through every transformation.
- **Overlap too large** wastes storage and returns near-duplicate chunks; too small orphans context. Tune it.

## Key takeaways

- A **`Document`** is `page_content` + `metadata`; **loaders** produce them from any source, **splitters** cut them into retrieval-sized pieces.
- Use **`lazy_load()`** for large corpora; remember that **parsing quality** caps everything downstream.
- Prefer **`RecursiveCharacterTextSplitter`** for prose and **structure-aware** splitters for Markdown/HTML/code; consider token-based or semantic splitting when it pays off.
- Tune **`chunk_size`** and **`chunk_overlap`** deliberately — they're primary RAG quality dials — and preserve rich **metadata** for filtering and citations.

> **Visualization:** `visualization.html` lets you drag chunk-size and overlap sliders over a sample document and watch the chunks (and their overlaps) form in real time.

---

## Check your understanding

**Q1.** Answers from your RAG bot are vague and miss facts that are clearly in the source PDFs. Per the module, what should you suspect *first*?

- A) The chat model is too small — upgrade it.
- B) The vector store's ANN recall is misconfigured.
- **C) Chunking and PDF parsing quality — bad chunk size/overlap or mangled parsing is where RAG silently fails before embeddings or the model.** ✅
- D) The temperature is too high.

*Why:* The module stresses that chunking/parsing is where RAG quality is born or lost; investigate it before blaming the model or embeddings.

**Q2.** Why does `RecursiveCharacterTextSplitter` generally beat cutting text every N characters?

- A) It encrypts the chunks for safety.
- B) It guarantees every chunk is exactly the same token count.
- **C) It splits on a hierarchy of separators (paragraphs → lines → sentences → words), descending only when needed, so semantically related text stays together.** ✅
- D) It removes stop-words to shrink chunks.

*Why:* Respecting natural boundaries keeps related text intact, unlike a blind character cut that slices sentences and separates claims from context.

**Q3.** You set `chunk_overlap` to zero to save storage. What risk does the module warn about?

- A) Chunks will overlap anyway, wasting the setting.
- **B) A key sentence sitting exactly on a boundary can be orphaned — split across two chunks so neither retrieves cleanly — hurting continuity.** ✅
- C) Embeddings will refuse to process the chunks.
- D) Metadata is lost when overlap is zero.

*Why:* A small overlap preserves continuity across cuts; zero overlap risks orphaning boundary-spanning context (while too much overlap wastes space and returns near-duplicates).
