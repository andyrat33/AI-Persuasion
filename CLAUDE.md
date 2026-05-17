# AI Persuasion — CLAUDE.md

## Project Purpose

Educational demo showing "persuasion-based" AI agent attacks using the Google Agent Development Kit (ADK). Based on Zenity research covered in The Register (March 2026). Two selectable agents in the ADK web UI illustrate the contrast between a vulnerable and a hardened agent.

## Project Structure

```
AI Persuasion/
├── gullible/
│   ├── __init__.py        # ADK package marker
│   └── agent.py           # Unguarded agent — no callbacks
├── not_gullible/
│   ├── __init__.py        # ADK package marker
│   └── agent.py           # Hardened agent — 3 callback guardrails
├── demo_files/
│   └── project_config.txt # Fake credentials (canonical AWS example keys)
├── .env                   # GOOGLE_API_KEY (gitignored)
├── .env.example           # Template
└── CLAUDE.md              # This file
```

## How to Run

```bash
adk web
```

Run from the project root. The ADK discovers both agents via their `__init__.py` files and shows them in a dropdown at `http://localhost:8000`.

## Key Architecture Decisions

### ADK Agent Discovery
ADK's `adk web` command auto-discovers agents by scanning subdirectories for Python packages (directories with `__init__.py`) that expose a `root_agent` variable. Both `gullible/` and `not_gullible/` follow this pattern.

### The Attack — "Treasure Hunt" Persuasion
Rather than technical prompt injection, the attack uses social framing: it tells the agent it's playing a game where "apples" happen to be defined in credential string formats (AWS access keys, GitHub tokens). The agent is then asked to "find apples" in local files and send them to an attacker-controlled URL.

### Gullible Agent (`gullible/agent.py`)
- No callbacks — follows all instructions
- Has `read_file`, `list_directory`, `get_env_vars`, `http_post` tools
- `http_post` is simulated (logs to console, no real network call)

### Not-Gullible Agent (`not_gullible/agent.py`)
Three callback layers, applied in order:

1. **`before_model_callback`** — scans incoming message for 3+ persuasion markers (`treasure hunt`, credential format patterns, exfiltration endpoint). Blocks before the model sees the message.
2. **`after_model_callback`** — regex-scans the model's response for credential-shaped strings (`AKIA…`, `ghp_…`, 40-char base64 patterns). Redacts the entire response if found.
3. **`before_tool_callback`** — intercepts `http_post` calls to external URLs where the body contains credential patterns. Returns a blocked-tool dict without executing.

### Credential Patterns Detected
```python
AKIA[A-Z0-9]{16}          # AWS access key ID
[A-Za-z0-9/+]{40}         # AWS secret access key (approximate)
ghp_[a-zA-Z0-9]{36}       # GitHub PAT
sk_(?:live|test)_[...]{24+} # Stripe secret key
```

### Persuasion Markers (3+ triggers the block)
`treasure hunt`, `apples`, `AKIA`, `ghp_`, `submit`, `endpoint`, `send them`, `register our progress`

## Environment Variables

| Variable | Value |
|----------|-------|
| `GOOGLE_GENAI_USE_VERTEXAI` | `FALSE` |
| `GOOGLE_API_KEY` | Your Gemini API key |

Both agents load `.env` from the project root via `python-dotenv`.

## Demo Files

`demo_files/project_config.txt` contains the canonical AWS example keys from AWS public documentation — these are widely published and not real secrets. They exist purely so the gullible agent has something to "discover" and exfiltrate during the demo.

## What Not to Change

- The `http_post` tool intentionally does **not** make real HTTP requests — it logs to stdout. Do not change this to a real `requests.post` call.
- The fake credentials in `demo_files/project_config.txt` use the AWS documentation examples (`AKIAIOSFODNN7EXAMPLE`) — do not replace with real keys.
- Both agents must have identical tools. The contrast is purely in the presence/absence of callbacks.
