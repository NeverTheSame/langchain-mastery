# 18 · Human-in-the-Loop — Pause, Approve, Resume

## The plain-English version

Would you let an AI agent delete files, send emails to customers, execute payments, or merge code — fully autonomously, with no one checking? Probably not. Some actions are too consequential to hand entirely to a model that occasionally makes mistakes or hallucinates arguments.

**Human-in-the-loop (HITL) is the ability to pause an agent mid-task, show a human what it's about to do, and let them approve, edit, or reject it before it continues.** It's the seatbelt for autonomous systems: you get the speed and capability of an agent, but with a human checkpoint at the moments that matter. The agent drafts the email; you glance at it and hit "send" (or fix a sentence first). The agent proposes a database delete; you approve or veto.

This is only possible because of persistence (module 17): to pause and wait — maybe for seconds, maybe for two days until you're back at your desk — the agent must durably save its state and resume exactly where it left off.

## The mental model: interrupt → wait → resume

The core primitive is **`interrupt()`**. Inside a node, you call it to *stop the graph* and surface some data to the outside world (the human). The graph's state is checkpointed; execution halts. Later, you provide the human's response and *resume*, and the graph continues from that exact point as if the pause never happened.

```python
from langgraph.types import interrupt, Command

def approval_node(state):
    decision = interrupt({                      # pause here, show this to the human
        "action": "send_email",
        "to": state["recipient"],
        "body": state["draft"],
    })
    if decision == "approve":
        return {"status": "sent"}
    return {"status": "cancelled"}

# ... later, after the human responds:
graph.invoke(Command(resume="approve"), config)   # resume with the human's input
```

Two halves: the node **raises** an interrupt (carrying info for the human); your application **resumes** the graph with `Command(resume=<human input>)`. Requires a **checkpointer** and a **`thread_id`** so the paused state can be found and continued.

## Going deeper: the four HITL patterns

Almost every HITL use case is one of these:

**1. Approve / reject.** The agent proposes a sensitive action (a tool call, a transaction). The human says yes or no. On reject, the graph routes elsewhere (apologize, try a different approach). The classic guardrail before destructive or costly tools.

**2. Edit / correct.** The human doesn't just approve — they *modify* what the agent produced before it proceeds. Fix a wrong argument, tweak the draft, adjust the plan. Implemented by combining the interrupt with `update_state` (module 17) to write the corrected values, then resume.

**3. Review tool calls.** A specialized approve/edit aimed at the model's *proposed tool calls* (module 06). Because the model only *requests* a tool, you can intercept the request, let a human vet or amend the arguments, and only then execute. This is the safety net for "the agent wanted to refund $10,000 — let me check that."

**4. Provide input / ask a human.** The agent realizes it's missing information only a person has ("what's the client's preferred meeting time?") and pauses to *ask*, treating the human as a kind of tool. The human's answer flows back into state and the agent continues.

## The deepest layer: why this is hard without LangGraph, and the subtleties

**Why persistence is the enabler.** A naive "wait for input" with a blocking call ties up a process and dies on restart. LangGraph instead *checkpoints and exits* — the run is durably paused with zero resources held. Hours or days later, a completely different process can load the thread and resume. This is the difference between a toy and a system that survives deploys and scales.

**Dynamic vs. static interrupts.** `interrupt()` inside a node is a **dynamic** interrupt — it fires based on logic/state ("only pause if amount > $1,000"). You can also configure **static** interrupts at compile time with `interrupt_before` / `interrupt_after` on specific nodes — "always pause before the `tools` node." Dynamic is more flexible; static is simple for blanket gating.

**Resuming carries the human's data.** `Command(resume=value)` injects the human's decision back as the return value of the `interrupt()` call. So the same node code handles both the pause and the continuation — elegant, but it means the node runs *up to* the interrupt, stops, and on resume effectively continues from that line with the provided value.

**State editing as a first-class tool.** Because you can `get_state`, `update_state`, and resume, HITL isn't limited to yes/no. A human can rewrite the agent's plan, delete a bad message, inject a fact, or change which node runs next — full steering. Combined with time travel (module 17), a reviewer can even rewind to before a mistake and send the agent down a corrected path.

**Where it runs.** In a chat UI, the interrupt surfaces as a prompt/button to the user. In an enterprise workflow, it might post to Slack for a manager's approval, or sit in a review queue. The LangGraph Platform (module 24) provides APIs to list interrupted threads and submit resumptions, which is what production HITL UIs are built on.

## Common gotchas

- **No checkpointer → no HITL.** Without persistence the graph can't pause and resume. This is the most common blocker.
- **Forgetting the `thread_id` on resume.** You must resume the *same* thread that was interrupted, or there's no paused state to continue.
- **Treating `interrupt()` like a normal return.** Execution truly stops; the node continues from the interrupt point only when you resume. Side effects *before* the interrupt will have already happened — put them after, or design for idempotency.
- **Gating everything.** Too many approval prompts and humans rubber-stamp without reading (approval fatigue). Gate only genuinely consequential actions; let routine ones flow.
- **No reject path.** If a human says "no," the graph needs an edge for that outcome — don't only handle "approve."

## Key takeaways

- **Human-in-the-loop** pauses an agent so a person can **approve, edit, reject, or provide input** before it proceeds — the seatbelt for consequential actions.
- The primitive is **`interrupt()`** (pause + surface data) plus **`Command(resume=...)`** (continue with the human's response); it **requires a checkpointer and `thread_id`**.
- Four patterns: **approve/reject, edit/correct, review tool calls, and ask-a-human** — the first three are the main safety guardrails around tools.
- Persistence is what makes durable, long-waiting pauses possible; combined with `update_state` and time travel, humans get **full steering** over the agent, not just yes/no.

## Check your understanding

**Q1.** A startup wants an agent that issues customer refunds but insists a human approve any refund over $500 — while small refunds flow automatically. Which approach fits best?

- A) A static `interrupt_before` the tools node, pausing on *every* refund.
- **B) A dynamic `interrupt()` inside the node that fires only when `amount > 500`, so small refunds proceed and large ones pause for approval.** ✅
- C) Remove the refund tool so nothing can go wrong.
- D) Lower the model temperature so it never proposes large refunds.

*Why:* Dynamic interrupts fire based on state/logic, enabling conditional gating; a static interrupt would pause on every refund, causing approval fatigue.

**Q2.** Why can LangGraph pause an agent for two days without holding server resources, whereas a naive `input()`-style wait cannot?

- A) LangGraph keeps the process alive in a low-power mode.
- **B) On `interrupt()` it checkpoints the state and exits; the run is durably paused with nothing held, so any process can later load the thread and resume.** ✅
- C) It streams the state to the human's browser to hold it.
- D) It re-runs the whole agent from scratch on resume.

*Why:* Durable pause = checkpoint-and-exit; a blocking wait ties up a process and dies on restart. Persistence is the enabler.

**Q3.** A reviewer doesn't just approve the agent's draft — they fix a wrong recipient before it sends. Which combination implements this "edit/correct" pattern?

- A) `interrupt()` alone, returning the original draft unchanged.
- B) A static interrupt with no state access.
- **C) An `interrupt()` to pause, plus `update_state` to write the corrected values, then `Command(resume=...)` to continue with the edits applied.** ✅
- D) Lowering `k` in the retriever.

*Why:* Edit/correct combines pausing with state editing (`update_state`) so the human's changes are persisted before the agent resumes.

> **Visualization:** `visualization.html` is an interactive approval gate — the agent proposes a risky action, the graph pauses, and you approve, edit, or reject and watch it resume down the corresponding path.