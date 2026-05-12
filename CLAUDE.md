# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hermes Agent is a self-improving AI agent built by Nous Research. It runs an agent loop (LLM + tools), supports multi-platform messaging (Telegram, Discord, Slack, etc.), persists sessions with SQLite/FTS5, has a skill system for procedural memory, and a plugin system for extensibility. Python 3.11+, any OpenAI-compatible API.

Entry points (from `pyproject.toml` scripts):
- `hermes` → `hermes_cli.main:main` (interactive CLI)
- `hermes-agent` → `run_agent:main` (headless agent)
- `hermes-acp` → `acp_adapter.entry:main` (VS Code/JetBrains adapter)

## Common Commands

```bash
# Development environment
uv venv .venv --python 3.11 && source .venv/bin/activate
uv pip install -e ".[all,dev]"

# Run the full test suite (preferred — matches CI exactly)
scripts/run_tests.sh

# Run tests in a subdirectory or specific file
scripts/run_tests.sh tests/agent/
scripts/run_tests.sh tests/tools/test_file_operations.py::TestClass::test_method

# Type checking (exclude tinker-atropos submodule)
ty check

# Linting — only PLW1514 (explicit encoding) enforced; everything else is off
ruff check .

# Single pytest run (without hermetic CI env — use run_tests.sh for PRs)
pytest tests/ -v -m "not integration" -n 4

# Manual install for development
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
hermes doctor
```

## Architecture

### Core Loop (`run_agent.py`)
```
User message → AIAgent._run_agent_loop()
  ├── Build system prompt (agent/prompt_builder.py)
  ├── Build API kwargs (model, messages, tools)
  ├── Call LLM (OpenAI-compatible API)
  ├── If tool_calls → execute via registry → add results → loop
  ├── If text → persist to DB → return response
  └── Context compression if near token limit
```

### Self-Registering Tools (`tools/`)
Each `tools/*.py` file calls `registry.register()` at import time. `model_tools.py` imports `tools/registry.py` which auto-discovers all tool modules via `discover_builtin_tools()`. No manual import list.

**Adding a tool:** Create `tools/my_tool.py` with schema + handler + `registry.register(...)`. Then add the tool name to the appropriate list in `toolsets.py` (e.g. `_HERMES_CORE_TOOLS`) — the tool registers but won't be exposed to the agent without that step.

### Session Persistence (`hermes_state.py`)
SQLite with FTS5 full-text search. WAL mode by default; falls back to `journal_mode=DELETE` on NFS/SMB where WAL breaks. SessionDB chain: parent_session_id for compression-triggered splits. Batch runner and RL trajectories use separate storage.

### Provider Abstraction
Any OpenAI-compatible API. Provider resolution: Nous Portal OAuth, OpenRouter API key, or custom endpoint. `provider_routing` in config.yaml controls OpenRouter provider selection (throughput/latency/price/data retention) — injected as `extra_body.provider` in API requests.

### Messaging Gateway (`gateway/`)
`GatewayRunner` manages platform adapters (telegram, discord, slack, whatsapp, etc.). `gateway/session.py` is the session store for gateway sessions. Adding a new platform: copy `gateway/platforms/ADDING_A_PLATFORM.md` and follow the pattern.

### Skills System
- `skills/` — bundled skills (ship with every install, broadly useful)
- `optional-skills/` — official but niche (discoverable via hub, not activated by default)
- Skills Hub — community-contributed (upload to registry, install via `hermes skills install`)
- SKILL.md format: YAML frontmatter + Markdown body. Frontmatter controls platform restrictions, env var requirements, conditional activation.

**Skill vs Tool:** Prefer skill. Skill = instructions + shell commands + existing tools. Tool = requires custom API key management, binary data, real-time events, or precise custom logic.

### Plugins (`plugins/`)
Extensible via plugin interface. Categories: memory providers, context engines, inference backends, kanban, image gen, observability, etc. See `plugins/` directory for existing implementations.

## Cross-Platform Compatibility

Hermes runs on Linux, macOS, native Windows, and WSL2. All OS-specific code must be guarded.

**Critical Windows footguns (caught by `scripts/check-windows-footguns.py`):**
- `os.kill(pid, 0)` for liveness checks — maps to `CTRL_C_EVENT` on Windows and broadcasts to the entire process group. Use `psutil.pid_exists(pid)` instead (core dep).
- `os.setsid` / `os.killpg` / `os.fork` — POSIX-only. Guard with `hasattr(os, "setsid")` or `platform.system() != "Windows"`.
- `signal.SIGALRM`, `SIGCHLD`, `SIGHUP`, etc. — don't exist on Windows.
- `subprocess.Popen` with `.cmd`/`.bat` shims — use `shutil.which()` to resolve, not bare strings.
- Symlinks — need elevated privileges on Windows. Skip with `@pytest.mark.skipif(sys.platform == "win32", ...)`.
- Detached Windows daemons — use `pythonw.exe` + `CREATE_NO_WINDOW | DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP | CREATE_BREAKAWAY_FROM_JOB`.

Run `scripts/check-windows-footguns.py` on any diff touching process management, file I/O, or shell commands.

## User Config (all in `~/.hermes/`)

| Path | Purpose |
|------|---------|
| `config.yaml` | Settings (model, terminal, toolsets, compression, display, etc.) |
| `.env` | API keys and secrets |
| `auth.json` | OAuth credentials (Nous Portal) |
| `skills/` | All active skills (bundled + hub + agent-created) |
| `memories/` | Persistent memory (MEMORY.md, USER.md) |
| `state.db` | SQLite session database with FTS5 |
| `sessions/` | JSON session logs |
| `cron/` | Scheduled job data |
| `logs/` | `agent.log`, `errors.log`, `gateway.log` |

Profile-aware paths via `get_hermes_home()` in `hermes_constants.py`. Browse logs: `hermes logs [--follow] [--level ...] [--session ...]`.

## Security Notes

- Terminal access is unrestricted by default — dangerous command detection in `tools/approval.py` prompts for approval.
- Skills guard (`tools/skills_guard.py`) scans hub-installed skills for suspicious patterns.
- Cron prompt injection scanner in `tools/cronjob_tools.py` blocks instruction-override patterns.
- Write deny list resolves symlinks with `os.path.realpath()` before checking protected paths.
- Code execution (`execute_code`) strips API keys from the subprocess environment.
- Never log secrets — use `logger.warning/logger.error` with `exc_info=True` for unexpected errors.
