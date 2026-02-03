# Project Specifications

> **Vision:** nanobot is an ultra-lightweight personal AI assistant framework (~4,000 LOC) that delivers core agent functionality — tool calling, multi-channel messaging, persistent memory, and scheduled tasks — in a clean, research-ready codebase 99% smaller than comparable frameworks.

---

## Architecture Decisions

### [2025-02-01] Async-First Event-Driven Architecture

**REQ-IDs:** All

**Context:** Need a framework that handles concurrent channels, tool execution, and scheduling without blocking.

**Decision:** Full async/await architecture with asyncio.Queue-based message bus decoupling channels from agent processing.

**Alternatives:**
- Threaded architecture: More complex, harder to reason about shared state
- Synchronous: Can't handle concurrent channels or background tasks

**Consequences:**
- All components are async-compatible
- Message bus provides clean separation between channels and agent
- Subagents run as in-process async tasks (not distributed)

---

### [2025-02-01] LiteLLM as Unified LLM Provider

**REQ-IDs:** FR-CORE-001

**Context:** Need to support multiple LLM providers (OpenRouter, Anthropic, OpenAI, Gemini, Zhipu, vLLM) without per-provider code.

**Decision:** Use LiteLLM as the unified provider abstraction layer with automatic provider detection from API key prefix or api_base.

**Alternatives:**
- Direct provider SDKs: More code, harder to maintain, provider-specific bugs
- Custom abstraction: Reinventing what LiteLLM already does

**Consequences:**
- Single implementation handles all providers
- Tool calling works across providers
- No streaming support currently (LiteLLM supports it, not yet integrated)

---

### [2025-02-01] Markdown-Based Skill System

**REQ-IDs:** FR-CORE-003

**Context:** Need extensible agent capabilities without complex plugin architecture.

**Decision:** Skills are markdown files with YAML frontmatter. Progressive loading: always-loaded skills get full content in system prompt, others get summaries and are loaded on-demand via read_file.

**Alternatives:**
- Python plugin system: More powerful but heavier, harder for non-developers
- JSON config: Less expressive, no inline instructions

**Consequences:**
- Easy to create and share skills (just markdown)
- Workspace skills override built-in skills
- Requirement checking for external dependencies (bins, env vars)
- Limited to instruction-based skills (no Python hooks)

---

### [2025-02-01] JSONL Session Storage

**REQ-IDs:** FR-CORE-002

**Context:** Need persistent conversation history that's human-readable and appendable.

**Decision:** Store sessions as JSONL files (one JSON object per line) in ~/.nanobot/sessions/. First line is metadata, following lines are messages.

**Alternatives:**
- SQLite: More query power, but overkill for personal assistant
- Full JSON: Requires rewriting entire file on each message

**Consequences:**
- Efficient append-only writes
- Human-readable with standard tools
- No encryption (plaintext storage)
- Max 50 messages loaded into LLM context

---

### [2025-02-01] Node.js Bridge for WhatsApp

**REQ-IDs:** FR-CHAN-002

**Context:** WhatsApp has no official bot API. Need to use WhatsApp Web protocol.

**Decision:** Separate Node.js bridge process using @whiskeysockets/baileys, connected to Python via WebSocket.

**Alternatives:**
- Pure Python: No mature WhatsApp Web library in Python
- Headless browser: Heavy, fragile, slow

**Consequences:**
- Requires Node.js >= 18 for WhatsApp support
- Bridge must run as separate process
- WebSocket provides clean async communication
- Auto-reconnection with backoff

---

## Technical Stack

**Language:** Python 3.11+
**CLI:** Typer + Rich
**LLM:** LiteLLM (OpenRouter, Anthropic, OpenAI, Gemini, Zhipu, vLLM)
**Validation:** Pydantic 2.0+
**HTTP:** httpx (async)
**Messaging:** python-telegram-bot, websockets (WhatsApp bridge)
**Scheduling:** croniter
**Web Extraction:** readability-lxml
**Logging:** loguru
**Build:** Hatchling

---

## Conventions

### Code Style
- Line length: 100 (ruff)
- Linter: ruff (E, F, I, N, W rules)
- Target: Python 3.11
- File naming: snake_case
- Folder structure: Feature-based (`nanobot/agent/`, `nanobot/channels/`, etc.)
- Functions: snake_case, verb-first (`load_config`, `build_system_prompt`)
- Classes: PascalCase (`AgentLoop`, `ToolRegistry`, `SessionManager`)
- Async: All I/O-bound operations use async/await

### Git Workflow
- Branches: `feature/description` or `fix/description`
- Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:` prefix
- PRs: Squash and merge to main

### Configuration
- File: `~/.nanobot/config.json` (camelCase keys)
- Python: snake_case via Pydantic conversion
- Env overrides: `NANOBOT_AGENTS__DEFAULTS__MODEL` (double underscore nesting)

### Project Layout
```
nanobot/
├── agent/          # Core agent logic (loop, context, memory, skills, subagent)
│   └── tools/      # Built-in tools (filesystem, shell, web, message, spawn)
├── bus/            # Message routing (events, queue)
├── channels/       # Chat platforms (base, telegram, whatsapp, manager)
├── cli/            # CLI commands
├── config/         # Configuration (schema, loader)
├── cron/           # Scheduled tasks (types, service)
├── heartbeat/      # Periodic wake-up
├── providers/      # LLM providers (base, litellm)
├── session/        # Conversation sessions
├── skills/         # Built-in skills (github, weather, summarize, tmux, skill-creator)
└── utils/          # Helpers
```

---

## Agent Loading Strategy

**Agents should load:**
- This file once at session start
- Only relevant sections as needed
- Never load entire specs.md repeatedly

**Context budget:**
- Full specs.md: ~1.5k tokens
- Single decision: ~200 tokens
- Conventions section: ~400 tokens

---

*Last updated: 2026-02-03*
