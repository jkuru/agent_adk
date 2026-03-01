# Real MCP — From Mock to Production

*A learning exercise: how we built a fish-buying AI pipeline with 4 agents, 8 tools, and A2A negotiation — first as mock Python, then swapped the plumbing to real MCP. The agents didn't change at all.*

> **Note:** This article is part of a hands-on Google ADK learning curriculum based on Antonio Gullí's *Agentic Design Patterns* (2025). The code runs in a Jupyter notebook. Everything here is a teaching scenario — the goal is understanding MCP fundamentals, not building a production fish market.

---

## The Business Case

You're building an automated purchasing system for Pike Place Fish Market in Seattle. The system needs to discover which vendors carry a specific fish, check quality and freshness, negotiate a fair price, get human approval, record the deal, and learn from past purchases to make smarter decisions over time.

This is a realistic procurement scenario with real constraints: budget limits, quality minimums (sashimi requires freshness ≥ 8/10), delivery deadlines, and multiple competing vendors with different prices and reliability scores. The customer says "I need 5 lbs of fresh King Salmon for sashimi, budget $150, deliver by Saturday" — and the system handles everything.

### Why This Scenario Works for Learning MCP

Pike Place gives us natural reasons to use every pattern:

- **MCP discovery** — 5 vendors, each with different fish types. The buyer agent needs to find who carries King Salmon without hardcoding the answer.
- **Thin client / thick server** — vendor data, pricing rules, and purchase history belong on the server. The notebook client shouldn't store any of it.
- **A2A negotiation** — the vendor has a minimum price, the buyer has a maximum. Multi-round negotiation creates a natural agent-to-tool conversation.
- **Persistence** — the system buys fish repeatedly. The learning engine should remember past purchases and recommend better vendors over time.
- **Human-in-the-loop** — no one wants an AI spending $150 on fish without approval.

---

## System Context

Before diving into code, here's how the pieces fit together:

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROTOCOL STACK                                │
│                                                                  │
│  Layer 3: A2A ──── Agent ↔ Agent (negotiation, delegation)      │
│  Layer 2: MCP ──── App ↔ Tool Server (discovery, transport)     │
│  Layer 1: FC ───── LLM ↔ App (function calling, JSON schema)   │
│                                                                  │
│  This article focuses on Layer 2 (MCP) with Layer 1 (FC) under │
│  the hood, and touches Layer 3 (A2A) during negotiation.        │
└─────────────────────────────────────────────────────────────────┘
```

**Function Calling (Layer 1)** is how the LLM talks to your app — it outputs a JSON `function_call` with a name and args. This happens inside Gemini, managed by ADK. You don't touch it directly.

**MCP (Layer 2)** is how tools declare themselves and get executed across process boundaries. The FastMCP server exposes 8 tools over stdio. The client discovers them at runtime. This is the focus of this article.

**A2A (Layer 3)** is when tools become agents — when the server-side logic involves reasoning, not just computation. Our negotiation protocol sits at the boundary: the math is deterministic (Layer 2), but the LLM's strategy between rounds is reasoning (Layer 3).

### The Learning Progression

This article covers Part 5 of a 6-part notebook:

| Part | What It Teaches |
|------|----------------|
| Part 1 | Conceptual foundation: FC vs MCP vs A2A, design principles, 16 pattern catalog |
| Part 2 | The Pike Place scenario: vendor data, customer requirements |
| Part 3 | Infrastructure classes: memory, learning, goals, exceptions, negotiation |
| Part 4 | **Mock MCP** — same patterns with `FunctionTool()` and in-memory Python classes |
| **Part 5** | **Real MCP** — same patterns with `McpToolset`, FastMCP server, SQLite persistence |
| Part 6 | Reflection: pattern coverage, trade-offs, A2A upgrade roadmap |

The key insight: **Parts 4 and 5 implement the same 16 agentic patterns with the same agent code.** Only the plumbing changes. If you understand Part 4 (mock), Part 5 shows you what "real" looks like — and why MCP matters.

---

## Fundamental 1: The Server Declares Its Tools

In a traditional setup, you hardcode which functions an agent can call:

```python
# Part 4 (mock) — tools are hardcoded at definition time
buyer_agent = LlmAgent(
    name="BuyerAgent",
    model="gemini-2.0-flash",
    instruction="You are a fish buyer at Pike Place Market...",
    tools=[
        FunctionTool(query_vendor_catalog),   # ← you decide these
        FunctionTool(check_freshness),
        FunctionTool(estimate_shipping),
        FunctionTool(get_learning_insights),
    ],
)
```

With MCP, you give the agent a **connection**, not a function list:

```python
# Part 5 (real MCP) — tools are discovered from the server at runtime
buyer_agent = LlmAgent(
    name="BuyerAgent",
    model="gemini-2.0-flash",
    instruction="You are a fish buyer at Pike Place Market...",
    tools=[
        McpToolset(connection_params=MCP_CONNECTION),  # ← server decides
    ],
)
```

That `McpToolset` is a connection to a FastMCP server. The agent doesn't know what tools exist yet. It discovers them at runtime.

### How Discovery Works

When `Runner.run_async()` starts, ADK does a handshake before the LLM sees anything:

```
1. ADK spawns: python pike_place_server.py    (subprocess)
2. ADK sends:  "tools/list"                   (JSON-RPC over stdio)
3. Server responds with 8 tool schemas:
   [
     { "name": "get_market_snapshot",    "inputSchema": {...} },
     { "name": "query_inventory",        "inputSchema": {...} },
     { "name": "check_freshness",        "inputSchema": {...} },
     { "name": "estimate_shipping",      "inputSchema": {...} },
     { "name": "negotiate_price",        "inputSchema": {...} },
     { "name": "get_purchase_history",   "inputSchema": {...} },
     { "name": "run_learning_inference", "inputSchema": {...} },
     { "name": "record_purchase",        "inputSchema": {...} },
   ]
4. ADK injects these into the LLM's context as available tools
5. LLM reads descriptions + schemas, decides which to call
```

The LLM picks `get_market_snapshot` because its description says "Get a complete market snapshot in ONE call" — that matches the task. The LLM matched the tool by reading its description, the same way you'd read API docs.

---

## Fundamental 2: The Server Is a Real Process

The server is a standalone Python file that runs as a subprocess. Here's how it's structured:

```python
# pike_place_server.py
from fastmcp import FastMCP

mcp = FastMCP("Pike Place Vendor Server")

@mcp.tool()
def get_market_snapshot(fish_type: str, qty_lbs: float = 1.0) -> str:
    """Get a complete market snapshot for a fish type in ONE call.

    Returns inventory, freshness, quality grade, and shipping estimate
    for ALL vendors that carry this fish type.
    """
    results = []
    for vendor_name, vendor in VENDORS.items():
        if fish_type not in vendor["inventory"]:
            continue
        freshness = vendor["freshness_score"][fish_type]
        quality = ("EXCELLENT" if freshness >= 9 else "GOOD" if freshness >= 7
                   else "FAIR" if freshness >= 5 else "POOR")
        results.append({
            "vendor": vendor_name,
            "price_per_lb": vendor["price_per_lb"][fish_type],
            "freshness_score": freshness,
            "quality_grade": quality,
            "sashimi_suitable": freshness >= 8,
            # ... shipping, reliability, etc.
        })
    results.sort(key=lambda r: (-r["freshness_score"], r["price_per_lb"]))
    return json.dumps({"fish_type": fish_type, "snapshot": results})
```

The `@mcp.tool()` decorator is all it takes. FastMCP reads the function signature and docstring, generates a JSON Schema, and exposes it via `tools/list`. The LLM never sees your Python code — it sees the schema and description.

Every `@mcp.tool()` function becomes one MCP tool. Our server has 8.

### The Connection Config

The client connects via stdio — ADK spawns the server as a subprocess and communicates over stdin/stdout using JSON-RPC 2.0:

```python
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

MCP_CONNECTION = StdioConnectionParams(
    server_params=StdioServerParameters(
        command="python",
        args=["pike_place_server.py"],
    ),
    timeout=10.0,
)
```

No HTTP. No ports. No network. Just a subprocess pipe. This is the `stdio` transport — the simplest MCP transport and the right choice for local development and notebooks.

---

## Fundamental 3: Thin Client, Thick Server

The architectural split is deliberate:

```
┌─ Client (notebook) ──────────────────────┐
│  BuyerAgent        → 1 instruction only  │
│  LearnerAgent      → 1 instruction only  │
│  NegotiatorAgent   → 1 instruction only  │
│  RecorderAgent     → 1 instruction only  │
│  MCPOrchestrator   → Python driver       │
│                                          │
│  No vendor data. No business logic.      │
│  No persistence. No negotiation math.    │
└──────────────┬───────────────────────────┘
               │ stdio (JSON-RPC 2.0)
┌──────────────▼───────────────────────────┐
│  FastMCP Server                          │
│  ┌─ Stateless tools ──────────────────┐  │
│  │  get_market_snapshot               │  │
│  │  query_inventory, check_freshness  │  │
│  │  estimate_shipping                 │  │
│  └────────────────────────────────────┘  │
│  ┌─ A2A Negotiation Protocol ─────────┐  │
│  │  negotiate_price (tracks rounds)   │  │
│  └────────────────────────────────────┘  │
│  ┌─ ADK SqliteSessionService ─────────┐  │
│  │  get_purchase_history              │  │
│  │  run_learning_inference            │  │
│  │  record_purchase                   │  │
│  │  (persists to pike_place_memory.db)│  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

**Server owns**: vendor data, business logic, persistence (SQLite), negotiation state, learning engine.

**Client owns**: orchestration (which agent runs when), routing (which message goes to which agent), display (what the user sees), approval (human-in-the-loop gate).

Each agent is just an instruction string and an MCP connection. Here's the learner agent in its entirety:

```python
learner_agent = LlmAgent(
    name="LearnerAgent",
    model="gemini-2.0-flash",
    instruction="""
You query the learning engine for purchase history insights.

RULES:
1. Call run_learning_inference with the fish type provided.
2. Report EXACTLY what the learning engine returns — nothing more.
3. Do NOT call any other tools (no query_inventory, no check_freshness).
4. If the result says NO_HISTORY, say so clearly.
5. If the result says HAS_HISTORY, report: best vendor, average price,
   past purchases count, and best freshness seen.
""",
    tools=[McpToolset(connection_params=MCP_CONNECTION)],
)
```

Notice: the learner agent has access to all 8 server tools (via `McpToolset`), but the instruction constrains it to only call `run_learning_inference`. The LLM respects this because the instruction is clear. You could also create a server that exposes fewer tools to specific clients — but for a learning notebook, this approach shows the principle clearly.

---

## Fundamental 4: The Event Stream

When you call an agent via `Runner.run_async()`, you get an async generator of `Event` objects. Each event is one step in the agent's turn, streamed in real time:

```python
async for event in runner.run_async(
    user_id="orchestrator", session_id=session.id,
    new_message=content
):
    if event.content and event.content.parts:
        for part in event.content.parts:
            if hasattr(part, "function_call") and part.function_call:
                # LLM decided to call an MCP tool
                print(f"Tool: {part.function_call.name}({part.function_call.args})")
            elif hasattr(part, "function_response") and part.function_response:
                # MCP server returned a result
                print(f"Result: {part.function_response.response}")
            elif hasattr(part, "text") and part.text:
                # LLM is reasoning or giving its final answer
                print(f"LLM: {part.text}")
    if event.is_final_response():
        break
```

A typical event sequence for the buyer agent looks like:

```
Event 1: function_call     → get_market_snapshot("King Salmon", 5.0)
Event 2: function_response → {"vendors_found": 2, "snapshot": [...]}
Event 3: text              → "Wild Salmon Co has the best King Salmon..."
Event 4: is_final_response → done
```

This is how you observe the LLM's decision-making. It read the 8 tool schemas, picked `get_market_snapshot`, got the result, and synthesized an answer. All over MCP, all streamed live.

### Parallel vs Sequential

Whether the LLM calls tools in parallel or sequentially is the LLM's decision, not yours. If Gemini decides to check freshness for 3 vendors at once, it emits 3 `function_call` parts in a single event — ADK executes them concurrently. In practice with this pipeline, calls are mostly sequential because each depends on the previous result.

---

## Fundamental 5: A2A Negotiation Over MCP

The negotiator agent demonstrates the most interesting pattern. It calls `negotiate_price` multiple times, and the server tracks the negotiation state across calls.

Here's the full agent definition — read the instruction carefully, then watch how the LLM follows it in the event stream below:

```python
negotiator_agent = LlmAgent(
    name="NegotiatorAgent",
    model="gemini-2.0-flash",
    instruction="""
You are a tough but fair price negotiator for Pike Place Market.

STRATEGY — always try to get the best deal:
1. COMPUTE your opening offer: start 10-15% BELOW the target price.
   For example, if target is $28.00/lb, open around $23.80-$25.20/lb.
   Pick the exact number — this is YOUR strategic decision.
2. If you get a COUNTER_OFFER, DO NOT immediately accept it.
   Instead, offer a price BETWEEN your last offer and the counter
   (split the difference). This shows you're willing to negotiate.
3. If you get another COUNTER_OFFER, you may accept it by offering
   at or above the counter price — but only after at least 2 rounds
   of back-and-forth.
4. The server enforces a max of 5 rounds, so don't worry about
   going forever.

RULES:
- Never offer above your target price.
- Always call negotiate_price to make each offer — that's how
  the server tracks the A2A protocol.
- To ACCEPT a counter, call negotiate_price with offer >= the counter.
- Report the final deal with: price, total cost, rounds, and A2A trace.
""",
    tools=[McpToolset(connection_params=MCP_CONNECTION)],
)
```

Now watch the event stream and trace the instruction → behavior connection:

- **Instruction says**: "COMPUTE your opening offer: start 10-15% below target" → LLM calculates ~15% below $28 = $23.80, opens there (Round 1)
- **Instruction says**: "If COUNTER_OFFER, split the difference" → LLM gets counter $25.90, offers $24.85 = midpoint of $23.80 and $25.90 (Round 2)
- **Instruction says**: "After 2+ rounds, you may accept by offering >= the counter" → LLM gets counter $26.43, offers exactly $26.43 to accept (Round 3)

Here's what the event stream looks like during negotiation:

```
🔧 negotiate_price(offer=23.80, vendor="Wild Salmon Co", fish="King Salmon")
   ↳ {"status": "COUNTER_OFFER", "counter_price": 25.90, "rounds_used": 1}

💬 LLM: "They countered at $25.90. Let me split the difference — $24.85."

🔧 negotiate_price(offer=24.85, vendor="Wild Salmon Co", fish="King Salmon")
   ↳ {"status": "COUNTER_OFFER", "counter_price": 26.43, "rounds_used": 2}

💬 LLM: "They countered at $26.43. Two rounds done, let me accept."

🔧 negotiate_price(offer=26.43, vendor="Wild Salmon Co", fish="King Salmon")
   ↳ {"status": "ACCEPTED", "final_price": 26.43, "total_cost": 132.15}

💬 LLM: "Deal done. 5lb King Salmon at $26.43/lb, total $132.15."
```

The LLM **reasons between rounds**. It reads the counter-offer, decides a strategy ("split the difference"), and makes the next call. The server tracks rounds and enforces a 5-round maximum. The negotiation math is deterministic (server-side) — the strategy is LLM-driven (client-side).

### Server-Side Negotiation Logic

The server's `negotiate_price` tool implements a simple but effective protocol:

```python
@mcp.tool()
def negotiate_price(vendor_name, fish_type, qty_lbs, offer_price_per_lb):
    cost = vendor["cost_basis"][fish_type]
    min_price = cost * (1 + vendor["min_margin"])   # e.g., $18 * 1.20 = $21.60
    list_price = vendor["price_per_lb"][fish_type]   # e.g., $28.00

    if offer >= list_price:
        return ACCEPTED at list_price               # buyer offered full price
    if offer >= last_counter:
        return ACCEPTED at counter                   # buyer met the counter
    if rounds >= MAX_ROUNDS and offer >= min_price:
        return ACCEPTED at offer                     # deadline deal
    if offer >= min_price:
        counter = (offer + list_price) / 2           # midpoint convergence
        return COUNTER_OFFER at counter
    else:
        return REJECTED                              # below cost + margin
```

The midpoint formula creates natural convergence: each round, the gap shrinks by half. The LLM doesn't know this formula — it just sees the counter price and decides its next move.

### Two Brains, Two Jobs

There are two brains in this negotiation — and they do completely different jobs.

**The LLM brain (all strategy):** The orchestrator gives the NegotiatorAgent a target price ($28.00/lb) and says "decide your opening offer." The LLM reads its instruction ("start 10-15% below target"), does the math, and opens at $23.80. Every offer in every round is the LLM's decision — the opening, the counter-split, the final acceptance. Change the instruction and you change every offer.

**The server brain (all math):** When the server receives an offer of $23.80, it runs fixed rules:

```
cost_basis = $18.00
min_price  = $18 × 1.20 = $21.60    (20% margin, non-negotiable)
list_price = $28.00

Is $23.80 >= $28.00 (list)?     No  → don't accept
Is $23.80 >= $21.60 (minimum)?  Yes → counter-offer
counter = ($23.80 + $28.00) / 2 = $25.90
```

That's pure arithmetic. No reasoning, no judgment, no LLM. The server always counters at the midpoint between the offer and the list price. It will do this identically every single time for the same input.

**The split visualized across 3 rounds — who decides what:**

```
Round 1:
  LLM decides strategy   → offer $23.80                (instruction: "start 10-15%
    "Target is $28. I'll open ~15% below: $23.80."       below target" → $28 × 0.85)
  Server computes         → counter $25.90              (math: midpoint of 23.80
                                                         and list price 28.00)

Round 2:
  LLM decides strategy   → offer $24.85                (instruction: "split the
    "They countered at $25.90. My last offer was          difference" → midpoint of
     $23.80. Split the difference."                       $23.80 and $25.90)
  Server computes         → counter $26.43              (math: midpoint of 24.85
                                                         and 28.00)

Round 3:
  LLM decides strategy   → offer $26.43                (instruction: "after 2+
    "Two rounds done. Let me accept at $26.43."           rounds, accept by offering
                                                          >= the counter")
  Server computes         → ACCEPTED at $26.43          (rule: offer >= last counter)
```

Two brains, cleanly separated:

- **LLM (Gemini)** → decided every offer in every round. The opening price, the counter-splits, the acceptance. Change the instruction and every number changes. This is strategy.
- **Server (FastMCP)** → computed every counter-offer and acceptance. Pure math. Same output every time for the same input. This is business rules.

The orchestrator (Python) controls *when* to negotiate and *what the parameters are* (target price, vendor, quantity) — but it doesn't touch the pricing strategy. That's entirely the LLM's job.

**Why this two-brain split matters:** The LLM decides *how much* to offer — that requires reading context, following strategy, and making judgment calls. The server decides *what the vendor will accept* — that's margin math, and you never want an LLM doing margin math (it might hallucinate that $20 is above a $21.60 minimum). Each brain does what it's good at. LLMs are good at strategy in context. Servers are good at enforcing business rules.

---

## Fundamental 6: Persistence Across Subprocess Restarts

Each `_call_agent()` call may spawn a new server subprocess (stdio transport starts and stops per connection). But purchase history persists because the server uses ADK's `SqliteSessionService`:

```python
# Server-side persistence setup
from google.adk.sessions.sqlite_session_service import SqliteSessionService

DB_PATH = "pike_place_memory.db"
session_service = SqliteSessionService(db_path=DB_PATH)

@mcp.tool()
async def record_purchase(vendor_name, fish_type, qty_lbs,
                          price_per_lb, freshness_score):
    session = await _get_or_create_session()
    history = session.state.get("purchase_history", [])

    record = {
        "vendor": vendor_name, "fish_type": fish_type,
        "qty": qty_lbs, "price_per_lb": price_per_lb,
        "freshness": freshness_score,
        "timestamp": datetime.now().isoformat(),
    }
    history.append(record)

    # Persist to SQLite via ADK event
    await _update_state(session, purchase_history=history)
    return json.dumps({"status": "RECORDED", "record": record})
```

This is the key: **the subprocess dies after each agent call, but the SQLite file on disk keeps everything**. When the next agent call spawns a new subprocess, it reads from the same database.

The learning engine uses this persisted data:

```python
@mcp.tool()
async def run_learning_inference(fish_type):
    session = await _get_or_create_session()
    history = session.state.get("purchase_history", [])
    fish_history = [h for h in history if h["fish_type"] == fish_type]

    if not fish_history:
        return {"status": "NO_HISTORY", "message": f"No past purchases for {fish_type}"}

    # Best vendor by freshness-to-price ratio
    vendor_scores = {}
    for h in fish_history:
        score = h["freshness"] / max(h["price_per_lb"], 1)
        vendor_scores.setdefault(h["vendor"], []).append(score)
    best = max(avg_scores, key=avg_scores.get)

    return {"status": "HAS_HISTORY", "best_vendor": best, ...}
```

Run the pipeline twice: first time the learner says "no history." Second time it says "best vendor: Wild Salmon Co, average price: $26.43" — because Scenario 1's purchase was persisted to SQLite and survived the subprocess restart.

---

## Fundamental 7: The Orchestrator Is Python, Not an LLM

The `MCPOrchestrator` is a regular Python class. It decides which agent runs when, in what order, with what message. No LLM reasoning at the orchestration level:

```python
class MCPOrchestrator:
    async def run(self, fish_type, qty, budget, target_price, vendor):

        # Phase 1: Discovery — BuyerAgent finds vendors via MCP
        discovery = await self._call_agent(buyer_agent,
            f"Find all vendors with {fish_type}. Budget ${budget} for {qty}lb.")

        # Phase 2: Learning — LearnerAgent checks past purchases
        insights = await self._call_agent(learner_agent,
            f"Check what we know about {fish_type}.")

        # Phase 3: Negotiation — NegotiatorAgent decides opening offer + haggles
        negotiation = await self._call_agent(negotiator_agent,
            f"Negotiate with {vendor} for {qty}lb {fish_type}. "
            f"Target price: ${target_price}/lb. "
            f"Decide your opening offer (aim 10-15% below target).")

        # Phase 4: Human approval gate (Python, not LLM)
        if auto_approve:
            approved = True
        else:
            approved = input("Approve? (yes/no): ") == "yes"

        # Phase 5: Record — RecorderAgent persists to SQLite via MCP
        await self._call_agent(recorder_agent,
            f"Record purchase: vendor={vendor}, fish={fish_type}, ...")

        return deal
```

The orchestrator is deterministic. It always runs phases 1→2→3→4→5 in order. The LLM reasoning happens inside each `_call_agent()` call — the orchestrator just wires them together.

This is the design principle: **Python controls the flow, LLMs control the decisions within each step.** The negotiator decides *how much* to offer. The orchestrator decides *when* to negotiate.

---

## The Before/After

| | Part 4 (Mock) | Part 5 (Real MCP) |
|---|---|---|
| Tool calls | `FunctionTool(query_vendor_catalog)` | `McpToolset(connection_params=...)` over stdio |
| Memory | `PurchaseMemory` class (in-memory, dies with notebook) | `SqliteSessionService` (persists to disk) |
| Learning | `LearningEngine` class (in-memory) | `run_learning_inference` tool (reads from SQLite) |
| Negotiation | `MockMCPVendorServer.negotiate()` (direct call) | `negotiate_price` MCP tool (JSON-RPC over stdio) |
| Discovery | Hardcoded at definition time | Server declares tools at runtime via `tools/list` |
| Agent code | Same | Same |
| Orchestrator code | Same structure | Same structure |
| Business logic | Client-side Python classes | Server-side FastMCP tools |

The last two rows are the point: **the agents and orchestrator didn't change**. We moved the business logic to a server and swapped the plumbing. That's MCP's promise — and it works.

---

## Running It: What You'll See

```
############################################################
# MCP PIPELINE
# 5.0lb King Salmon, budget $150.0, target $28.0/lb
############################################################

┌─ Phase 1: DISCOVERY (BuyerAgent → MCP)
  │ AGENT: BuyerAgent (BUYER)
  │ 🔧 [ 1] + 1.895s  get_market_snapshot(fish_type='King Salmon', qty_lbs=5)
  │     ↳ Result: {"vendors_found": 2, "snapshot": [...]}
  │ 💬 LLM: Wild Salmon Co: $28/lb, freshness 9/10 (EXCELLENT, sashimi-grade)...
  │ Summary: 1 MCP calls, 3 events, 3.7s total

├─ Phase 2: LEARNING (LearnerAgent → 1 MCP call)
  │ AGENT: LearnerAgent (LEARN)
  │ 🔧 [ 1] + 1.719s  run_learning_inference(fish_type='King Salmon')
  │     ↳ Result: {"status": "NO_HISTORY"}
  │ Summary: 1 MCP calls, 3 events, 2.2s total

├─ Phase 3: NEGOTIATION (A2A on server)
  │ AGENT: NegotiatorAgent (NEGOTIATE)
  │ 🔧 [ 1] + 2.362s  negotiate_price(offer=23.80)  → COUNTER at $25.90
  │ 💬 LLM: "Let's meet in the middle at $24.85."
  │ 🔧 [ 2] + 3.378s  negotiate_price(offer=24.85)  → COUNTER at $26.43
  │ 💬 LLM: "Okay, let's accept at $26.43."
  │ 🔧 [ 3] + 4.506s  negotiate_price(offer=26.43)  → ACCEPTED
  │ Summary: 3 MCP calls, 7 events, 7.2s total
  │ 💰 Negotiated $26.43/lb (saved $1.57/lb)

├─ Phase 4: HUMAN APPROVAL
  │ ✅ Auto-approved

├─ Phase 5: RECORD (persist to ADK SqliteSessionService)
  │ 🔧 [ 1] + 1.837s  record_purchase(vendor=Wild Salmon Co, price=26.43)
  │ ✓ Purchase persisted to SQLite

└─ ✅ PIPELINE COMPLETE (15.9s total)
   {'vendor': 'Wild Salmon Co', 'price_per_lb': 26.43, 'total': 132.15}
```

6 MCP calls across 4 agents in 15.9 seconds. Every tool call went over stdio to the FastMCP server. The negotiator made 3 rounds of A2A negotiation, reasoning between each. The purchase was persisted to SQLite. Next run, the learner will remember this deal.

---

## Key Takeaways

1. **`McpToolset` replaces `FunctionTool`** — one connection gives the agent access to all server tools, discovered at runtime via `tools/list`. You don't hardcode function references.

2. **The server is the brain, the client is the coordinator.** Business logic, persistence, negotiation math — all server-side. Agents are just instructions + an MCP connection.

3. **The event stream is your debugger.** `async for event in runner.run_async()` gives you real-time visibility into every tool call, every response, every piece of LLM reasoning. Add timestamps and you can see parallel vs sequential execution.

4. **Persistence survives subprocess restarts.** `SqliteSessionService` writes to a file on disk. The server process can die and restart — the data is still there. This is how the learning engine improves across runs.

5. **A2A negotiation is just repeated MCP tool calls.** The LLM calls `negotiate_price` multiple times, the server tracks rounds and state. The LLM reasons between rounds. The server enforces the rules. Neither needs to know how the other works internally.

6. **The orchestrator is Python, not an LLM.** Deterministic flow control. Phases run in order. The LLM handles decisions within each phase. This separation makes the system predictable and debuggable.

`#MCP` `#GoogleADK` `#FastMCP` `#AgenticAI` `#A2A` `#ProtocolDesign`
