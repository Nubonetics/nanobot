# Requirements Registry

> **Anti-Ghost Rule:** No code without a REQ-ID listed here

---

## Status Legend

- `[ ]` Pending
- `[~]` In Progress
- `[x]` Done
- `[!]` Blocked
- `[?]` Needs Clarification
- `[L]` Legacy (pre-SRT²)

---

## Core Agent (Legacy)

**FR-CORE-001** `[L]` LLM-powered agent loop with tool calling

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Foundation of the assistant — processes messages, calls LLMs, executes tools in a loop

**Acceptance Criteria:**
- Agent consumes inbound messages from bus
- Builds context (system prompt, history, memory, skills)
- Calls LLM provider with tool definitions
- Executes tool calls and feeds results back
- Max 20 iterations per message

**Test:** `TEST-CORE-001`

**Commits:** pre-SRT²

---

**FR-CORE-002** `[L]` Session persistence with conversation history

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Users expect continuity across conversations

**Acceptance Criteria:**
- Sessions stored as JSONL in ~/.nanobot/sessions/
- First line: metadata, following lines: messages
- Get/create sessions by key (channel:chat_id)
- Load last 50 messages for LLM context

**Test:** `TEST-CORE-002`

**Commits:** pre-SRT²

---

**FR-CORE-003** `[L]` Markdown-based extensible skill system

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Agents need domain-specific capabilities without code changes

**Acceptance Criteria:**
- Skills defined as SKILL.md with YAML frontmatter
- Progressive loading (always-loaded vs on-demand summary)
- Workspace skills override built-in skills
- Requirement checking (bins, env vars)
- 5 built-in skills: github, weather, summarize, tmux, skill-creator

**Test:** `TEST-CORE-003`

**Commits:** pre-SRT²

---

**FR-CORE-004** `[L]` Persistent memory (long-term + daily notes)

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Assistant needs to remember important information across sessions

**Acceptance Criteria:**
- Long-term memory: MEMORY.md (persistent)
- Daily notes: memory/YYYY-MM-DD.md (timestamped)
- Recent memories (last N days) included in context
- Read/write/append operations

**Test:** `TEST-CORE-004`

**Commits:** pre-SRT²

---

**FR-CORE-005** `[L]` Background subagent execution

**Owner:** -
**Branch:** `main`
**Dependencies:** FR-CORE-001

**Why:** Long-running tasks shouldn't block the main agent conversation

**Acceptance Criteria:**
- Spawn tool creates background async tasks
- Subagents run with limited tools (no message, no spawn)
- Results announced back via "system" channel
- Max 15 iterations per subagent

**Test:** `TEST-CORE-005`

**Commits:** pre-SRT²

---

## Tools (Legacy)

**FR-TOOL-001** `[L]` File operations (read, write, edit, list_dir)

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Agent needs to manage files in the workspace

**Acceptance Criteria:**
- read_file: Read file contents, expand ~
- write_file: Create/overwrite files, create parent dirs
- edit_file: Replace text patterns, warn on multiple matches
- list_dir: Pretty directory listing

**Test:** `TEST-TOOL-001`

**Commits:** pre-SRT²

---

**FR-TOOL-002** `[L]` Shell command execution

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Agent needs to run system commands (git, npm, etc.)

**Acceptance Criteria:**
- Execute shell commands with 60s timeout
- Capture stdout and stderr
- Truncate output at 10,000 characters
- Return exit code information

**Test:** `TEST-TOOL-002`

**Commits:** pre-SRT²

---

**FR-TOOL-003** `[L]` Web access (search + fetch)

**Owner:** -
**Branch:** `main`
**Dependencies:** None

**Why:** Agent needs internet access for research and information retrieval

**Acceptance Criteria:**
- web_search: Brave Search API, configurable results (1-10)
- web_fetch: URL fetching with readability extraction
- Output truncation for large pages

**Test:** `TEST-TOOL-003`

**Commits:** pre-SRT²

---

## Channels (Legacy)

**FR-CHAN-001** `[L]` Telegram channel with media support

**Owner:** -
**Branch:** `main`
**Dependencies:** FR-CORE-001

**Why:** Telegram is the primary chat interface for nanobot

**Acceptance Criteria:**
- Long polling (no webhook needed)
- Text, photo, voice, audio, document handling
- Media downloads to ~/.nanobot/media
- Allow-list filtering by user ID or username
- Markdown-to-Telegram-HTML conversion

**Test:** `TEST-CHAN-001`

**Commits:** pre-SRT²

---

**FR-CHAN-002** `[L]` WhatsApp channel via Node.js bridge

**Owner:** -
**Branch:** `main`
**Dependencies:** FR-CORE-001

**Why:** WhatsApp is widely used for personal messaging

**Acceptance Criteria:**
- WebSocket connection to Node.js bridge
- Message send/receive
- QR code authentication flow
- Auto-reconnection with 5s backoff
- Allow-list filtering by phone number

**Test:** `TEST-CHAN-002`

**Commits:** pre-SRT²

---

## Scheduling (Legacy)

**FR-CRON-001** `[L]` Cron-based job scheduling

**Owner:** -
**Branch:** `main`
**Dependencies:** FR-CORE-001

**Why:** Users need scheduled reminders and recurring tasks

**Acceptance Criteria:**
- Three schedule types: at (one-time), every (interval), cron (expression)
- Persistent JSON storage
- One-shot jobs auto-delete or disable after run
- Recurring jobs recompute next run
- CLI management: add, list, remove, enable, disable, run

**Test:** `TEST-CRON-001`

**Commits:** pre-SRT²

---

**FR-CRON-002** `[L]` Heartbeat periodic check

**Owner:** -
**Branch:** `main`
**Dependencies:** FR-CORE-001

**Why:** Agent should proactively handle recurring tasks without user prompting

**Acceptance Criteria:**
- Check HEARTBEAT.md every 30 minutes
- Skip if file empty (only headers/comments)
- Execute tasks through agent processing
- HEARTBEAT_OK response means nothing to do

**Test:** `TEST-CRON-002`

**Commits:** pre-SRT²

---

## Test Infrastructure

**FR-TEST-001** `[ ]` Unit test suite for core components

**Owner:** Unassigned
**Branch:** -
**Dependencies:** None

**Why:** Zero test coverage currently; need baseline confidence for all future changes

**Acceptance Criteria:**
- pytest + pytest-asyncio test infrastructure
- Unit tests for ToolRegistry (register, execute, definitions)
- Unit tests for MessageBus (publish, consume, subscribe)
- Unit tests for SessionManager (create, save, load, delete)
- Unit tests for MemoryStore (read, write, append, recent)
- Unit tests for ConfigLoader (load, save, env overrides)
- Unit tests for CronService (add, remove, next_run computation)
- Coverage report generation
- All tests pass in CI

**Test:** `TEST-TEST-001`

**Commits:** -

---

**FR-TEST-002** `[ ]` Integration tests for agent loop

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-TEST-001

**Why:** Need to verify end-to-end message flow through the agent

**Acceptance Criteria:**
- Mock LLM provider for deterministic responses
- Test: message in → tool call → tool result → response out
- Test: multi-turn tool calling (up to max iterations)
- Test: context building with memory and skills
- Test: subagent spawn and result announcement

**Test:** `TEST-TEST-002`

**Commits:** -

---

## Streaming & Performance

**FR-PERF-001** `[ ]` LLM response streaming

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001

**Why:** Long responses feel slow without streaming; users see nothing until full response completes

**Acceptance Criteria:**
- LiteLLM streaming mode enabled
- Tokens streamed to channel as they arrive
- Tool calls still batched (stream text only)
- Works with Telegram and WhatsApp channels
- Fallback to non-streaming if provider doesn't support it

**Test:** `TEST-PERF-001`

**Commits:** -

---

**FR-PERF-002** `[ ]` API rate limiting and retry logic

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001

**Why:** API calls can fail due to rate limits; need graceful handling

**Acceptance Criteria:**
- Exponential backoff on 429/rate-limit responses
- Max 3 retries per LLM call
- Configurable rate limits per provider
- User notification on persistent failures

**Test:** `TEST-PERF-002`

**Commits:** -

---

## Multi-Modal

**FR-MODAL-001** `[ ]` Enhanced vision support (image analysis)

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001, FR-CHAN-001

**Why:** Users send images via chat and expect the assistant to understand them

**Acceptance Criteria:**
- Image attachments included in LLM context as base64
- Support JPEG, PNG, GIF, WebP
- Works with vision-capable models (GPT-4o, Claude, Gemini)
- Graceful fallback for non-vision models
- Telegram: auto-download photos and include in context

**Test:** `TEST-MODAL-001`

**Commits:** -

---

**FR-MODAL-002** `[ ]` Voice message transcription

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CHAN-001

**Why:** Voice messages are common on mobile; assistant should understand them

**Acceptance Criteria:**
- Transcribe voice messages to text using Whisper API or equivalent
- Include transcription in agent context
- Support Telegram voice messages and audio files
- Configurable transcription provider

**Test:** `TEST-MODAL-002`

**Commits:** -

---

## More Integrations

**FR-CHAN-003** `[ ]` Discord channel

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001

**Why:** Discord is widely used for communities and personal servers

**Acceptance Criteria:**
- Discord bot using discord.py
- Text message handling in DMs and designated channels
- Allow-list filtering by user ID
- Markdown formatting support
- Configuration in channels.discord section

**Test:** `TEST-CHAN-003`

**Commits:** -

---

**FR-CHAN-004** `[ ]` Slack channel

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001

**Why:** Slack is the primary tool for workplace communication

**Acceptance Criteria:**
- Slack app using Socket Mode (no public URL needed)
- Direct message and @mention handling
- Allow-list filtering by user ID
- Markdown/mrkdwn formatting
- Configuration in channels.slack section

**Test:** `TEST-CHAN-004`

**Commits:** -

---

## Memory & Reasoning

**FR-MEM-001** `[ ]` Configurable session history limits

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-002

**Why:** Hardcoded 50-message limit may be too low or too high depending on use case

**Acceptance Criteria:**
- Configurable max_messages in agents.defaults config
- Default remains 50
- Session cleanup for old sessions (configurable retention)
- Memory-aware truncation (summarize old messages instead of dropping)

**Test:** `TEST-MEM-001`

**Commits:** -

---

**FR-MEM-002** `[ ]` Semantic memory search

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-004

**Why:** Current memory is date-based; agent can't find specific memories efficiently

**Acceptance Criteria:**
- Vector embeddings for memory entries
- Semantic search across all memories
- Top-K relevant memories included in context
- Embedding provider configurable (OpenAI, local)

**Test:** `TEST-MEM-002`

**Commits:** -

---

## Robustness

**FR-ROB-001** `[ ]` Cron timezone support

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CRON-001

**Why:** Users in different timezones need reminders at their local time

**Acceptance Criteria:**
- Timezone field in cron job configuration
- Default to system timezone
- Cron expressions evaluated in specified timezone
- CLI displays times in user's timezone

**Test:** `TEST-ROB-001`

**Commits:** -

---

**FR-ROB-002** `[ ]` Graceful crash recovery

**Owner:** Unassigned
**Branch:** -
**Dependencies:** FR-CORE-001, FR-CRON-001

**Why:** In-memory state is lost on crash; messages and jobs may be lost

**Acceptance Criteria:**
- Outbound message queue persisted to disk
- Undelivered messages retried on restart
- Cron jobs resume correctly after crash
- Agent loop restarts cleanly after exception

**Test:** `TEST-ROB-002`

**Commits:** -

---

## Agent Notes

**Context Loading:**
- Agents load ONLY their assigned REQ-IDs
- Example: Agent Alpha loads only FR-TEST-001 (~500 tokens)
- Never load all requirements

**Ownership:**
- Each REQ-ID assigned to one agent
- Prevents merge conflicts
- Parallel execution enabled

---

*Total: 22 | Done: 0 | In Progress: 0 | Pending: 10 | Legacy: 12*
