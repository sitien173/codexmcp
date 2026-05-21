# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## Project Overview

CodexMCP is a FastMCP server that wraps the OpenAI Codex CLI, exposing it as an MCP tool for agentic workflows. It allows MCP clients (like Claude Code) to invoke `codex exec` non-interactively and receive structured JSON results.

## Commands

```bash
# Install the project (editable mode, using uv)
uv sync

# Run the MCP server (stdio transport)
uv run codexmcp

# Run the server directly via Python
uv run python -m codexmcp.cli

# Build the package
uv build
```

There is no test suite in this repository. There are no linting or type-checking commands configured.

## Architecture

The entire server is a single-tool MCP server with three source files under `src/codexmcp/`:

- **`server.py`** — Core logic. Contains the `FastMCP` instance, the `codex` tool definition, and two key helpers:
  - `run_shell_command(cmd)` — Spawns `codex exec` as a subprocess, streams its JSON output via a background thread and a `queue.Queue`, and yields parsed lines. It detects `turn.completed` events to gracefully terminate the Codex process.
  - `windows_escape(prompt)` — Escapes special characters in prompts for Windows command-line compatibility.
  - The `codex` tool is an `async` function that builds the CLI command, streams output through `run_shell_command`, parses JSON event lines to extract `agent_message` text and `thread_id`, and returns `{"success": True, "SESSION_ID": ..., "agent_messages": ...}` or `{"success": False, "error": ...}`.

- **`cli.py`** — Entry point registered as the `codexmcp` console script. Calls `server.run()`.

- **`__init__.py`** — Package marker with `__version__`.

The Codex CLI is invoked with `--json --yolo --skip-git-repo-check` flags. Session resumption is supported via the `resume` subcommand with a `SESSION_ID`. The `model` and `profile` parameters are passed through only when explicitly provided.

## Key Design Decisions

- **Subprocess-based**: The server does not use a Codex SDK/API. It shells out to the `codex` CLI binary, which must be installed and available on `PATH`.
- **Streaming output**: Codex emits newline-delimited JSON events. The server reads these in a background thread, puts them on a queue, and the main coroutine yields them. This avoids blocking the async event loop.
- **Graceful shutdown**: When a `turn.completed` event is detected, the server waits 0.3s then terminates the Codex process, rather than waiting for it to exit naturally.
- **Windows handling**: On Windows (`os.name == "nt"`), prompts are escaped via `windows_escape()` before being passed as a CLI argument.

## Dependencies

- `mcp[cli]>=1.20.0` — FastMCP framework and CLI utilities
- `pydantic>=2.0` — Parameter validation and type annotations
- Python >=3.12

Package management uses `uv` with `pyproject.toml` (hatchling build backend).

## MCP Server Configuration

### Prerequisites

- Python >=3.12
- [uv](https://docs.astral.sh/uv/) package manager
- [Codex CLI](https://github.com/openai/codex) installed and available on `PATH` (run `codex --version` to verify)

### Installation Methods

**Using `uvx` (recommended, no clone required):**

```bash
claude mcp add codex -s user --transport stdio -- uvx --from git+https://github.com/GuDaStudio/codexmcp.git codexmcp
```

**Using local editable install:**

```bash
git clone https://github.com/GuDaStudio/codexmcp.git
cd codexmcp
uv sync
claude mcp add codex -s user --transport stdio -- uv run --directory /path/to/codexmcp codexmcp
```

### Tool: `codex`

Executes a non-interactive Codex CLI session for AI-assisted coding tasks.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `PROMPT` | `str` | Yes | The task instruction to send to Codex. |
| `cd` | `str` (path) | Yes | Workspace root directory for Codex execution. |
| `SESSION_ID` | `str` | No | Resume an existing session. Omit or pass `""` to start a new session. |
| `image` | `list[str]` | No | Image file paths to attach to the prompt. |
| `model` | `str` | No | Model override for the session. **Only use when the user explicitly requests it.** |
| `profile` | `str` | No | Config profile from `~/.codex/config.toml`. **Only use when the user explicitly requests it.** |

**Return value (on success):**

```json
{
  "success": true,
  "SESSION_ID": "thread-uuid-here",
  "agent_messages": "Concatenated text output from Codex agent..."
}
```

**Return value (on failure):**

```json
{
  "success": false,
  "error": "Description of what went wrong..."
}
```

### Session Continuation

On a successful call, the response includes `SESSION_ID`. To continue the same conversation, pass that `SESSION_ID` on the next call. This enables iterative workflows where Codex retains context across multiple tool invocations.

### How It Works Internally

1. The server builds a `codex exec` command with `--json --yolo --skip-git-repo-check` flags.
2. Codex runs as a subprocess, emitting newline-delimited JSON events.
3. A background thread streams stdout lines onto a queue; the main coroutine yields them.
4. The server parses each line for `agent_message` items (accumulating text) and captures `thread_id`.
5. When a `turn.completed` event arrives, the server gracefully terminates the Codex process.
6. The accumulated `agent_messages` and `SESSION_ID` are returned to the caller.
