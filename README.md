---
title: deep_research
app_file: app.py
sdk: gradio
sdk_version: 6.14.0
---

# Deep Research Agent

A multi-agent research app that takes a question, plans several web searches, gathers summaries, writes a long-form markdown report, and delivers it by email (or Pushover). The UI is a Gradio app (`app.py`).

This project uses the **OpenAI Agents SDK** (`openai-agents`) to orchestrate specialized agents, with **OpenAI web search** as the research tool.

## How it works

`ResearchManager` runs a traced pipeline:

1. **Plan** — Planner agent outputs a structured list of search terms (`WebSearchPlan`).
2. **Search** — Each term is searched in parallel by the Search agent (`WebSearchTool`).
3. **Write** — Writer agent turns the original query plus summaries into a detailed report (`ReportData`).
4. **Deliver** — Email agent converts the report to HTML and sends it (SMTP) or a Pushover notification.

Status messages stream into the UI while the pipeline runs. The last yield is the markdown report.

```
Query → Planner → parallel Search agents → Writer → Email / Pushover
                      ↓
              Gradio status + report
```

## Agents and tools

| Agent | Role | Tools / output |
| --- | --- | --- |
| **Planner** (`planner_agent.py`) | Chooses search terms and why each one matters | Structured `WebSearchPlan` (Pydantic) |
| **Search** (`search_agent.py`) | Looks up a term and returns a short summary | OpenAI `WebSearchTool` (tool use required) |
| **Writer** (`writer_agent.py`) | Produces a long markdown report | `ReportData`: short summary, markdown, follow-up questions |
| **Email** (`email_agent.py`) | Formats and sends the report | `send_email_tool` → SMTP or Pushover |

Orchestration uses `Runner.run`, `trace`, and `gen_trace_id` from `openai-agents`. Traces open in the [OpenAI traces UI](https://platform.openai.com/traces).

## Tech stack

- **Python** 3.12+
- **[openai-agents](https://github.com/openai/openai-agents-python)** — agents, runner, tracing, `WebSearchTool`, `function_tool`
- **OpenAI API** — chat models (default `gpt-5.4-mini` via `DEFAULT_MODEL_NAME`)
- **Gradio** — web UI (`app.py`; simpler variant in `simple.py`)
- **Pydantic** — structured outputs for plans and reports
- **python-dotenv** — environment loading
- **requests** — Pushover HTTP API
- **smtplib** — email delivery

Packaging is via `pyproject.toml` / `uv.lock`. Dependencies are also listed in `requirements.txt`.

## Project layout

| File | Purpose |
| --- | --- |
| `app.py` | Main Gradio UI (custom CSS/JS from `styles.py`) |
| `simple.py` | Minimal Gradio UI without custom styling |
| `research_manager.py` | Pipeline orchestration |
| `planner_agent.py` | Search-plan agent |
| `search_agent.py` | Web-search agent |
| `writer_agent.py` | Report-writing agent |
| `email_agent.py` | Delivery agent |
| `messenger.py` | SMTP email and Pushover helpers |
| `styles.py` | Header, example queries, CSS, and focus JS |

## Setup

1. Clone the repo and create a virtual environment (Python 3.12+).

```bash
uv sync
# or: pip install -r requirements.txt
```

2. Create a `.env` file in the project root (this file is gitignored).

| Variable | Purpose |
| --- | --- |
| `OPENAI_API_KEY` | OpenAI API key (required for agents and web search) |
| `DEFAULT_MODEL_NAME` | Model id (default `gpt-5.4-mini`) |
| `HOW_MANY_SEARCHES` | Number of planned searches (default `5`) |
| `USE_EMAIL` | `true` to send SMTP email; otherwise Pushover (`true` by default) |
| `EMAIL_ADDRESS` | From/to address for the report |
| `EMAIL_SMTP_SERVER` | SMTP host (STARTTLS on port 587) |
| `EMAIL_APP_PASSWORD` | SMTP / app password |
| `PUSHOVER_USER` | Pushover user key (used when `USE_EMAIL` is not true) |
| `PUSHOVER_TOKEN` | Pushover API token |

## Run

```bash
python app.py
```

Open the Gradio URL in the browser, enter a research question (or pick an example), and click **Investigate**. Progress updates appear first; the finished markdown report follows after email/push delivery.

A plainer UI:

```bash
python simple.py
```

## Notes

- Search calls run concurrently with `asyncio.gather`.
- The writer is instructed to produce a long, detailed markdown report (on the order of 5–10 pages / 1000+ words).
- Email is sent to `EMAIL_ADDRESS` (same address for from and to).
- Hugging Face Spaces can use the YAML front matter at the top of this file (`app_file: app.py`).
