# 22 · Evaluation — Measuring Quality Instead of Guessing

## The plain-English version

You tweak a prompt, swap an embedding model, or add a re-ranker, and your three test questions look better. Did you actually *improve* the app — or did you just get lucky on those three, while quietly breaking ten others you didn't check? With normal software you'd run a test suite. With LLM apps, "does it work?" is fuzzy: there's rarely one correct output, and the model is non-deterministic. "It seems better" is not a quality bar.

**Evaluation is the discipline of measuring how good your AI application is, systematically and repeatably, so that changes are decisions backed by numbers rather than vibes.** You assemble examples, run your app on them, and score the outputs against criteria. Do this every time you change something, and you catch regressions before users do. In 2026, evals are table stakes, not an advanced topic — the difference between a demo and a product.

## The mental model: dataset + target + evaluators

Three ingredients, and LangSmith orchestrates them:

- **Dataset** — a collection of **examples**, each typically an *input* (and optionally a *reference/expected output*). This is your test set — the questions you'll grade against.
- **Target (the thing under test)** — your app, chain, or agent. The evaluation runs each example's input through it to produce an actual output.
- **Evaluators** — functions that *score* each output. An evaluator looks at the input, the actual output, and (optionally) the reference, and returns a score or label (e.g. correctness 0–1, "faithful: yes/no", latency in ms).

```python
from langsmith import evaluate

def correctness(run, example) -> dict:
    score = grade(run.outputs["answer"], example.outputs["expected"])
    return {"key": "correctness", "score": score}

evaluate(my_app, data="my-dataset", evaluators=[correctness])
```

You run this, and LangSmith gives you per-example scores and aggregate metrics — a report card for your app.

## Going deeper: kinds of evaluators

How do you actually score a free-text answer? Several strategies, used together:

- **Exact / heuristic match** — for structured or short outputs (a label, a number, valid JSON). Cheap, deterministic, but only works when there's one right answer.
- **LLM-as-judge** — use a *model* to grade the output against criteria ("Is this answer faithful to the provided context? Is it relevant to the question? Is it helpful?"). This is the workhorse for open-ended text, because it scales human-like judgment. You write a grading prompt (a rubric); the judge model returns a score with a rationale. Care is needed — judges have biases (length, position, self-preference) — so you calibrate them against human labels.
- **Human evaluation** — people score outputs directly, often via **annotation queues**. The gold standard for nuanced quality and for calibrating your automated evaluators, but slow and costly, so used as a sample.
- **Pairwise / preference** — instead of an absolute score, compare two outputs (A vs. B) and pick the better. Great for "is the new version better than the old?" comparisons where absolute scores are hard.

## The deepest layer: offline vs. online, RAG/agent specifics, and the loop

**Offline vs. online evaluation:**

- **Offline** — run against a fixed dataset *before* shipping (like CI tests). Catch regressions when you change a prompt/model/retriever. This is your pre-deployment gate.
- **Online** — evaluate *live production traffic* continuously, attaching automated evaluator scores to real traces (module 21). Catches drift, edge cases, and real-world failures that your dataset didn't anticipate. Production data feeds back into your datasets, closing the loop.

**Evaluating RAG (module 11) has two halves** — measure them separately or you won't know what's broken:

- **Retrieval quality** — did we fetch the right documents? (recall/precision over the known relevant chunks).
- **Generation quality** — given what was retrieved, is the answer **faithful** (grounded in the context, not hallucinated), **relevant** (actually answers the question), and **correct** (matches ground truth)? "Faithfulness" and "answer relevance" are the canonical RAG metrics.

**Evaluating agents (module 13)** adds dimensions beyond the final answer: did it choose the **right tools**? Take an efficient **trajectory** (not 11 steps for a 3-step task)? Avoid loops? You can evaluate the *single final output* or the *whole trajectory* of steps.

**The improvement loop — why this is the engine of quality:**

```
build → trace prod (21) → harvest failures into a dataset →
evaluate (22) → change prompt/model/retriever → re-evaluate →
ship the version that scores higher → repeat
```

Evaluation is what makes iteration *principled*. Without it you're guessing; with it, every change is measured. This loop — production traces becoming test cases, evals gating changes — is the core LangSmith workflow and the reason observability and evaluation are sibling modules.

**Datasets are living assets.** Seed them from real traces (module 21), add every bug you find as a new example (regression tests), version them, and split by scenario. A good dataset is one of the most valuable things your team owns — it encodes what "good" means for your app.

## Common gotchas

- **"It looks better on a few examples."** Anecdote isn't evaluation. Build a dataset and measure aggregate scores.
- **Trusting an uncalibrated LLM judge.** Judges have biases; validate them against human labels before relying on their scores.
- **Only measuring the final answer in RAG/agents.** Separate retrieval vs. generation (RAG) and consider trajectory (agents), or you won't know *what* to fix.
- **No online evaluation.** Offline datasets miss real-world edge cases; production drifts. Score live traffic too.
- **Static datasets.** If you never add new failure cases, your evals stop reflecting reality. Keep harvesting from production.
- **Optimizing the metric, not the product.** A judge rubric that rewards verbosity will give you verbose, worse answers. Make sure metrics track real user value.

## Key takeaways

- **Evaluation** measures app quality systematically via a **dataset** (examples), a **target** (your app), and **evaluators** (scorers) — replacing "seems better" with numbers.
- Evaluator types: **exact/heuristic**, **LLM-as-judge** (the workhorse for open text — but calibrate it), **human** (gold standard, sampled), and **pairwise preference**.
- Do both **offline** (pre-ship regression gate) and **online** (score live traffic) evaluation; for **RAG** measure **retrieval vs. generation** (faithfulness, relevance) separately, and for **agents** consider tool choice and **trajectory**.
- Evaluation powers the **improvement loop** — prod traces → datasets → evals gate changes → ship the higher-scoring version — making iteration principled rather than guesswork.

## Check your understanding

**Q1.** A RAG system's answers are unfaithful (it hallucinates). A teammate proposes a single "answer correctness" score to track quality. Why is that insufficient, per the module?

- A) Correctness can't be measured for RAG at all.
- **B) RAG quality has two separable halves — retrieval (did we fetch the right docs?) and generation (faithfulness/relevance given those docs) — and one blended score won't tell you which half is broken.** ✅
- C) Only human evaluation works for RAG.
- D) Faithfulness is identical to retrieval recall.

*Why:* You must measure retrieval and generation separately; a unfaithful answer could stem from bad retrieval or a model ignoring good context, and a single score hides which.

**Q2.** Why does the module insist on calibrating an LLM-as-judge before trusting its scores?

- A) Because judge models are slower than heuristics.
- B) Because judges can only output pass/fail.
- **C) Because judge models have biases (e.g. favoring length, position, or their own style), so their scores must be validated against human labels to be trustworthy.** ✅
- D) Because LLM judges can't read reference answers.

*Why:* LLM judges scale human-like grading but carry known biases; calibrating against human labels is what makes their scores reliable.

**Q3.** What is the relationship between observability (module 21) and evaluation in the LangSmith workflow?

- A) They're unrelated; evaluation replaces observability.
- B) Observability is only for production; evaluation only for development; they never connect.
- **C) Production traces are harvested into datasets and failures become regression examples; evals then gate changes — observability feeds evaluation in a continuous improvement loop.** ✅
- D) Evaluation generates traces that observability later deletes.

*Why:* The core loop turns real traces into test cases and uses evals to gate changes, which is why the two modules are siblings.

> **Visualization:** `visualization.html` is a mini eval dashboard — run two app versions against a dataset, watch per-example scores (correctness, faithfulness) populate, and see the aggregate verdict on whether the change was an improvement or a regression.