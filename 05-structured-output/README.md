# 05 · Structured Output — Getting Typed, Validated Data Back

## The plain-English version

By default a model replies with prose — a paragraph of text. That's great for a chatbot, but terrible if your *code* needs to use the answer. If you ask "extract the name, email, and amount from this invoice," you don't want a friendly sentence; you want `{"name": "...", "email": "...", "amount": 412.50}` so your program can store it in a database.

**Structured output is the technique of forcing the model to answer in a precise, machine-readable shape that you define in advance** — and getting back a validated object instead of a string you'd have to parse with fragile regexes. It's the bridge between the fuzzy, language world of the model and the strict, typed world of your software.

## The mental model

You describe the shape you want (a schema), and LangChain makes the model conform to it. The modern, one-line way:

```python
from pydantic import BaseModel, Field

class Invoice(BaseModel):
    name: str = Field(description="Customer's full name")
    email: str
    amount: float = Field(description="Total in USD")

structured_model = model.with_structured_output(Invoice)
result = structured_model.invoke("Bill to Jane Doe, jane@x.com, total $412.50")
# result is an Invoice instance, not text:
result.amount   # -> 412.5  (a real float)
```

`with_structured_output(Schema)` returns a new model that yields **instances of your schema**, already validated. No manual parsing. The schema can be a **Pydantic model** (best — gives you validation and types), a **TypedDict**, or a raw **JSON Schema** dict.

## Going deeper: how the model is actually constrained

There isn't one mechanism — LangChain picks the best the provider supports, and you can choose explicitly via the `method` argument:

**1. Tool/function calling (most common).** Your schema is presented to the model as a "tool" it must call. The model returns its answer as the *arguments* to that tool — which providers format as clean JSON. LangChain then parses those arguments into your object. This reuses the same tool-calling machinery from module 06.

**2. JSON mode / JSON schema mode.** Some providers offer a native "respond only with JSON matching this schema" mode (sometimes called *structured outputs* or *constrained decoding*). Here the provider constrains generation at the token level so the output is *guaranteed* to be valid JSON of the right shape. This is the most reliable when available.

**3. Prompt-and-parse (fallback).** For models without native support, you instruct the model to output JSON and use an **output parser** to extract and validate it.

The beauty of `with_structured_output` is that it hides this choice — you get an object back regardless of which mechanism the provider uses.

## Output parsers: the older, more flexible cousin

Before `with_structured_output` existed, you composed an **output parser** into your chain. They're still useful, especially for streaming and custom formats:

- **`StrOutputParser`** — pulls the plain string out of an `AIMessage`. The simplest, most common parser.
- **`PydanticOutputParser`** — parses text into a Pydantic object and exposes `get_format_instructions()` to inject into your prompt telling the model what JSON to produce.
- **`JsonOutputParser`** — parses to a dict; supports **partial parsing while streaming**, so you can render fields as they arrive.
- Specialized ones: comma-separated lists, enums, datetimes, etc.

```python
chain = prompt | model | StrOutputParser()   # parser as the last pipe stage
```

## The deepest layer: validation, retries, and when to use which

**Validation is the real payoff.** A Pydantic schema doesn't just shape the output — it *validates* it. If the model returns `amount: "a lot"`, Pydantic raises an error instead of silently passing garbage downstream. You can add constraints (`Field(gt=0)`, regex patterns, enums) and the model's output must satisfy them.

**Descriptions are prompt engineering.** The `description=` on each field is sent to the model as guidance. Good field descriptions dramatically improve extraction accuracy — treat them as mini-prompts, not documentation.

**Handling failures.** Models occasionally produce malformed output (especially smaller models in prompt-and-parse mode). Patterns to harden this: `OutputFixingParser` (sends the broken output back to a model to repair), `RetryOutputParser` (retries with the original prompt for context), or simply preferring native JSON-schema mode where the provider guarantees validity.

**Nested and complex shapes.** Schemas can nest: a list of line items, optional fields, unions ("classify as one of these types"). This makes structured output a general-purpose tool for **extraction**, **classification**, **routing** (decide which branch to take), and **tagging**.

**Optional vs. required.** Mark fields `Optional` when the source may not contain them; otherwise the model may hallucinate a value to satisfy a required field. This is a subtle but important reliability lever.

## Common gotchas

- **Not every model supports every method.** Tool-calling mode needs a tool-capable model; native JSON-schema mode needs provider support. `with_structured_output` falls back, but reliability varies.
- **Streaming + structured output is nuanced.** A fully-validated Pydantic object can't exist until the JSON is complete; for live partial rendering use `JsonOutputParser`.
- **Over-constraining hurts.** Extremely deep/strict schemas can confuse smaller models. Start simple, add constraints as needed.
- **Required fields invite hallucination** when the data is absent. Use `Optional` and clear descriptions.

## Key takeaways

- Structured output turns model replies into **typed, validated objects** your code can use directly.
- `model.with_structured_output(Schema)` is the one-line modern approach; the schema can be **Pydantic** (preferred), TypedDict, or JSON Schema.
- Under the hood it uses **tool calling**, **native JSON-schema mode**, or **prompt-and-parse** — chosen automatically per provider.
- **Output parsers** remain valuable for streaming and custom formats; `StrOutputParser` and `JsonOutputParser` are the workhorses.
- Field **descriptions, validation constraints, and `Optional`** are your levers for accuracy and reliability.

> **Visualization:** `visualization.html` shows the same messy invoice text being squeezed through a schema into a clean, validated object — with a toggle to see what happens when validation fails.

---

## Check your understanding

**Q1.** An extraction schema marks `discount: float` as **required**, but many source documents simply have no discount. What's the likely failure mode and the better design?

- A) The model returns `null`; no change needed.
- **B) The model may hallucinate a value to satisfy the required field; making it `Optional` lets it legitimately return nothing.** ✅
- C) Pydantic auto-fills 0.0, which is always correct.
- D) The provider rejects the schema entirely.

*Why:* Required fields pressure the model to invent data when the source lacks it. Marking genuinely-absent fields `Optional` (plus clear descriptions) reduces hallucination.

**Q2.** Why can a fully-validated Pydantic object not be emitted incrementally during streaming, and what do you use instead for live partial rendering?

- A) Pydantic doesn't support streaming at all; use `StrOutputParser`.
- B) Streaming requires tool-calling mode; use `bind_tools`.
- **C) A validated object can't exist until the JSON is complete; `JsonOutputParser` can emit partial dicts as fields arrive.** ✅
- D) You must disable validation with `model_kwargs`.

*Why:* Validation needs the whole object, so for progressive UI you use `JsonOutputParser`'s partial parsing and validate at the end if needed.

**Q3.** `with_structured_output` may use tool/function calling, native JSON-schema (constrained decoding), or prompt-and-parse depending on the provider. Why does this matter for reliability?

- A) It doesn't — all three are byte-identical.
- B) Prompt-and-parse is always the most reliable.
- **C) Native JSON-schema mode constrains generation at the token level so output is guaranteed valid, while prompt-and-parse (for unsupported models) can produce malformed JSON needing repair/retry.** ✅
- D) Tool-calling mode disables validation.

*Why:* The mechanism determines guarantees: constrained decoding is the most reliable when available; prompt-and-parse is the fragile fallback that benefits from `OutputFixingParser`/retries.
