# Test Coverage Matrix

> **Rule:** Every REQ-ID must have test coverage

---

## Coverage Dashboard

**Overall:** 0% (0/0 passing)

| REQ-ID | Test ID | Type | Status | Owner | Location |
|--------|---------|------|--------|-------|----------|
| FR-CORE-001 | TEST-CORE-001 | Integration | :black_circle: Not Started | - | - |
| FR-CORE-002 | TEST-CORE-002 | Unit | :black_circle: Not Started | - | - |
| FR-CORE-003 | TEST-CORE-003 | Unit | :black_circle: Not Started | - | - |
| FR-CORE-004 | TEST-CORE-004 | Unit | :black_circle: Not Started | - | - |
| FR-CORE-005 | TEST-CORE-005 | Integration | :black_circle: Not Started | - | - |
| FR-TOOL-001 | TEST-TOOL-001 | Unit | :black_circle: Not Started | - | - |
| FR-TOOL-002 | TEST-TOOL-002 | Unit | :black_circle: Not Started | - | - |
| FR-TOOL-003 | TEST-TOOL-003 | Unit | :black_circle: Not Started | - | - |
| FR-CHAN-001 | TEST-CHAN-001 | Integration | :black_circle: Not Started | - | - |
| FR-CHAN-002 | TEST-CHAN-002 | Integration | :black_circle: Not Started | - | - |
| FR-CRON-001 | TEST-CRON-001 | Unit | :black_circle: Not Started | - | - |
| FR-CRON-002 | TEST-CRON-002 | Unit | :black_circle: Not Started | - | - |
| FR-TEST-001 | TEST-TEST-001 | Unit | :black_circle: Not Started | - | - |
| FR-TEST-002 | TEST-TEST-002 | Integration | :black_circle: Not Started | - | - |
| FR-PERF-001 | TEST-PERF-001 | Integration | :black_circle: Not Started | - | - |
| FR-PERF-002 | TEST-PERF-002 | Unit | :black_circle: Not Started | - | - |
| FR-MODAL-001 | TEST-MODAL-001 | Integration | :black_circle: Not Started | - | - |
| FR-MODAL-002 | TEST-MODAL-002 | Integration | :black_circle: Not Started | - | - |
| FR-CHAN-003 | TEST-CHAN-003 | Integration | :black_circle: Not Started | - | - |
| FR-CHAN-004 | TEST-CHAN-004 | Integration | :black_circle: Not Started | - | - |
| FR-MEM-001 | TEST-MEM-001 | Unit | :black_circle: Not Started | - | - |
| FR-MEM-002 | TEST-MEM-002 | Integration | :black_circle: Not Started | - | - |
| FR-ROB-001 | TEST-ROB-001 | Unit | :black_circle: Not Started | - | - |
| FR-ROB-002 | TEST-ROB-002 | Integration | :black_circle: Not Started | - | - |

**Status Icons:**
- :white_check_mark: Pass - All tests passing
- :yellow_circle: WIP - Tests written, some failing
- :red_circle: Fail - Tests exist, all failing
- :black_circle: Not Started - No tests yet

---

## Test Suites

### TEST-CORE-001: Agent Loop

**REQ-ID:** FR-CORE-001
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_loop.py`

```python
describe('AgentLoop'):
    # Message processing
    - processes inbound message and produces outbound response
    - calls LLM provider with correct context and tool definitions
    - executes tool calls returned by LLM
    - feeds tool results back to LLM for next iteration
    - stops after max_tool_iterations (20)
    - handles LLM errors gracefully (returns error message)

    # System messages
    - processes subagent announcements via system channel
    - routes system messages to correct session

    # Direct mode
    - process_direct returns response for CLI usage
```

**Run:** `pytest tests/agent/test_loop.py`

---

### TEST-CORE-002: Session Manager

**REQ-ID:** FR-CORE-002
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/session/test_manager.py`

```python
describe('SessionManager'):
    # CRUD
    - creates new session with unique key
    - retrieves existing session by key
    - saves session to JSONL file
    - loads session from JSONL file
    - deletes session and removes file
    - lists all sessions

    # Messages
    - adds user message to session
    - adds assistant message to session
    - get_history returns last 50 messages
    - get_history respects max_messages parameter
    - clears session messages

    # Persistence
    - first line of JSONL is metadata
    - subsequent lines are messages
    - handles corrupted JSONL gracefully
```

**Run:** `pytest tests/session/test_manager.py`

---

### TEST-CORE-003: Skills Loader

**REQ-ID:** FR-CORE-003
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_skills.py`

```python
describe('SkillsLoader'):
    # Discovery
    - lists built-in skills
    - lists workspace skills
    - workspace skills override built-in with same name
    - filters unavailable skills (missing requirements)

    # Loading
    - loads skill content by name
    - extracts YAML frontmatter metadata
    - builds XML summary for agent context
    - identifies always-loaded skills

    # Requirements
    - checks binary requirements (bins)
    - checks environment variable requirements (env)
    - reports missing requirements
```

**Run:** `pytest tests/agent/test_skills.py`

---

### TEST-CORE-004: Memory Store

**REQ-ID:** FR-CORE-004
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_memory.py`

```python
describe('MemoryStore'):
    # Daily notes
    - read_today returns empty string for new day
    - append_today adds timestamped entry
    - append_today creates file if not exists

    # Long-term memory
    - read_long_term returns MEMORY.md contents
    - write_long_term overwrites MEMORY.md

    # Recent memories
    - get_recent_memories returns last N days
    - get_recent_memories skips missing days
    - get_memory_context formats for agent prompt
```

**Run:** `pytest tests/agent/test_memory.py`

---

### TEST-CORE-005: Subagent Manager

**REQ-ID:** FR-CORE-005
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_subagent.py`

```python
describe('SubagentManager'):
    - spawns background task with focused prompt
    - subagent has limited tools (no message, no spawn)
    - subagent respects max iterations (15)
    - announces result back via system channel
    - handles subagent errors gracefully
```

**Run:** `pytest tests/agent/test_subagent.py`

---

### TEST-TOOL-001: File Tools

**REQ-ID:** FR-TOOL-001
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/tools/test_filesystem.py`

```python
describe('FileTools'):
    # read_file
    - reads file contents
    - expands ~ in path
    - returns error for non-existent file

    # write_file
    - writes content to file
    - creates parent directories
    - overwrites existing file

    # edit_file
    - replaces text pattern in file
    - warns on multiple pattern matches
    - returns error if pattern not found

    # list_dir
    - lists directory contents
    - formats with file/dir indicators
    - returns error for non-existent directory
```

**Run:** `pytest tests/agent/tools/test_filesystem.py`

---

### TEST-TOOL-002: Shell Tool

**REQ-ID:** FR-TOOL-002
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/tools/test_shell.py`

```python
describe('ShellTool'):
    - executes simple command (echo)
    - captures stdout
    - captures stderr
    - returns exit code on failure
    - enforces 60s timeout
    - truncates output at 10,000 characters
```

**Run:** `pytest tests/agent/tools/test_shell.py`

---

### TEST-TOOL-003: Web Tools

**REQ-ID:** FR-TOOL-003
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/tools/test_web.py`

```python
describe('WebTools'):
    # web_search
    - returns search results from Brave API
    - respects max_results configuration
    - handles API errors gracefully
    - returns error when no API key configured

    # web_fetch
    - fetches URL and extracts content
    - uses readability for HTML extraction
    - truncates long content
    - handles connection errors
    - handles invalid URLs
```

**Run:** `pytest tests/agent/tools/test_web.py`

---

### TEST-CHAN-001: Telegram Channel

**REQ-ID:** FR-CHAN-001
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/channels/test_telegram.py`

```python
describe('TelegramChannel'):
    # Message handling
    - receives text message and publishes to bus
    - receives photo and downloads media
    - receives voice message
    - receives document

    # Permissions
    - allows messages from allowed user IDs
    - allows messages from allowed usernames
    - rejects messages from unknown senders

    # Outbound
    - sends text message to chat
    - converts markdown to Telegram HTML
    - handles send errors gracefully
```

**Run:** `pytest tests/channels/test_telegram.py`

---

### TEST-CHAN-002: WhatsApp Channel

**REQ-ID:** FR-CHAN-002
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/channels/test_whatsapp.py`

```python
describe('WhatsAppChannel'):
    # Connection
    - connects to bridge via WebSocket
    - handles connection loss with reconnection
    - processes status messages

    # Messages
    - receives message and publishes to bus
    - sends outbound message via bridge
    - filters by allow_from phone numbers
```

**Run:** `pytest tests/channels/test_whatsapp.py`

---

### TEST-CRON-001: Cron Service

**REQ-ID:** FR-CRON-001
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/cron/test_service.py`

```python
describe('CronService'):
    # Job management
    - adds job with cron expression
    - adds job with interval (every)
    - adds one-time job (at)
    - removes job by ID
    - enables/disables job
    - lists jobs with optional filter

    # Scheduling
    - computes next run for cron expression
    - computes next run for interval
    - computes next run for one-time (returns None if past)
    - one-time jobs auto-delete after run (if configured)
    - recurring jobs recompute next run after execution

    # Execution
    - calls callback when job is due
    - tracks last_run and last_status
    - records errors in last_error
```

**Run:** `pytest tests/cron/test_service.py`

---

### TEST-CRON-002: Heartbeat Service

**REQ-ID:** FR-CRON-002
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/heartbeat/test_service.py`

```python
describe('HeartbeatService'):
    - triggers callback when HEARTBEAT.md has tasks
    - skips when HEARTBEAT.md is empty (only headers/comments)
    - skips completed checkboxes
    - handles HEARTBEAT_OK response
    - respects interval configuration
```

**Run:** `pytest tests/heartbeat/test_service.py`

---

### TEST-TEST-001: Test Infrastructure

**REQ-ID:** FR-TEST-001
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/`

```python
describe('Test Infrastructure'):
    - pytest discovers and runs all test files
    - pytest-asyncio handles async test functions
    - conftest.py provides shared fixtures
    - tmp_path fixtures for isolated file operations
    - mock LLM provider fixture for deterministic tests
```

**Run:** `pytest tests/`

---

### TEST-TEST-002: Agent Integration Tests

**REQ-ID:** FR-TEST-002
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_integration.py`

```python
describe('Agent Integration'):
    - processes message end-to-end (inbound → LLM → tool → response → outbound)
    - handles multi-turn tool calling
    - builds context with memory and skills correctly
    - spawns subagent and receives result
```

**Run:** `pytest tests/agent/test_integration.py`

---

### TEST-PERF-001: Streaming

**REQ-ID:** FR-PERF-001
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/providers/test_streaming.py`

```python
describe('LLM Streaming'):
    - streams text tokens to channel incrementally
    - batches tool calls (not streamed)
    - falls back to non-streaming if unsupported
    - works with Telegram channel
    - works with WhatsApp channel
```

**Run:** `pytest tests/providers/test_streaming.py`

---

### TEST-PERF-002: Rate Limiting

**REQ-ID:** FR-PERF-002
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/providers/test_ratelimit.py`

```python
describe('Rate Limiting'):
    - retries on 429 response with exponential backoff
    - stops after max retries (3)
    - notifies user on persistent failure
```

**Run:** `pytest tests/providers/test_ratelimit.py`

---

### TEST-MODAL-001: Vision Support

**REQ-ID:** FR-MODAL-001
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_vision.py`

```python
describe('Vision Support'):
    - includes base64 image in LLM context
    - supports JPEG, PNG, GIF, WebP
    - Telegram photo auto-download and inclusion
    - graceful fallback for non-vision models
```

**Run:** `pytest tests/agent/test_vision.py`

---

### TEST-MODAL-002: Voice Transcription

**REQ-ID:** FR-MODAL-002
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_voice.py`

```python
describe('Voice Transcription'):
    - transcribes voice message to text
    - includes transcription in agent context
    - handles transcription API errors
```

**Run:** `pytest tests/agent/test_voice.py`

---

### TEST-CHAN-003: Discord Channel

**REQ-ID:** FR-CHAN-003
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/channels/test_discord.py`

```python
describe('DiscordChannel'):
    - receives DM and publishes to bus
    - receives @mention in channel
    - sends outbound message
    - filters by allow_from user IDs
```

**Run:** `pytest tests/channels/test_discord.py`

---

### TEST-CHAN-004: Slack Channel

**REQ-ID:** FR-CHAN-004
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/channels/test_slack.py`

```python
describe('SlackChannel'):
    - receives DM via Socket Mode
    - receives @mention
    - sends outbound message
    - filters by allow_from user IDs
```

**Run:** `pytest tests/channels/test_slack.py`

---

### TEST-MEM-001: Configurable Session Limits

**REQ-ID:** FR-MEM-001
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/session/test_limits.py`

```python
describe('Session Limits'):
    - respects configurable max_messages
    - defaults to 50 if not configured
    - old sessions cleaned up after retention period
```

**Run:** `pytest tests/session/test_limits.py`

---

### TEST-MEM-002: Semantic Memory

**REQ-ID:** FR-MEM-002
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/agent/test_semantic_memory.py`

```python
describe('Semantic Memory'):
    - generates embeddings for memory entries
    - searches memories by semantic similarity
    - returns top-K relevant results
    - includes relevant memories in agent context
```

**Run:** `pytest tests/agent/test_semantic_memory.py`

---

### TEST-ROB-001: Cron Timezone

**REQ-ID:** FR-ROB-001
**Owner:** -
**Type:** Unit
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/cron/test_timezone.py`

```python
describe('Cron Timezone'):
    - evaluates cron expression in specified timezone
    - defaults to system timezone
    - CLI displays times in user timezone
```

**Run:** `pytest tests/cron/test_timezone.py`

---

### TEST-ROB-002: Crash Recovery

**REQ-ID:** FR-ROB-002
**Owner:** -
**Type:** Integration
**Status:** :black_circle: Not Started (0/0)
**Location:** `tests/test_recovery.py`

```python
describe('Crash Recovery'):
    - persists outbound message queue to disk
    - retries undelivered messages on restart
    - cron jobs resume after crash
    - agent loop restarts cleanly after exception
```

**Run:** `pytest tests/test_recovery.py`

---

## Agent Protocol

**Before claiming task:**
- Check test status for your REQ-ID
- Create test placeholder if :black_circle: Not Started

**During implementation:**
- Update status: :black_circle: -> :red_circle: -> :yellow_circle: -> :white_check_mark:
- Commit test file with implementation

**After completion:**
- Verify all tests :white_check_mark: Pass
- Update this file with final status
- Signal orchestrator for merge

---

*Last test run: Never (no tests exist yet)*
