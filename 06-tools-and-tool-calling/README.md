# 06 · Tools & Tool Calling — Letting Models Act

## The plain-English version

A language model on its own can only do one thing: produce text. It can't check today's weather, look up a customer record, run a calculation reliably, or send an email. It's a brain in a jar — brilliant at reasoning, but with no hands.

**Tools are the hands.** A tool is just a function you write — `get_weather(city)`, `search_database(query)`, `send_email(to, body)` — that you make available to the model. **Tool calling** is the mechanism by which the model, mid-conversation, says "I need to run `get_weather` with `city='Paris'`," your code runs it, and you hand the result back. Suddenly the brain can act on the world.

This is *the* idea that turns a chatbot into an agent. Everything in modules 13–20 is built on it.

## The mental model: the model requests, your code executes

Critically, **the model never runs the tool itself.** It only *requests* a call. The control loop is:

1. You give the model a question *and* a list of available tools (their names, descriptions, and argument schemas).
2. The model decides whether it needs a tool. If so, it returns an `AIMessage` with `.tool_calls = [{name, args, id}]` — structured, not prose.
3. **Your code** executes the actual function with those args.
4. You append a `ToolMessage(result, tool_call_id=id)` and call the model again.
5. The model uses the result to answer (or to request another tool).

This separation is a safety and control feature: you decide what tools exist, you validate the arguments, and you can refuse or modify a call before it runs.

## Defining a tool

The easiest way is the `@tool` decorator on a plain function:

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""   # <- the description the model sees
    return weather_api.lookup(city)
```

Three things become the tool's "interface" that the model reads:

- **Name** — the function name (`get_weather`).
- **Description** — the docstring. This is how the model knows *when* to use the tool. Treat it as prompt engineering: a vague docstring means the model picks the wrong tool.
- **Argument schema** — inferred from the type hints (`city: str`). You can use Pydantic for richer validation and per-argument descriptions via `Field`.

## Going deeper: binding tools and the execution loop

You attach tools to a model with `bind_tools`:

```python
model_with_tools = model.bind_tools([get_weather, search_db])
ai = model_with_tools.invoke("What's the weather in Paris?")
ai.tool_calls   # -> [{'name':'get_weather','args':{'city':'Paris'},'id':'call_1'}]
```

Then you execute and continue. By hand it looks like:

```python
messages = [HumanMessage("Weather in Paris?")]
ai = model_with_tools.invoke(messages); messages.append(ai)
for call in ai.tool_calls:
    result = get_weather.invoke(call["args"])
    messages.append(ToolMessage(result, tool_call_id=call["id"]))
final = model_with_tools.invoke(messages)
```

In practice you almost never write this loop yourself — the prebuilt `agent` (module 13) and LangGraph's `ToolNode` do it for you, including running multiple tool calls in parallel and handling errors. But knowing the loop exists is essential to debugging agents.

## The deepest layer: schemas, errors, and advanced tool features

**Rich argument schemas.** Use Pydantic for tools whose inputs need validation or documentation:

```python
class SearchArgs(BaseModel):
    query: str = Field(description="The search terms")
    limit: int = Field(default=5, description="Max results")

@tool(args_schema=SearchArgs)
def search(query: str, limit: int = 5) -> list: ...
```

**ToolNode and parallel calls.** Models can emit several tool calls at once. LangGraph's `ToolNode` executes them concurrently and appends all the `ToolMessage`s. This is a real latency win.

**Error handling.** Tools fail (network down, bad args). You can configure tools/`ToolNode` to catch exceptions and return the error *as a ToolMessage*, letting the model see the failure and retry or apologize, rather than crashing the whole run.

**`InjectedState` / `InjectedToolArg`.** Sometimes a tool needs data the model shouldn't (and can't) supply — the current user's ID, a database handle, the graph's state. These injection markers let your code provide those arguments at runtime while hiding them from the model's schema. Crucial for security: the model can't spoof a user ID it never sees.

**Tools returning more than strings.** A tool can return a `Command` to directly update graph state or steer control flow (e.g., a "hand off to another agent" tool in multi-agent systems, module 20). Tools can also return content blocks (images, files).

**ToolKits and MCP.** Collections of related tools ship as toolkits. The **Model Context Protocol (MCP)** lets you plug in whole external tool servers; LangChain can adapt MCP tools into native LangChain tools, instantly expanding what your agent can do.

**Human approval.** Because the model only *requests* a call, you can insert a human-in-the-loop checkpoint (module 18) before executing sensitive tools — approve, edit the args, or reject.

## Common gotchas

- **The docstring is not optional.** No/weak description = the model misuses or ignores the tool. This is the #1 cause of "my agent won't call my tool."
- **Type hints drive the schema.** Missing or wrong hints produce a broken argument schema. Be explicit.
- **The model can hallucinate arguments.** Validate with Pydantic; never trust tool args blindly, especially for destructive actions.
- **Forgetting the ToolMessage.** Every requested `tool_call_id` must get a matching `ToolMessage` before the next model call, or the provider errors (see module 03).
- **Too many tools confuse the model.** Beyond ~10–20 tools, selection accuracy drops; consider grouping, routing, or retrieval over tools.

## Key takeaways

- A **tool** is a function you expose to the model; **tool calling** lets the model *request* it while your code executes it — a deliberate separation for control and safety.
- Define tools with `@tool`; the **name, docstring, and typed arguments** form the interface the model reasons over.
- `bind_tools` attaches them; the **request → execute → ToolMessage → continue** loop is the engine of every agent.
- Advanced levers: **Pydantic schemas, parallel execution via ToolNode, error-as-ToolMessage, argument injection, human approval, and MCP** for external tools.

> **Visualization:** `visualization.html` animates the full tool-calling loop — the model reaches for a tool, your runtime executes it, and the result flows back so the model can answer.

---

## Check your understanding

**Q1.** A tool must act on behalf of the *currently logged-in user*, but you must ensure the model can never spoof a different user ID. Which mechanism fits?

- A) Put the user ID in the tool's docstring.
- B) Add `user_id` as a normal typed argument so the model fills it in.
- **C) Use `InjectedToolArg`/`InjectedState` so your runtime supplies the user ID at execution time while hiding it from the model's schema.** ✅
- D) Encode the user ID into the tool's name.

*Why:* Injected arguments are provided by your code and excluded from the schema the model sees, so the model can't fabricate or override them — a key security pattern.

**Q2.** Your agent stubbornly refuses to use a tool you defined. Following the module, what is the most probable root cause?

- A) The model temperature is too low.
- B) You used `@tool` instead of a Pydantic schema.
- **C) The tool's docstring/description is missing or vague, so the model can't tell *when* the tool applies.** ✅
- D) The tool returns a string instead of a dict.

*Why:* The docstring is the model's only signal for *when* to call a tool; a weak description is the #1 cause of unused or misused tools.

**Q3.** Why is "the model never runs the tool itself" framed as a feature rather than a limitation?

- A) Because it makes responses faster.
- B) Because it removes the need for `ToolMessage`s.
- **C) Because the model only *requests* a call, your code can validate args, require human approval, or reject the call before executing — giving you control and safety.** ✅
- D) Because it lets the model access your database credentials directly.

*Why:* The request/execute separation is a deliberate control boundary enabling validation, human-in-the-loop approval, and safe handling of destructive actions.
