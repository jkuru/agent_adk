# Agentic Design Patterns — Setup Guide

## Quick Start

```bash
uv venv --python 3.12
source .venv/bin/activate          # macOS / Linux  |  .venv\Scripts\activate on Windows
uv pip install -r requirements.txt
cp .env.example .env               # then fill in your API key
uv run python -m ipykernel install --user --name=adk_env --display-name "Python (ADK)"
uv run jupyter notebook
```

---

## 1. Prerequisites

**Python 3.12** is recommended. MCP (used by Google ADK) requires Python 3.10+ and will fail on older versions.

### Install uv (if you don't have it)

- **macOS / Linux**: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Windows**: `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
- **pip fallback**: `pip install uv`

Verify: `uv --version`

uv will automatically download and manage Python 3.12 for you if it's not already installed.

---

## 2. Create the Virtual Environment

```bash
# From the project root — uv downloads Python 3.12 if needed
uv venv --python 3.12

# Activate it
source .venv/bin/activate      # macOS / Linux
.venv\Scripts\activate          # Windows
```

Your terminal prompt will show the venv name when active.

---

## 3. Install Packages

```bash
uv pip install -r requirements.txt
```

Then register the Jupyter kernel:

```bash
uv run python -m ipykernel install --user --name=adk_env --display-name "Python (ADK)"
```

---

## 4. Configure API Keys

```bash
cp .env.example .env
```

Then edit `.env`:

```
GOOGLE_API_KEY=your-google-api-key-here
```

Get your key at [Google AI Studio](https://aistudio.google.com/apikey) → "Create API Key".

The notebooks load keys automatically via `python-dotenv`. The `.env` file is in `.gitignore` so your keys are never committed.

---

## 5. Project Structure

```
agent_adk/
├── .env.example
├── .env                  (git-ignored)
├── .gitignore
├── requirements.txt
├── SETUP_GUIDE.md
├── README.md
└── notebooks/
    ├── Chapter_2_Routing.ipynb
    ├── Chapter_3_Parallelization.ipynb
    ├── Chapter_5_Tool_Use_Google_Search.ipynb
    ├── Chapter_5_Tool_Use_Code_Execution.ipynb
    ├── Chapter_5_Tool_Use_Vertex_AI_Search.ipynb
    ├── Chapter_7_Sequential.ipynb
    ├── Chapter_7_Parallel.ipynb
    ├── Chapter_7_Coordinator.ipynb
    ├── Chapter_7_Loop.ipynb
    └── Chapter_7_AgentTool.ipynb
```

---

## 6. Quick Verification

Run this in a notebook cell to confirm everything works:

```python
import os
from dotenv import load_dotenv
load_dotenv()

from google.adk.agents import Agent
from google.adk.tools import google_search

print("Google API Key set:", bool(os.environ.get("GOOGLE_API_KEY")))
print("All good! You're ready to run the notebooks.")
```

---

## 7. Troubleshooting

**"[Errno 13] Permission denied: '/usr/local/share'"** when installing the Jupyter kernel — You're missing the `--user` flag. Always use `--user` to install the kernel into your home directory (`~/Library/Jupyter/kernels/` on macOS) instead of the system-wide location:
```bash
uv run python -m ipykernel install --user --name=adk_env --display-name "Python (ADK)"
```

**"MCP requires Python 3.10 or above"** — Delete `.venv` and recreate with `uv venv --python 3.12`.

**"No module named google.adk"** — Activate your venv and run `uv pip install -r requirements.txt`.

**"GOOGLE_API_KEY not set"** — Ensure `.env` exists with a valid key and `load_dotenv()` is called.

**Wrong kernel in Jupyter** — Select "Python (ADK)" from Kernel > Change Kernel. If it doesn't appear, re-run the `ipykernel install --user` command.
