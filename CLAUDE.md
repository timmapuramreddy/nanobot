# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**nanobot** is an ultra-lightweight personal AI assistant (~3,700 lines of core code). Published as `nanobot-ai` on PyPI. Python >= 3.11, built with **hatchling**.

## Common Commands

```bash
# Install (development)
pip install -e .

# Lint & format
ruff check .
ruff format .

# Run all tests
pytest

# Run a single test
pytest tests/test_commands.py

# Build
python -m build

# Docker
docker build -t nanobot .
docker compose up -d nanobot-gateway
```

Ruff config: line length 100, target Python 3.11, rules E/F/I/N/W (E501 ignored). Tests use `asyncio_mode = "auto"`.

## Architecture

### Data Flow

```
Channel (Telegram/Discord/etc.) → MessageBus.inbound (asyncio.Queue)
  → AgentLoop._process_message() → ContextBuilder.build_messages()
  → LLMProvider.chat() → ToolRegistry.execute() (loops until no tool calls)
  → SessionManager.save() → MessageBus.outbound → Channel.send()
```

### Key Components

- **AgentLoop** (`nanobot/agent/loop.py`): Central controller. Loads session, builds prompt, runs LLM in a tool-call loop, triggers memory consolidation when session exceeds `memory_window` (default 50 messages).
- **MessageBus** (`nanobot/bus/queue.py`): Decouples channels from agent via two async queues (inbound/outbound).
- **ContextBuilder** (`nanobot/agent/context.py`): Assembles system prompt from workspace files (AGENTS.md, SOUL.md, USER.md, TOOLS.md), memory, and skill summaries.
- **ToolRegistry** (`nanobot/agent/tools/registry.py`): Dynamic tool registration and execution. Tools return strings (errors prefixed with `"Error: ..."`).
- **SessionManager** (`nanobot/session/manager.py`): JSONL storage at `<workspace>/sessions/<channel>_<chat_id>.jsonl`. Session keys: `"channel:chat_id"`.
- **MemoryStore** (`nanobot/agent/memory.py`): Two-layer — `memory/MEMORY.md` (LLM-rewritten facts) + `memory/HISTORY.md` (append-only log).
- **SkillsLoader** (`nanobot/agent/skills.py`): Loads SKILL.md files with YAML frontmatter. Workspace skills override built-ins by name.
- **SubagentManager** (`nanobot/agent/subagent.py`): Spawns background asyncio tasks with independent tool registries (no message/spawn tools), max 15 iterations.
- **ProviderSpec registry** (`nanobot/providers/registry.py`): Single source of truth for all LLM providers. Adding a provider = add entry here + config field in `config/schema.py`.

### Channels

All in `nanobot/channels/`, inherit `BaseChannel` ABC. Implement `start()`, `stop()`, `send()`, `is_running`. Channels: telegram, discord, whatsapp (Node.js bridge via WebSocket), feishu, dingtalk, slack, qq, email, mochat.

### Config

File: `~/.nanobot/config.json`. Pydantic with `alias_generator=to_camel` (both camelCase and snake_case accepted). Env vars: `NANOBOT_` prefix, `__` as nested delimiter.

## Adding New Components

**New Provider**: (1) Add `ProviderSpec` to `PROVIDERS` in `nanobot/providers/registry.py`, (2) Add field to `ProvidersConfig` in `nanobot/config/schema.py`. Nothing else needed.

**New Tool**: (1) Create class inheriting `Tool` ABC in `nanobot/agent/tools/`, (2) Implement `name`, `description`, `parameters` properties and `async execute(**kwargs) -> str`, (3) Register in `AgentLoop._register_default_tools()` in `nanobot/agent/loop.py`.

**New Channel**: (1) Create `nanobot/channels/<name>.py` inheriting `BaseChannel`, (2) Implement `start()`, `stop()`, `send()`, `is_running`, (3) Add config to `ChannelsConfig` in `nanobot/config/schema.py`, (4) Register in `ChannelManager._init_channels()`.

## Key Patterns

- **Tool results** are always strings; errors start with `"Error: ..."`
- **MCP tools** registered as `mcp_{server_name}_{tool_name}` to avoid collisions
- **LiteLLMProvider** auto-prefixes model names (e.g., `deepseek-chat` → `deepseek/deepseek-chat`) based on registry
- **ExecTool** blocks dangerous shell patterns via regex
- `restrictToWorkspace: true` sandboxes file/shell tools to workspace directory
- Skill frontmatter: `metadata: '{"nanobot": {"always": false, "requires": {"bins": ["gh"]}}}'`
- WhatsApp channel connects to a Node.js bridge in `bridge/` via WebSocket
