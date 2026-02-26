# Agentic Design Patterns — Google ADK

A hands-on collection of Jupyter notebooks demonstrating **agentic design patterns** using the **Google Agent Development Kit (ADK)** with Gemini models. Covers 14+ patterns from basic routing to a full client-server MCP pipeline.

## What's Inside

| Chapter | Pattern | Notebook(s) |
|---------|---------|-------------|
| **2** | Routing | Coordinator delegates to specialist sub-agents |
| **3** | Parallelization | Parallel research agents with result synthesis |
| **5** | Tool Use | Google Search, Code Execution, Vertex AI Search (3 notebooks) |
| **6** | Reflection | Enterprise supply chain with quality gates |
| **7** | Multi-Agent Collaboration | Sequential, Parallel, Coordinator, Loop, AgentTool (5 notebooks) |
| **8a** | ReAct | Single agent with iterative reasoning + tool use |
| **8b** | Plan-and-Execute | Planner → Executor → Replanner pipeline |
| **MCP** | MCP Client-Server | Real FastMCP server + ADK McpToolset client |
| **Capstone** | All 14 Patterns | Pike Place Fish Market end-to-end buying system |

## Architecture Highlights

**Chapter MCP Server** splits agents into a lightweight **client** (ADK notebook) and a heavy **server** (FastMCP subprocess):

```
CLIENT (Notebook)                    SERVER (pike_place_server.py)
├── BuyerAgent (LlmAgent)    ──►    ├── Vendor data (5 vendors)
├── NegotiatorAgent           ──►    ├── A2A Negotiation Protocol
├── RecorderAgent             ──►    ├── PurchaseMemory
└── MCPOrchestrator (Python)         ├── LearningEngine
                                     └── 7 @mcp.tool() endpoints
```

**Capstone** composes all patterns into one pipeline: routing, parallelization, tool use, reflection, multi-agent collaboration, sequential, AgentTool, ReAct, Plan-and-Execute, memory management, learning & adaptation, MCP server, A2A protocol, goal tracking, exception handling, and human-in-the-loop.

## Quick Start

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
cp .env.example .env               # add your GOOGLE_API_KEY
uv run python -m ipykernel install --user --name=adk_env --display-name "Python (ADK)"
uv run jupyter notebook
```

## Requirements

- [uv](https://docs.astral.sh/uv/) — fast Python package manager
- Python 3.12 (3.10+ minimum for MCP compatibility)
- Google API Key from [Google AI Studio](https://aistudio.google.com/apikey)
- [FastMCP](https://gofastmcp.com/) — for the MCP Server chapter (`pip install fastmcp>=2.0.0`)

## Key Patterns Demonstrated

| Pattern | Where |
|---------|-------|
| ADK native retry (`HttpRetryOptions`) | All notebooks (Ch 8b+) |
| Deterministic mock data | Ch 6, 8a, 8b, Capstone |
| `%%writefile` for MCP server creation | Chapter MCP Server |
| `McpToolset` + `StdioServerParameters` | Chapter MCP Server |
| A2A negotiation protocol | Capstone + MCP Server |
| Server-side memory & learning | MCP Server |

See `SETUP_GUIDE.md` for detailed setup instructions and recommended notebook order.
