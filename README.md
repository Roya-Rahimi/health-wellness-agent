# Health & Wellness Coach Agent

An LLM-powered agent that answers health questions and decides when to call external tools for information it cannot reliably answer on its own. Built in Python with LLaMA 3.2 served through Ollama.

Course project, Machine Learning I, University of North Carolina Wilmington (Spring 2026).

---

## What it does

The agent takes a health question and either answers directly or calls one of four tools, then uses the tool's output to compose a response. It keeps conversation history across turns and runs behind a Gradio web interface.

**Tools**

| Tool | Source | Purpose |
|---|---|---|
| `check_symptoms` | DuckDuckGo (scoped to Mayo Clinic, WebMD, Healthline) | Look up symptom information |
| `find_nearby_clinics` | OpenStreetMap Nominatim + Overpass | Geocode a location, then find clinics, hospitals or pharmacies within 5 km |
| `get_medication_info` | OpenFDA drug label API | Retrieve indications, warnings, dosage and adverse reactions |
| `get_wellness_tips` | DuckDuckGo (scoped to Healthline, Mayo Clinic) | Find wellness and diet guidance |

## How it works

```
User input (Gradio UI)
        │
        ▼
  LLaMA 3.2 via Ollama
        │
        ├─ returns JSON  ──► parse_tool_call() ──► run tool ──► feed result back
        │
        └─ returns prose ──► final answer to user
```

The agent uses a JSON tool-calling protocol. The system prompt instructs the model to reply with `{"tool": ..., "args": ...}` and nothing else when a tool is needed. Because models often wrap that JSON in explanatory text, `parse_tool_call()` extracts the JSON substring and validates that it contains both keys; anything else is treated as a final answer. The loop runs up to six iterations per turn.

Every tool returns a structured dict and handles its own errors, so a failed API call surfaces as data rather than an exception.

## Known limitation: grounding

Testing surfaced a reliability problem worth documenting.

When `find_nearby_clinics` returned no results, the agent stated that no results were found and then listed clinic names, addresses and phone numbers anyway — information that did not come from the tool.

Lowering the sampling temperature did not fix it. Temperature controls randomness in token selection, not whether the model grounds its answer in the tool output. The cause is upstream: after a tool call, the prompt asks the model for a helpful, well-formatted response, and with an empty result it fills the gap from prior knowledge.

Two fixes worth trying:

1. Handle the empty-result case in code, returning a fixed message to the user without a second model call.
2. Add an explicit instruction that when a tool returns no data, the agent must say so and stop.

## Running it

Developed against a self-hosted Ollama instance running on a university GPU server, so the project uses no paid API services. Any Ollama host serving `llama3.2` works. Set the endpoint at the top of the notebook:

```python
OLLAMA_BASE_URL = "http://localhost:11434"   # or your own Ollama host
OLLAMA_MODEL    = "llama3.2"
```

```bash
pip install requests duckduckgo-search gradio
jupyter notebook health_wellness_final.ipynb
```

Run the cells in order. The Gradio interface launches from the final cell.

## Stack

Python · LLaMA 3.2 · Ollama · Gradio · OpenFDA API · OpenStreetMap (Nominatim, Overpass) · DuckDuckGo Search

---

Informational project only. Not medical advice.
