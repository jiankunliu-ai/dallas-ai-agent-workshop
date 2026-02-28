# Workshop Agent Architecture

This repo is designed to be read *alongside* `workshop.ipynb`. This document is a code tour of the two main agent implementations:

- `agent_lib.py`: code-execution agent (plan -> code -> exec -> fix)
- `research_agent.py`: research agent (plan queries -> web search -> synthesize report)

## Mental Model: What Is An "Agent" Here?

In this workshop, an "agent" is:

1. A **loop** that decides the next action (planning).
2. One or more **tools** it can call (e.g., run Python, search the web).
3. A **state object** that carries inputs/outputs across steps.
4. **Guardrails** that keep the system reliable for a live demo.

If you understand the state + nodes + tools, you understand the agent.

## `agent_lib.py`: Code-Execution Agent (LangGraph)

## Diagram: The Agent Graph

```text
                 +------------------+
                 |   planner_node   |
                 |  (LLM -> code)   |
                 +---------+--------+
                           |
                           v
                 +------------------+
                 |    exec_node     |
                 | (run_python tool)|
                 +----+--------+----+
                      |        |
                 ok=True   ok=False
                      |        |
                      v        v
                 +---------+  +------------------+
                 | finish  |  |    fixer_node    |
                 | (END)   |  | (LLM fixes code) |
                 +---------+  +---------+--------+
                                        |
                                        v
                                   +---------+
                                   | exec    |
                                   +---------+
```

How to read this:

- The **state** flows along the arrows.
- `planner_node` creates a plan and a runnable Python snippet (must print).
- `exec_node` executes the code and captures stdout/stderr.
- If execution succeeds, the workflow finishes.
- If execution fails, `fixer_node` uses the error output to generate a corrected snippet and retries.

### State

`AgentState` is the shared state that flows through the graph:

- `task`: user task string
- `plan`: LLM text (plan + code block)
- `code`: extracted Python code to run
- `last_run`: result dict from the Python tool (`ok`, `stdout`, `stderr`, ...)
- `attempts`: retry counter
- `done`: termination flag

### Nodes (The Loop)

The graph is intentionally simple and visible:

`plan -> exec -> (fix -> exec)* -> finish`

- `planner_node(state)`
  - Calls the LLM with a prompt that *requires* executable code that prints the final answer.
  - Extracts code from a fenced block and stores it in `state["code"]`.

- `exec_node(state)`
  - Runs the generated code via `tools.run_python(...)`.
  - Includes a fallback guard: if the model outputs a single expression like `2+2` without `print(...)`,
    it rewrites it to `print(2+2)` so stdout is always produced for simple tasks.
  - Prints the generated code for workshop debugging.

- `fixer_node(state)`
  - If execution failed, sends the previous code + stdout/stderr back to the LLM and asks for a corrected code block.

- `decide_node(state)`
  - Stops early on success (`ok == True`) or after a small retry limit.

### OpenRouter Client Configuration

`make_client()` builds an OpenRouter-backed OpenAI client:

- `base_url="https://openrouter.ai/api/v1"`
- `api_key=os.environ["OPENROUTER_API_KEY"].strip()`
- `default_headers`:
  - `HTTP-Referer: http://localhost:8888`
  - `X-Title: Dallas Agent Workshop`

The model defaults to:

- `arcee-ai/trinity-large-preview:free`

### Deterministic `.env` Loading

`run_task()` loads `.env` via:

- `load_dotenv(dotenv_path=".env", override=False)`

This is workshop-safe because it:

- Uses `.env` as the local source of truth.
- Avoids unexpectedly overriding env vars if the user intentionally sets them in their shell/runtime.

Tip: if the notebook runtime inherits a stale `OPENROUTER_API_KEY` from your shell, you may see a 401. In that case:

- Restart kernel/runtime and ensure env vars are set correctly
- Or run `unset OPENROUTER_API_KEY OPENROUTER_MODEL` before starting Jupyter

## `tools.py`: The Python Execution Tool

The agent does *not* execute code directly in-process. It calls `run_python(code)` which:

- Writes the code to a temporary file
- Runs a fresh Python interpreter with a timeout
- Captures `stdout`/`stderr` and returns them as a dict
- Blocks a small set of risky patterns (imports, `open(...)`, `eval(...)`, `exec(...)`, etc.)

Important: this is "meetup-grade" safety, not a hardened sandbox.

## `research_agent.py`: Research Agent (Plan -> Search -> Synthesize)

This is a second workflow that demonstrates tool use beyond Python execution.

## Diagram: The Research Graph

```text
               +-------------------+
               |      planner      |
               | (LLM -> queries)  |
               +---------+---------+
                         |
                         v
               +-------------------+
               |     searcher      |
               | (Tavily -> urls)  |
               +---------+---------+
                         |
                         v
               +-------------------+
               |    synthesizer    |
               | (LLM -> report)   |
               +---------+---------+
                         |
                         v
                        END
```

How to read this:

- The **state** carries `question`, `search_queries`, `raw_results`, and `report`.
- `planner` proposes a few targeted queries.
- `searcher` calls Tavily and collects a small set of sources.
- `synthesizer` uses only those sources to write a structured report with citations.

### Dependencies and Keys

Research requires:

- `TAVILY_API_KEY` for web search (Tavily)
- `OPENROUTER_API_KEY` for synthesis (OpenRouter)

`research_agent.py` is designed so that importing the module does not immediately fail if keys are missing.
Instead, it raises a clear error when the workflow tries to use the missing tool.

### Workflow

The research graph is:

`planner -> searcher -> synthesizer`

- `planner`
  - Uses the LLM to propose 2-4 search queries.
- `searcher`
  - Runs Tavily searches and collects a small set of sources.
- `synthesizer`
  - Asks the LLM to produce a structured report and to cite sources using `[1]`, `[2]`, etc.

### Output Shape

The final result is a dict containing:

- `question`
- `search_queries`
- `raw_results` (sources)
- `report` (final structured write-up)

## How To Use This Document In The Workshop

- Live walkthrough: show the `agent_lib.py` loop and how `AgentState` changes per node.
- Hands-on: have attendees edit prompts, add a node, or adjust `tools.py` restrictions.
- Debugging: show how stdout/stderr + generated code logging makes failures tractable.

## Multi-Agent + Handoffs (Conceptual Only)

We are not adding new hands-on examples in this repo for multi-agent systems, and we are not changing the workshop notebooks.
This section is a deeper conceptual guide so attendees can recognize the same architecture patterns in real systems.

In practice, "multi-agent" usually means an **orchestrator** delegates work to **specialists** and then combines results.
In LangGraph terms, this can be implemented as:

- A single graph with multiple nodes (each node acts like a specialist), and state is the handoff mechanism.
- Multiple graphs composed together (a higher-level graph calls sub-graphs), again with explicit state handoffs.

### Diagram: Orchestrator + Specialists

```text
User Request
    |
    v
+-------------------+        +-------------------+
|   Orchestrator    |------->|  Specialist A     |
| (routing + state  |<-------| (tool / skill 1)  |
|  + guardrails)    |        +-------------------+
|                   |------->+-------------------+
|                   |<-------|  Specialist B     |
|                   |        | (tool / skill 2)  |
|                   |------->+-------------------+
|                   |<-------|  Specialist C     |
+---------+---------+        | (writeup / QA)    |
          |                  +-------------------+
          v
     Final Output
```

How to read this:

- The orchestrator is responsible for **what happens next** and for maintaining a **single source of truth state**.
- Each specialist is responsible for **one kind of work** and should return a predictable output format.
- Handoffs are not "vibes" or chat history; they are **explicit state fields** and **tool outputs**.

### Key Concepts

Key ideas to communicate (and to enforce in code):

- **Handoff contract**: define what data moves between steps (schemas, fields, expected formats).
- **Role separation**: planner nodes plan; tool nodes use tools; synthesizer nodes write conclusions.
- **Verification**: each step should produce observable outputs so failures are debuggable.
- **Tool boundaries**: restrict which step can call which tool to reduce risk and confusion.
- **Governance**: log tool calls, keep secrets out of prompts, and avoid giving broad permissions by default.

Mapping to this workshop:

- `agent_lib.py` already demonstrates handoffs via `AgentState` across `planner_node -> exec_node -> fixer_node`.
- `research_agent.py` demonstrates a workflow handoff across `planner -> searcher -> synthesizer`.

So even though we only run "one agent" at a time in the workshop, the architecture is already the same building block used for
multi-agent systems: **explicit state + specialized steps + controlled tools**.

### Handoff Contracts (What Makes Multi-Agent Stable)

A handoff contract is just a schema. Even if you're not using formal schema validation, you should think in terms of:

- Inputs: what fields a node reads
- Outputs: what fields it writes
- Invariants: what must be true after the node runs

Example (conceptual) contract for a research-style multi-agent system:

```text
state = {
  question: str,
  queries: list[str],        # created by planner
  sources: list[Source],     # created by searcher
  claims: list[Claim],       # optional: created by extractor/analyst
  report: str,               # created by synthesizer
  errors: list[str],         # append-only
  trace: list[StepLog],      # append-only (for debugging / governance)
}
```

If you treat the handoff like a contract, you get:

- faster debugging (you can inspect state at each step)
- better prompts (you know what to ask for)
- safer tool use (only certain nodes touch certain tools)

### Context vs. State vs. Memory

These are related but different:

- **State**: structured data passed between steps in a single run (the handoff mechanism).
- **Context**: what you include in the model prompt *right now* (often derived from state).
- **Memory**: what persists across runs (optional), e.g. user preferences or past results.

Workshop takeaway:

- Keep **state** explicit and small.
- Keep **context** minimal and structured (include only what the next node needs).
- Add **memory** only after you can pass repeatable tests, because memory can amplify mistakes.

### Governance & Safety Checklist

For a meetup-grade multi-agent system that runs reliably:

- Do not leak secrets into prompts or logs.
- Log tool calls and key intermediate outputs (small, redacted).
- Add timeouts and retry limits to every external call.
- Prefer allowlists over blocklists for dangerous tools.
- Always force observable outputs (stdout, structured tables/JSON, citations).
- Add a "stop" mechanism: max steps, max tokens, and user cancel.

### When To Use Multi-Agent (And When Not To)

Use multi-agent patterns when:

- The task naturally decomposes into steps with different tools/skills
- You need traceability (what sources led to what claims)
- You want clearer failure modes (each step can be validated)

Avoid multi-agent complexity when:

- A single prompt is enough
- Tool calls are minimal
- You cannot validate intermediate results
