# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

`business-agent-lab` is a student's lab repository for a university course on business AI agents. Each week's material lives in its own folder (`week-05/`, ...). The root `README.md` holds the student's personal notes, not project documentation.

Currently present: `week-05/` — Tools and Function Calling. Its own `week-05/README.md` is the source of truth for setup, run commands, and the exercise.

## Environment

- Python 3.13 virtualenv at `.venv/` (project root). Activate with `source .venv/bin/activate`.
- Dependencies: `week-05/requirements.txt` (`requests`, `python-dotenv`). Install with `pip install -r week-05/requirements.txt`.
- The API key lives in `.env` at the project root as `OPENROUTER_API_KEY=...`. Optional `OPENROUTER_MODEL=...` overrides the model. Never print, log, or commit `.env`.
- `.gitignore` excludes `/.env`, `/.venv/`, and `workspaces.code-workspace`.
- No build system, linter, or test suite.

## Running (from the project root)

```sh
python week-05/01_model_request.py     # one raw request to the model
python week-05/02_calculator_call.py   # model requests a calculator call; press Enter to run it
python week-05/03_assistant.py         # interactive multi-tool assistant; type exit to quit
```

## week-05 architecture

- `model_api.py` — `request_model(messages, tools=None)` POSTs to the OpenRouter chat completions endpoint and returns `choices[0].message`. Retries up to 3 times on HTTP 429; raises `ModelRequestError` on failure. Default model: `nvidia/nemotron-3-super-120b-a12b:free`. Loads `.env` from one folder above the file, so scripts work from any working directory.
- `tools.py` — each tool is a pair: an OpenAI-style tool definition dict (`CALCULATOR`, `READ_SCHEDULE`, `SEARCH_WIKIPEDIA`, `SAVE_NOTE`) and a Python function with the same name as the definition's `"name"` (`calculator`, `read_schedule`, `search_wikipedia`, `save_note`). The model only sees definitions; the program runs the functions.
  - `read_schedule` reads `data/schedule.json` (fictional events, `YYYY-MM-DD` dates).
  - `search_wikipedia` calls the English Wikipedia REST search API (top 3 results).
  - `save_note` writes `notes/<sanitized-name>.md`, never overwriting (appends `-2`, `-3`, ...).
- `01_model_request.py` — writes the request out in full (endpoint, body, response) without `model_api.py`, for teaching.
- `02_calculator_call.py` — a single tool round with the calculator.
- `03_assistant.py` — a loop: send messages + `TOOLS`; if the reply has `tool_calls`, run each via `dispatch()` (looked up in `FUNCTIONS`) and append a `role: "tool"` message with the matching `tool_call_id`; repeat until a text answer or `MAX_ROUNDS = 5`. Tool errors are returned to the model as `"error: ..."` strings. Prints a trace (`tool call`, `tool result`, `assistant`).

### Adding a tool

A new tool needs three parts:
1. the function in `tools.py`
2. its tool definition in `tools.py`, with `"name"` matching the function name
3. registration in `03_assistant.py`: the definition in `TOOLS`, the function in `FUNCTIONS`

### Current exercise (not yet done)

Add currency conversion: `convert_currency(amount, from_currency, to_currency)` plus `CONVERT_CURRENCY` in `tools.py`, registered in `03_assistant.py`. Rate source is Frankfurter (no key): `GET https://api.frankfurter.dev/v2/rate/USD/KRW` → `{"date", "base", "quote", "rate"}`. Return the rate, its date, and `amount × rate`. The student must be able to explain the code, so keep it simple and in the style of the existing tools.

## Conventions

- Match the existing style: short module docstrings, plain functions, comments written for beginners.
- Keep tool results JSON-serializable; `dispatch()` converts non-strings with `json.dumps(..., ensure_ascii=False)`.
