# Mindful Policy Deep Agent

A local, free and open-source **agentic AI system** for public-policy decisions. You ask a policy question; a
[LangChain Deep Agent](https://github.com/langchain-ai/deepagents) researches free scholarly sources, and the app returns
**each option with its justification and its trade-offs**, plus the evidence it relied on, as **PDF, Markdown and JSON**.

Default topic: *how can contemplative practices (meditation) be offered at scale to improve public mental wellbeing?*
Any policy question works, but the safeguard checks are tuned to wellbeing programmes.

## Run it (Windows, macOS, Linux)

1. Install **Python 3.11+** (https://www.python.org/downloads/) and **Ollama** (https://ollama.com/download).
2. Double-click **`start.bat`** (Windows) or run **`./start.sh`** (macOS/Linux).
   First run creates a virtual environment, installs packages, downloads the `qwen3:8b` model (about 5 GB) and opens
   http://127.0.0.1:8000.

You do **not** need Node.js: the React UI is pre-built in `frontend/dist`. Node is only for editing the UI (see below).
Hardware: about 8 GB of free RAM for `qwen3:8b` with the default 16k context. A run takes roughly 10 to 30 minutes on a
laptop CPU, much less on a GPU.

Check your setup any time: `python scripts/check_setup.py`

## What you get

Per run, in `outputs/<run-id>/` and as downloads in the UI:

| File | Contents |
|---|---|
| `decision.pdf` | 3 to 4 page report: summary, recommendation, options at a glance, per-option justification vs trade-offs, evidence table, method |
| `decision.md` | Same content as Markdown |
| `decision.json` | Machine-readable: `report`, `evidence_sources`, `validation`, `provenance` (schema at `GET /api/schema`) |

Each option has exactly this shape:

```json
{ "id": "O1", "option": "Opt-in school elective ...", "description": "...",
  "justification": "... cites evidence ids like [E1] ...", "evidence_ids": ["E1", "E3"],
  "evidence_strength": "limited",
  "trade_offs": ["...", "..."], "indicative_cost": "... [C1]", "safeguards": ["..."] }
```

## How it works

```
React UI  ->  FastAPI  ->  Pipeline
                            1. Deep Agent (research lead)  -- plans, reads skills, delegates to subagents
                                 |- evidence-researcher  -> MCP server: Europe PMC, PubMed, Crossref, web, page reader, seed pack
                                 |- options-analyst      -> cost calculator, safeguard screen
                               every tool result is registered in the Evidence Registry (E1, E2, ... C1 ...)
                            2. Synthesis: local model writes option / justification / trade-off as schema-constrained JSON
                            3. Validation + bounded retries; citations the tools never returned are removed
                            4. Export PDF / Markdown / JSON  -> SQLite history
```

* **Agent framework and harness:** LangChain Deep Agents (planning todos, in-memory scratch filesystem, subagents, native Skills).
* **MCP:** `mcp_servers/research_server.py` is a real stdio MCP server; the agent loads it with `langchain-mcp-adapters`.
  If it cannot start, the app falls back to identical in-process tools and says so. Add more servers in `config/mcp_servers.json`.
* **Skills:** `backend/skills/*/SKILL.md` (7 skills), mounted read-only into the agent.
* **Evidence you can trace:** every source gets an id with provenance (tool, query, time, status). The report may only cite
  those ids. Status badges: *abstract read*, *page read*, *seed, unverified*, *calculation*.
* **Everything is free:** see the in-app page **What it is built from**, or `docs/stack.md`, for licences.

## Verification status (please read)

Verified in the build environment: 23 offline tests pass (real Deep Agent + real MCP server + real exporters, driven by a
scripted stand-in for the LLM); PDF pages were rendered and inspected; the React UI was run against the real API in a
browser with no console errors.

**Not verified here (no internet, no Ollama in the build sandbox):** a run with a real `qwen3:8b`, and live calls to Europe PMC,
PubMed, Crossref and DuckDuckGo. The Windows/macOS start scripts were not executed. Small local models are less reliable than
large ones: if a run fails validation, raise *Review retries*, or set a stronger tool-calling model in `.env` (`OLLAMA_MODEL`).

## Limits you should know

* The validator checks **structure and traceability**, not truth. A real citation can still be misread by a small model.
  Open the links; the report says which sources were only abstract-level.
* `data/curated_evidence/seed_evidence.json` entries are **unverified seeds** written from author knowledge. Confirm each DOI.
* Cost figures are **illustrative arithmetic** from stated assumptions (facilitator time scales with cohorts of 25 by default).
* DuckDuckGo access uses an unofficial library and may be rate-limited. Fetched web pages are treated as untrusted; the page
  reader blocks private network addresses (a determined DNS-rebinding attack is out of scope for a local tool).
* Research keywords leave your machine when search tools run. The model and your background text do not.
* Not clinical, legal or safeguarding advice. Meditation does not replace qualified mental-health care.

## Commands

```
python run.py                                  # start the app
python -m backend.cli --question "..."          # headless run, writes outputs/<run>/decision.*
python -m pytest -q                            # tests (offline)
python -m mcp_servers.research_server          # run the MCP server on its own (stdio)
```

Edit the UI: `cd frontend && npm install && npm run dev` (http://localhost:5173, proxies to the API), then
`npm run build` to refresh `frontend/dist`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Red "Ollama not running" at bottom left | Start Ollama, reload the page |
| "Model 'qwen3:8b' is not installed" | `ollama pull qwen3:8b` |
| Run ends with "could not produce a valid decision report" | Raise *Review retries*; or use a larger model; keep `OLLAMA_NUM_CTX` at 16384 or higher |
| Few or no sources | Check internet; the seed pack still works offline (marked unverified) |
| Port 8000 busy | Set `PORT=8010` in `.env` |
