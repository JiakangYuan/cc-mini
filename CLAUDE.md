# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mini Claude Code — a minimal Python replica of Claude Code. A terminal-based AI coding assistant powered by the Anthropic API with an interactive REPL, streaming responses, agentic tool loop, and a permission system.

## Build & Run Commands

```bash
# Install (editable, with dev deps)
pip install -e ".[dev]"

# Run interactive REPL
python3 -m mini_claude.main

# One-shot prompt
python3 -m mini_claude.main "your prompt here"

# Non-interactive (print and exit)
python3 -m mini_claude.main -p "your prompt here"

# Auto-approve all tool permissions
python3 -m mini_claude.main --auto-approve

# Run all tests
pytest tests/ -v

# Run a single test file
pytest tests/test_engine.py -v

# Run a single test
pytest tests/test_engine.py::test_function_name -v
```

## Architecture

The codebase follows a clean layered design:

- **`main.py`** — CLI entry point. Parses args via `argparse`, wires up components through `AppConfig`, runs either REPL (via `prompt_toolkit`) or one-shot mode. Uses `rich` for styled terminal output with spinner feedback. `run_query()` consumes the event iterator from Engine and handles Esc/Ctrl+C cancellation via `EscListener`.
- **`engine.py`** — Core agentic loop. `Engine.submit()` is a generator that streams API responses and yields `("text", str)`, `("tool_call", name, input)`, `("waiting",)`, and `("tool_result", name, input, result)` events. It loops until the model stops requesting tools. Supports `abort()` to cancel mid-turn (closes the HTTP stream) and `cancel_turn()` to roll back incomplete messages. Raises `AbortedError` on cancellation.
- **`config.py`** — Configuration system with layered precedence: CLI args > env vars (`MINI_CLAUDE_MODEL`, `MINI_CLAUDE_MAX_TOKENS`, `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`) > TOML config files (`~/.config/mini-claude/config.toml`, `.mini-claude.toml`) > defaults. Handles model alias resolution and per-model max token limits.
- **`context.py`** — Builds the system prompt dynamically: base instructions + date + working directory + git status (branch, status, recent commits) + contents of `CLAUDE.md` if present in cwd.
- **`_keylistener.py`** — Background ESC key listener using a daemon thread in cbreak mode. Provides `pause()`/`resume()` to avoid stealing keystrokes during permission prompts or text streaming.
- **`permissions.py`** — `PermissionChecker` auto-allows read-only tools, prompts user (y/n/a) in single-key cbreak mode for write tools and bash. Tracks per-session "always allow" set. Integrates with `EscListener` to detect ESC during permission prompts.
- **`tools/base.py`** — `Tool` ABC with `name`, `description`, `input_schema`, `execute()`, `is_read_only()`, and `to_api_schema()`. `ToolResult` dataclass holds `content` + `is_error`.
- **`tools/`** — Five tool implementations: `Read` (read-only), `Glob` (read-only), `Grep` (read-only), `Edit` (write), `Bash` (write). Read-only tools override `is_read_only() -> True`.

**Key data flow:** `main.py` creates tools + Engine → user input → `Engine.submit()` calls Anthropic streaming API → yields text chunks + tool calls → tools executed with permission checks → tool results sent back to API → loop until no more tool calls. ESC/Ctrl+C triggers `abort()` → stream closed → `AbortedError` raised → `cancel_turn()` removes incomplete messages.

## Key Technical Details

- Python 3.11+ required
- Default model: `claude-sonnet-4` (configurable via CLI `--model`, env `MINI_CLAUDE_MODEL`, or TOML config)
- Max tokens: per-model defaults in `config.py:_MODEL_MAX_TOKENS` (e.g. sonnet-4 → 64000, opus-4 → 32000), fallback 8192
- Model resolution supports aliases (e.g. `claude-opus-4.1` → `claude-opus-4-1`)
- Uses `hatchling` as build backend
- Dependencies: `anthropic>=0.40.0`, `prompt_toolkit>=3.0.0`, `rich>=13.0.0`
- Dev dependencies: `pytest>=8.0`, `pytest-asyncio>=0.23`
- REPL history saved to `~/.mini_claude_history`
- Config file paths: `~/.config/mini-claude/config.toml` (global), `.mini-claude.toml` (project)

## Adding a New Tool

1. Create `mini_claude/tools/your_tool.py` subclassing `Tool` from `tools/base.py`
2. Implement `name`, `description`, `input_schema`, `execute()`, and optionally `is_read_only()`
3. Register it in `main.py` in the `tools` list passed to `Engine`
