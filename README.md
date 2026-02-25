# Agentic Design Patterns — Google ADK

A hands-on collection of Jupyter notebooks demonstrating **agentic design patterns** using the **Google Agent Development Kit (ADK)** with Gemini models.

## What's Inside

| Chapter | Pattern | Notebooks |
|---------|---------|-----------|
| **2** | Routing | Coordinator delegates to specialist sub-agents |
| **3** | Parallelization | Parallel research agents with result synthesis |
| **5** | Tool Use | Google Search, Code Execution, Vertex AI Search |
| **7** | Multi-Agent Collaboration | Sequential, Parallel, Coordinator, Loop, AgentTool |

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

See `SETUP_GUIDE.md` for detailed instructions.
