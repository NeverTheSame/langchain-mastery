# 04 · Prompt Templates — Reliable, Reusable Prompts

## The plain-English version

A prompt is the instruction you give the model. In a real app, that instruction isn't fixed — it changes with every request. "Summarize *this* article." "Answer *this* question using *these* documents." The parts in italics are variables that get filled in at runtime.

A **prompt template** is a fill-in-the-blanks form for prompts. You write the wording once, mark the blanks, and the template safely inserts the user's data each time. It's the difference between gluing strings together by hand (error-prone, insecure, hard to maintain) and using a proper, reusable form.

Why does this deserve its own concept? Because the wording of a prompt is one of the highest-leverage things in an LLM app — small changes in phrasing dramatically change output quality. Templates make that wording a **named, versionable, reusable artifact** instead of a string buried in your code.

## The mental model

There are two everyday template types:

**`PromptTemplate`** — produces a single string. Good for completion-style models or when you need one block of text.

```python
from langchain_core.prompts import PromptTemplate
tpl = PromptTemplate.from_template("Summarize this in {n} words:\n\n{text}")
tpl.invoke({"n": 20, "text": article})   # -> a StringPromptValue
```

**`ChatPromptTemplate`** — produces a *list of messages* (the format chat models actually want). This is what you'll use 95% of the time.

```python
from langchain_core.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {tone} assistant."),
    ("human", "{question}"),
])
prompt.invoke({"tone": "friendly", "question": "What is RAG?"})
```

A template is itself a **Runnable** (module 07), so it slots directly into a chain: `prompt | model | parser`. That single fact is why prompts compose so cleanly with everything else.

## Going deeper: the pieces that make templates powerful

**`MessagesPlaceholder`** — reserves a slot for a *list* of messages to be dropped in later. This is how you inject conversation history or retrieved context dynamically:

```python
ChatPromptTemplate.from_messages([
    ("system", "You are helpful."),
    MessagesPlaceholder("history"),   # prior turns go here
    ("human", "{question}"),
])
```

**Partial formatting.** You can pre-fill *some* variables and leave others for later with `prompt.partial(tone="formal")`. Useful when part of the prompt is known at setup time (a date, a persona) and the rest at call time.

**Few-shot templates.** `FewShotChatMessagePromptTemplate` formats a list of input/output examples into the prompt — the standard way to "show, don't tell" the model the format you want. Combined with an **example selector**, you can dynamically pick the *most relevant* examples (often by embedding similarity) for each input, instead of always sending the same static set.

**Template formats.** By default, templates use Python's `{variable}` (f-string) syntax. You can opt into other formats (e.g. mustache `{{variable}}`) when needed.

## The deepest layer: prompts as managed artifacts

**Prompts belong in version control — and in LangSmith.** The most consequential idea in this module is that a prompt is *not* a throwaway string; it's a tuned artifact whose wording you'll iterate on for the life of the product. LangSmith's Prompt Hub (module 23) lets you store prompts, version them, test variants, and pull a specific version into code with `hub.pull("my-prompt:v3")`. This separates "what the model is told" from "how the code runs," so non-engineers can tune prompts and you can roll back a bad prompt change without a code deploy.

**Why templates beat f-strings.** Beyond convenience, templates give you: declared input variables (so you get an error if you forget one, not a silently broken prompt), safe insertion, compatibility with the Runnable streaming/batching machinery, automatic tracing of the exact rendered prompt in LangSmith, and a clean separation between structure and data.

**Prompt rendering is observable.** Because the template is a Runnable, LangSmith records the *exact* messages produced after substitution. When an answer is wrong, you can see precisely what the model was shown — which is usually where the bug is.

## Common gotchas

- **Curly-brace collisions.** If your template text contains literal `{` or `}` (e.g. JSON examples), escape them as `{{` and `}}`, or the formatter will think they're variables.
- **Missing variables fail loudly** — which is good. Provide every declared variable or pass it via `partial`.
- **Don't smuggle untrusted user text into the *system* slot.** Keep user input in the human slot; putting it in the system instructions invites prompt injection.
- **MessagesPlaceholder expects a list of messages,** not a string. Feeding it the wrong type is a common first-time error.

## Key takeaways

- A prompt template is a **fill-in-the-blanks form**: write the wording once, insert data safely at runtime.
- Use **`ChatPromptTemplate`** for chat models; it outputs a message list and is a **Runnable**, so it pipes straight into `prompt | model | parser`.
- **`MessagesPlaceholder`** injects history/context; **few-shot templates** and **example selectors** teach format by example.
- Treat prompts as **versioned artifacts** (LangSmith Prompt Hub), not buried strings — it's one of the highest-leverage practices in the whole stack.

> **Visualization:** `visualization.html` is a live template playground — edit the variables and watch the rendered message list update in real time.

---

## Check your understanding

**Q1.** Your template text includes a literal JSON example containing `{"id": 1}`. The template throws an error about an unexpected variable. What's the fix?

- A) Switch to `PromptTemplate` because it ignores braces.
- B) Move the JSON into the system message.
- **C) Escape the literal braces by doubling them: `{{"id": 1}}`, so the f-string formatter treats them as literals, not variables.** ✅
- D) Remove all type hints from the example.

*Why:* Default templates use f-string syntax, so literal `{`/`}` must be escaped as `{{`/`}}` or they're parsed as variable placeholders.

**Q2.** A non-engineer needs to tune the wording of a production prompt without a code deploy, and you want to roll back instantly if it regresses. Which approach matches the module's "prompts as managed artifacts" idea?

- A) Hard-code the prompt as an f-string and redeploy on each change.
- **B) Store the prompt in LangSmith's Prompt Hub with versions and pull a specific version (e.g. `:v3`) into code, decoupling wording from code.** ✅
- C) Keep prompts only in the system message so they can't be changed.
- D) Use `partial()` to lock the prompt so it never changes.

*Why:* Treating prompts as versioned Hub artifacts separates "what the model is told" from "how the code runs," enabling non-engineer tuning and rollback without deploys.

**Q3.** What distinguishes `MessagesPlaceholder` from a normal `{variable}` slot?

- A) It only accepts strings and rejects message objects.
- B) It must always be the first item in the template.
- **C) It reserves a slot for a *list of messages* (e.g. conversation history) to be injected, not a single string value.** ✅
- D) It disables f-string formatting for the whole template.

*Why:* `MessagesPlaceholder` injects an entire list of messages (like history or retrieved context); feeding it a plain string is a common error.
