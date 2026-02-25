# Same LLM, Different Architecture, Completely Different Intelligence

*What I Learned Building Agentic AI Patterns with Google ADK*

**Jerry Kuru** · February 2026

---

## The Experiment

I built a 13-notebook curriculum to learn Google's Agent Development Kit (ADK) from scratch. Each notebook implements a different agentic design pattern: routing, parallelization, tool use, reflection, sequential pipelines, and planning agents.

Along the way I hit bugs that taught me more than the docs ever could. This article shares the two patterns that surprised me most and the key insight that connects them.

> **The core lesson:** how you wrap an LLM matters more than which LLM you use. Same model, same tools, same goal — but the architecture determines how it reasons, recovers, and adapts.

---

## Pattern 1: ReAct — One Brain Improvising

ReAct (Reason + Act) is a single agent that interleaves thinking and tool use. It reasons about what to do, calls a tool, observes the result, and decides what to do next. There is no upfront plan — the strategy emerges from the loop.

### The Architecture

```
Goal → [Think → Act → Observe → Think → Act → Observe → ...] → Final Answer

Three actors:
  Your Code   →  defines tools (Python functions)
  ADK         →  runs the multi-turn loop (invisible)
  Gemini      →  reasons + decides which tool to call next
```

### The Agent (9 Lines of Config)

```python
react_agent = LlmAgent(
    name="ReActResearcher",
    model="gemini-2.0-flash",
    instruction=REACT_INSTRUCTION,
    tools=[
        FunctionTool(web_search),
        FunctionTool(write_note),
        FunctionTool(list_notes),
    ],
)
```

### What Happened: 12 Autonomous Tool Calls

I gave the agent a research task: evaluate Rust vs Go for a payment gateway. The agent made 12 tool calls, all on its own. Here is the fascinating part — when a search returned vague results, the agent refined its query without anyone telling it to:

```
[2] TOOL CALL: web_search("Go memory management vs Rust")
    RESULT: "Several relevant articles found..."   # generic fallback

Gemini: "Okay, that's not very specific. Let me refine..."

[3] TOOL CALL: web_search("Go garbage collection vs Rust ownership")
    RESULT: detailed Rust ownership explanation    # matched!
```

> **No code told it to retry.** The reasoning happened inside Gemini — it compared the vague result to earlier rich results and autonomously decided to try a different query.

---

## Pattern 2: Plan-and-Execute — A Team of Specialists

Plan-and-Execute splits reasoning across three separate agents that cannot see each other's thinking. A Planner generates a JSON plan, an Executor runs one step at a time, and a Replanner reviews progress after each step.

### The Architecture

```
Goal
  |
  v
[Planner] ──► initial plan (JSON)
  |
  v
┌─ Pick next step ◄─────────────────────┐
│                                        │
v                                        │
[Executor] ──► step result               │
│                                        │
v                                        │
[Replanner] ──► continue ───────────────►┘
             └─► replan ─► revised ─────►┘
             └─► done ─► final summary
```

### Three Agents, Three Roles

| Agent | Has Tools? | Role |
|-------|-----------|------|
| PlannerAgent | No | Strategy only — outputs JSON plan |
| ExecutorAgent | Yes (4 tools) | Tactics only — completes one step |
| ReplannerAgent | No | Review only — continue, replan, or done |

### The Replanner in Action

Here is where it got interesting. The Executor was told to rename a file, but it had no rename tool. Instead of crashing, the system adapted:

```
[EXECUTOR] Step 3: Rename resource_management.py to rust_ownership_demo.py
[EXECUTOR] Result: I cannot directly rename files.

[REPLANNER] Revised plan — 2 steps remaining:
  Step 3: Save the script to a file named rust_ownership_demo.py
  Step 4: Execute rust_ownership_demo.py to verify it works
```

> **The Executor did not try to adapt** — it just said "I can't do this." But the Replanner, a completely different agent, looked at that failure and rewrote the plan to work around it. The intelligence came from the architecture, not from a single agent being clever.

---

## The Key Insight: Same LLM, Different Reasoning

Both patterns used the exact same model (gemini-2.0-flash), the same tools, and the same goal. But the way they recovered from problems was fundamentally different:

| Dimension | ReAct (Ch 8a) | Plan-and-Execute (Ch 8b) |
|-----------|--------------|--------------------------|
| Agents | 1 general | 3 specialized |
| How it plans | No plan — improvises each step | Explicit JSON plan upfront |
| How it adapts | Single agent refines its own queries | Replanner rewrites remaining steps |
| Recovery style | One brain self-correcting | A team compensating for each other |
| Trace visibility | Single agent trace | Across 3 agents |
| Best for | Short exploratory tasks | Long structured tasks (6+ steps) |

ReAct is one brain improvising. Plan-and-Execute is a team where each member has a narrow job and the system adapts through collaboration. Neither is better — they solve different problems.

---

## The Real Trade-Off: Context vs Control

The comparison table above hints at something deeper than just "use this pattern for short tasks, that pattern for long ones." The fundamental tension is between **context** and **control**.

### ReAct Maximizes Context

A single ReAct agent accumulates context across every tool call. It remembers that Step 1 returned rich data about Rust ownership, so when Step 2 returns a vague "Several relevant articles found..." result, it *knows* the difference. That accumulated context is what lets it self-correct — it's comparing new information against everything it's already learned.

The downside: all that context lives in one agent's head. There's no external artifact to inspect. If the agent "gets lost" — starts looping on the same query, or hallucinates a result it never received — there's no second opinion to catch it. The agent is both the worker and its own supervisor.

### Plan-and-Execute Maximizes Control

Plan-and-Execute trades context for control. The JSON plan is a tangible artifact: you can log it, version it, show it to a human for approval before execution begins. Each agent has a narrow scope and clear boundaries. The Planner never touches tools. The Executor never plans ahead. The Replanner never executes.

But this separation has a cost — what one reviewer called **"The Telephone Game" risk**. Because the Planner, Executor, and Replanner cannot see each other's thinking, information gets compressed at every handoff. The Executor sees a specific error, but it passes back a text summary. The Replanner reads that summary, not the raw error. If the summary loses a critical detail — an error code, a nuance in the failure — the Replanner might hallucinate a fix for a problem that doesn't actually exist.

**The architecture is only as good as the schema connecting the agents.** In our implementation, that schema is plain text messages. A production system might use structured JSON between agents, or even shared state (like the `output_key` blackboard pattern from Chapter 7) to reduce information loss.

### When to Choose Which

| You need... | Choose | Why |
|-------------|--------|-----|
| High agility, low overhead | ReAct | One agent, continuous context, fast pivots |
| Auditability for compliance | Plan-and-Execute | JSON plan is a loggable artifact |
| Short exploratory tasks (2-4 steps) | ReAct | Planning overhead isn't worth it |
| Long structured tasks (6+ steps) | Plan-and-Execute | ReAct can lose focus over many steps |
| Human-in-the-loop approval | Plan-and-Execute | Human reviews the plan before execution |
| Cost sensitivity | ReAct | Fewer LLM calls (1 agent vs 3) |

The honest answer: most real systems will use both. Use ReAct for the exploratory sub-tasks within a larger Plan-and-Execute framework. The patterns compose.

---

## What the Output Actually Looks Like

Seeing the raw execution output is where the learning clicks. Here is the ReAct agent's actual output — watch how Gemini's internal reasoning (its "thinking tokens") drive every decision:

### ReAct: The Agent Thinks Between Every Tool Call

```
[1] TOOL CALL: web_search("Rust ownership model")
    RESULT: "Rust's ownership system guarantees memory safety
             without a garbage collector..."

Gemini's thinking: "The ownership model seems to be a core
feature ensuring memory safety. Now I'll research Go's
memory management and compare that to Rust."

[2] TOOL CALL: web_search("Go memory management vs Rust")
    RESULT: "Several relevant articles found..."   ← generic!

Gemini's thinking: "Okay, that's not very specific.
Let me refine the search query."

[3] TOOL CALL: web_search("Go garbage collection vs Rust")
    RESULT: "Rust's ownership system guarantees..."  ← wrong match!

Gemini's thinking: "Still pretty generic. I need to be
more specific."
```

> **These "thinking" passages between tool calls are Gemini's reasoning tokens** — the model is literally thinking out loud. Each observation updates its mental model, and the next action reflects that updated understanding. No code orchestrates this; it's pure LLM reasoning.

### Plan-and-Execute: Agents That Can't See Each Other

Compare this to Plan-and-Execute, where the thinking happens across agent boundaries:

```
[PLANNER] 4 steps generated:    ← Planner thinks, outputs JSON
  Step 1: Research Rust ownership   (tool: web_search)
  Step 2: Write a Python script     (tool: write_file)
  Step 3: Save as rust_demo.py      (tool: none)
  Step 4: Run and verify            (tool: run_python)

[EXECUTOR] Step 3: Rename file
[EXECUTOR] Result: "I cannot directly rename files."

  ↑ The Executor doesn't adapt. It just reports failure.

[REPLANNER] Revised plan:          ← Different agent thinks
  Step 3: Save script to new filename
  Step 4: Execute to verify

  ↑ The Replanner sees the failure and rewrites the plan.
    Two separate "brains" collaborating through text.
```

### Thinking Tokens: The Hidden Driver

What you see in the output as text between tool calls are Gemini's reasoning tokens. In ReAct, these tokens form a continuous stream of consciousness — the agent builds context across every observation. It remembers that Step 1 returned rich data, so when Step 2 returns vague data, it knows the difference and adapts.

In Plan-and-Execute, each agent gets its own thinking tokens in isolation. The Planner reasons about decomposition. The Executor reasons about one step. The Replanner reasons about progress. No single agent has the full picture — but together, through the orchestrator passing messages between them, they achieve something none could alone.

**This is the fundamental tradeoff: ReAct gives one agent deep context across the whole task. Plan-and-Execute distributes reasoning but gains modularity and auditability.** The JSON plan is an artifact you can log, review, and debug — ReAct's reasoning exists only in the streaming trace.

---

## Code: The ReAct Agent

The entire ReAct agent is just an LlmAgent with tools and a good instruction. ADK handles the multi-turn loop internally:

```python
REACT_INSTRUCTION = """
You are a research agent that reasons step-by-step
and uses tools to gather information.

Your process for EVERY task:
1. Think about what information you need.
2. Call the appropriate tool.
3. Reflect on what you learned.
4. Repeat as many times as needed.
5. Save key findings using write_note.
6. Provide your complete answer.

IMPORTANT: You MUST actually call the tools —
do not simulate or role-play tool usage in text.
"""

react_agent = LlmAgent(
    name="ReActResearcher",
    model="gemini-2.0-flash",
    instruction=REACT_INSTRUCTION,
    tools=[FunctionTool(web_search),
           FunctionTool(write_note),
           FunctionTool(list_notes)],
)
```

## Code: The Plan-and-Execute Orchestrator

The orchestrator is a Python class, not an LLM. It drives the loop deterministically while each agent handles its narrow role:

```python
# Three agents, each with a focused role
planner_agent = LlmAgent(
    name="PlannerAgent",
    model="gemini-2.0-flash",
    instruction=PLANNER_INSTRUCTION,  # "Output ONLY valid JSON"
    generate_content_config=RETRY_CONFIG,  # auto-retry 429s
)

executor_agent = LlmAgent(
    name="ExecutorAgent",
    model="gemini-2.0-flash",
    instruction=EXECUTOR_INSTRUCTION,
    tools=[FunctionTool(web_search),
           FunctionTool(write_file),
           FunctionTool(read_file),
           FunctionTool(run_python)],
    generate_content_config=RETRY_CONFIG,
)

replanner_agent = LlmAgent(
    name="ReplannerAgent",
    model="gemini-2.0-flash",
    instruction=REPLANNER_INSTRUCTION,
    generate_content_config=RETRY_CONFIG,
)
```

## Code: ADK Retry Config for Rate Limits

One line of config saved hours of debugging. ADK retries 429 errors automatically with exponential backoff:

```python
from google.genai import types as genai_types

RETRY_CONFIG = genai_types.GenerateContentConfig(
    http_options=genai_types.HttpOptions(
        retry_options=genai_types.HttpRetryOptions(
            initial_delay=2,    # seconds before first retry
            attempts=3,         # 1 original + 2 retries
        ),
    ),
)
```

---

## 5 Things That Surprised Me

### 1. Gemini Will Role-Play Instead of Using Tools

If your prompt uses `Thought: / Action: / Observation:` labels (from the original ReAct paper), Gemini treats it as a text formatting exercise. It writes out simulated tool calls instead of actually invoking them through ADK's function calling protocol. The fix: remove those labels entirely and just say *"You MUST actually call the tools."*

### 2. runner.run_async() Returns an Async Generator

This burned all 13 notebooks. The pattern is `async for event in runner.run_async(...):` not `response = await runner.run_async(...)`. A subtle Python distinction that causes a TypeError if you get it wrong.

### 3. ADK Has Built-In Retry for Rate Limits

I initially built elaborate burst/cooldown logic in Python. Then I found ADK's `HttpRetryOptions` — three lines of config that handle 429 errors automatically with exponential backoff. I deleted 50 lines of manual throttling code.

### 4. The Replanner Is Where Plan-and-Execute Gets Smart

Without the Replanner, Plan-and-Execute is just a sequential pipeline that crashes on surprises. The Replanner gives it the ability to adapt — but unlike ReAct, the adaptation comes from a separate agent reviewing the situation, not from the executor itself.

### 5. Mock Tools Make Patterns Reproducible

Every notebook uses deterministic mock tools with hardcoded knowledge bases. This means the exact same research path happens every time you run it — perfect for learning, debugging, and writing articles about what happened.

---

## Try It Yourself

The full curriculum is 13 Jupyter notebooks covering routing, parallelization, tool use, code execution, enterprise pipelines, reflection, sequential and loop patterns, ReAct, and Plan-and-Execute. All built on Google ADK with mock tools for reproducibility.

**The takeaway I keep coming back to: the pattern you wrap around an LLM determines how it thinks.** Same model, same tools, same goal — but one architecture produces a single agent that self-corrects, and another produces a team that compensates for each other. Choosing the right pattern is an engineering decision, not an AI decision.

---

*Built with Google ADK · Gemini 2.0 Flash · Python · Jupyter*
