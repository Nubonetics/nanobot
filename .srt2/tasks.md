# Multi-Agent Task Execution

> **Current Milestone:** Milestone 1 — Test Foundation
> **Current Phase:** Phase 1 — Test Infrastructure & Core Coverage
> **Next Phase:** Phase 2 — Streaming, Multi-Modal & New Channels

---

## Milestone 1: Test Foundation

**Goal:** Establish baseline test coverage for all 12 legacy components. Go from 0% to full unit + integration coverage before any feature work begins.

**Tasks:** TASK-001 through TASK-010 (10 tasks)
**Agents:** team-test, team-core, team-infra (3 parallel)
**Cross-cutting:** code-reviewer (PR review), test-runner (validation), docs-agent (sync)

### Exit Criteria

- [ ] All 12 legacy REQ-IDs have passing test suites
- [ ] `pytest tests/` runs clean with zero external API calls
- [ ] Coverage report generated (target: all public methods covered)
- [ ] code-reviewer signs off on test quality
- [ ] docs-agent updates SRT² docs with final status
- [ ] Tag: `milestone-1-complete`

### Parallelization Plan

```
Day 1:  team-test  → TASK-001 (pytest infra, fixtures, mock provider)
        ──────────────────────────────────────────────────────────────
Day 2:  team-core  → TASK-002 (agent loop), TASK-004 (memory), TASK-005 (skills)
        team-test  → TASK-003 (sessions), TASK-006 (tools)
        team-infra → TASK-007 (cron/heartbeat), TASK-008 (channels), TASK-010 (config/bus)
        ──────────────────────────────────────────────────────────────
Day 3:  team-core  → TASK-009 (agent integration)
        code-reviewer → review all PRs from Day 2
        ──────────────────────────────────────────────────────────────
Day 4:  test-runner → full suite validation
        docs-agent  → SRT² sync, changelog entry
        sprint-coordinator → merge to main, tag milestone
```

### Agent Workload

| Agent | Tasks | Files Touched |
|-------|-------|---------------|
| team-test | TASK-001, 003, 006 | `tests/conftest.py`, `tests/session/`, `tests/agent/tools/` |
| team-core | TASK-002, 004, 005, 009 | `tests/agent/test_loop.py`, `test_memory.py`, `test_skills.py`, `test_integration.py` |
| team-infra | TASK-007, 008, 010 | `tests/cron/`, `tests/channels/`, `tests/config/`, `tests/bus/` |
| code-reviewer | PR review (Day 3) | Read-only |
| test-runner | Validation (Day 4) | Read-only |
| docs-agent | SRT² sync (Day 4) | `.srt2/*.md`, `CHANGELOG.md` |
| sprint-coordinator | Merge + tag (Day 4) | `main` branch |

No file overlap between implementation agents — parallel safe.

### Critical Path
```
TASK-001 → TASK-002 → TASK-009 → milestone sign-off
```

### Dependency Graph

```
TASK-001 (Test Infra) [team-test]
    ├─→ TASK-002 (Agent Loop tests) [team-core]
    │       └─→ TASK-009 (Agent Integration tests) [team-core]
    ├─→ TASK-003 (Session tests) [team-test]
    ├─→ TASK-004 (Memory tests) [team-core]
    ├─→ TASK-005 (Skills tests) [team-core]
    ├─→ TASK-006 (Tool tests) [team-test]
    ├─→ TASK-007 (Cron tests) [team-infra]
    ├─→ TASK-008 (Channel tests) [team-infra]
    └─→ TASK-010 (Config & Bus tests) [team-infra]
```

### Post-Milestone Dependencies

Milestone 1 completion unlocks:
```
TASK-009 (Integration) ─→ TASK-011 (Streaming)
                        ─→ TASK-014 (Vision)

TASK-008 (Channel tests) ─→ TASK-016 (Discord)
                         ─→ TASK-017 (Slack)

TASK-003 (Session tests) ─→ TASK-018 (Session limits)
TASK-004 (Memory tests)  ─→ TASK-019 (Semantic memory)
TASK-007 (Cron tests)    ─→ TASK-020 (Timezone support)
```

---

## Future Milestones (Preview)

| Milestone | Phase | Tasks | Goal |
|-----------|-------|-------|------|
| **M1: Test Foundation** | Phase 1 | TASK-001–010 | Baseline test coverage for all legacy code |
| M2: Performance | Phase 2 | TASK-011–013 | Streaming + rate limiting |
| M3: Multi-Modal | Phase 3 | TASK-014–015 | Vision + voice support |
| M4: Channels | Phase 4 | TASK-016–017 | Discord + Slack |
| M5: Robustness | Phase 5 | TASK-018–021 | Memory, sessions, crash recovery |

---

## Phase 1: Test Infrastructure & Core Coverage

### TASK-001: Test Infrastructure Setup `[ ]`

**REQ-ID:** FR-TEST-001
**Owner:** team-test
**Branch:** `feature/FR-TEST-001-test-infra`
**Dependencies:** None (can start immediately)

**Context Budget:**
- Load: specs.md (conventions), FR-TEST-001 only
- Total: ~2k tokens

**Implementation:**
```
tests/
├── conftest.py           # Shared fixtures
├── __init__.py
├── agent/
│   └── __init__.py
├── channels/
│   └── __init__.py
├── cron/
│   └── __init__.py
├── session/
│   └── __init__.py
└── providers/
    └── __init__.py
```

```python
# tests/conftest.py
- tmp_workspace fixture (tmp_path with workspace structure)
- mock_llm_provider fixture (returns deterministic responses)
- mock_bus fixture (MessageBus with in-memory queues)
- mock_config fixture (default Config with test overrides)
- sample_skill fixture (minimal SKILL.md for testing)
```

**Tests:**
```python
# Verify infrastructure works
- conftest fixtures load without error
- mock LLM provider returns expected responses
- tmp_workspace has correct structure
```

**Acceptance:**
- `pytest tests/` discovers and runs tests
- All fixtures work
- No external API calls in tests

**Status:** :black_circle: Waiting
**Can Start:** Now

---

### TASK-002: Agent Loop Unit Tests `[ ]`

**REQ-ID:** FR-CORE-001
**Owner:** team-core
**Branch:** `feature/FR-CORE-001-agent-loop-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CORE-001, TASK-001 results, agent/loop.py
- Total: ~3k tokens

**Implementation:**
```python
# tests/agent/test_loop.py
- Test _process_message with mock LLM (no tool calls → direct response)
- Test _process_message with tool call → tool result → final response
- Test max iterations enforcement (20)
- Test error handling when LLM fails
- Test process_direct for CLI mode
```

**Tests:** Self-referential (this IS the test task)

**Acceptance:**
- All tests pass
- Uses mock LLM provider (no real API calls)
- Covers happy path and error cases

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-003: Session Manager Tests `[ ]`

**REQ-ID:** FR-CORE-002
**Owner:** team-test
**Branch:** `feature/FR-CORE-002-session-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CORE-002, TASK-001 results, session/manager.py
- Total: ~2.5k tokens

**Implementation:**
```python
# tests/session/test_manager.py
- Test get_or_create (new session)
- Test get_or_create (existing session)
- Test save/load roundtrip (JSONL format)
- Test add_message (user, assistant roles)
- Test get_history (last 50 messages)
- Test clear and delete
- Test list_sessions
- Test corrupted JSONL handling
```

**Acceptance:**
- All tests pass with tmp_path isolation
- No filesystem side effects

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-004: Memory Store Tests `[ ]`

**REQ-ID:** FR-CORE-004
**Owner:** team-core
**Branch:** `feature/FR-CORE-004-memory-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CORE-004, TASK-001 results, agent/memory.py
- Total: ~2k tokens

**Implementation:**
```python
# tests/agent/test_memory.py
- Test read_today / append_today
- Test read_long_term / write_long_term
- Test get_recent_memories (last N days)
- Test get_memory_context formatting
- Test file creation on first write
```

**Acceptance:**
- All tests pass with tmp_path isolation
- Tests cover file creation edge cases

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-005: Skills Loader Tests `[ ]`

**REQ-ID:** FR-CORE-003
**Owner:** team-core
**Branch:** `feature/FR-CORE-003-skills-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CORE-003, TASK-001 results, agent/skills.py
- Total: ~2.5k tokens

**Implementation:**
```python
# tests/agent/test_skills.py
- Test list_skills discovers built-in skills
- Test workspace skill overrides built-in
- Test load_skill returns content
- Test get_skill_metadata extracts frontmatter
- Test build_skills_summary XML format
- Test get_always_skills filtering
- Test requirement checking (bins, env)
```

**Acceptance:**
- All tests pass
- Uses sample_skill fixture for isolated tests

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-006: Tool Unit Tests `[ ]`

**REQ-ID:** FR-TOOL-001, FR-TOOL-002, FR-TOOL-003
**Owner:** team-test
**Branch:** `feature/FR-TOOL-001-tool-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-TOOL-001/002/003, TASK-001, agent/tools/*.py
- Total: ~3.5k tokens

**Implementation:**
```python
# tests/agent/tools/test_filesystem.py
- read_file, write_file, edit_file, list_dir tests

# tests/agent/tools/test_shell.py
- exec command, timeout, truncation tests

# tests/agent/tools/test_web.py (mocked HTTP)
- web_search with mock Brave API response
- web_fetch with mock HTTP response

# tests/agent/tools/test_registry.py
- register, execute, get_definitions tests
```

**Acceptance:**
- All tests pass
- Web tests use httpx mock (no real HTTP calls)
- Shell tests use safe commands (echo, true, false)

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-007: Cron & Heartbeat Tests `[ ]`

**REQ-ID:** FR-CRON-001, FR-CRON-002
**Owner:** team-infra
**Branch:** `feature/FR-CRON-001-cron-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CRON-001/002, TASK-001, cron/*.py, heartbeat/*.py
- Total: ~3k tokens

**Implementation:**
```python
# tests/cron/test_service.py
- add/remove/list/enable/disable jobs
- next_run computation for all schedule types
- one-time job deletion
- callback execution

# tests/heartbeat/test_service.py
- trigger with tasks in HEARTBEAT.md
- skip when empty
- HEARTBEAT_OK handling
```

**Acceptance:**
- All tests pass with tmp_path isolation
- No real timers (mock asyncio.sleep)

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-008: Channel Tests `[ ]`

**REQ-ID:** FR-CHAN-001, FR-CHAN-002
**Owner:** team-infra
**Branch:** `feature/FR-CHAN-001-channel-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-CHAN-001/002, TASK-001, channels/*.py
- Total: ~3k tokens

**Implementation:**
```python
# tests/channels/test_telegram.py (mocked python-telegram-bot)
- message handling (text, photo, voice)
- allow_from filtering
- outbound send
- markdown-to-HTML conversion

# tests/channels/test_whatsapp.py (mocked websockets)
- WebSocket message handling
- reconnection behavior
- allow_from filtering

# tests/channels/test_manager.py
- channel initialization from config
- outbound dispatch routing
```

**Acceptance:**
- All tests pass
- No real Telegram/WhatsApp connections
- Mock all external APIs

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-009: Agent Integration Tests `[ ]`

**REQ-ID:** FR-TEST-002
**Owner:** team-core
**Branch:** `feature/FR-TEST-002-integration-tests`
**Dependencies:** TASK-002 (agent loop tests)

**Context Budget:**
- Load: FR-TEST-002, TASK-002 results
- Total: ~2.5k tokens

**Implementation:**
```python
# tests/agent/test_integration.py
- End-to-end: inbound message → LLM → response → outbound
- Multi-turn: message → tool call → tool result → LLM → response
- Context: memory and skills loaded correctly
- Subagent: spawn → execute → announce result
```

**Acceptance:**
- Full pipeline works with mock LLM
- Message bus routes correctly
- Context includes expected components

**Status:** :black_circle: Blocked
**Blocked By:** TASK-002

---

### TASK-010: Config & Bus Tests `[ ]`

**REQ-ID:** FR-TEST-001 (remaining)
**Owner:** team-infra
**Branch:** `feature/FR-TEST-001-config-bus-tests`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-TEST-001, TASK-001, config/*.py, bus/*.py
- Total: ~2.5k tokens

**Implementation:**
```python
# tests/config/test_loader.py
- load_config from file
- save_config roundtrip
- camelCase/snake_case conversion
- environment variable overrides
- default values

# tests/bus/test_queue.py
- publish/consume inbound
- publish/consume outbound
- subscriber callbacks
- timeout behavior
```

**Acceptance:**
- All tests pass
- Config tests use tmp_path
- Bus tests verify async queue behavior

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

## Phase 2: Streaming & Performance

### TASK-011: LLM Response Streaming `[ ]`

**REQ-ID:** FR-PERF-001
**Owner:** team-core
**Branch:** `feature/FR-PERF-001-streaming`
**Dependencies:** TASK-009 (integration tests passing)

**Context Budget:**
- Load: FR-PERF-001, providers/litellm_provider.py, channels/base.py
- Total: ~3k tokens

**Implementation:**
```python
# nanobot/providers/litellm_provider.py
- Add stream=True option to LiteLLM call
- Yield tokens as they arrive
- Batch tool_calls (don't stream those)

# nanobot/agent/loop.py
- Stream text responses to channel incrementally
- Buffer until tool_call detected

# nanobot/channels/base.py
- Add stream_send() method for chunked delivery
- Telegram: edit message with growing content
- WhatsApp: send partial then update
```

**Tests:**
```python
# tests/providers/test_streaming.py
- Mock streaming response
- Verify tokens yielded incrementally
- Verify tool calls batched
- Verify fallback to non-streaming
```

**Acceptance:**
- Text responses stream to channels
- Tool calls still work correctly
- Non-streaming providers fall back gracefully

**Status:** :black_circle: Blocked
**Blocked By:** TASK-009

---

### TASK-012: API Rate Limiting `[ ]`

**REQ-ID:** FR-PERF-002
**Owner:** team-core
**Branch:** `feature/FR-PERF-002-rate-limiting`
**Dependencies:** TASK-001 (test infrastructure)

**Context Budget:**
- Load: FR-PERF-002, providers/litellm_provider.py
- Total: ~2k tokens

**Implementation:**
```python
# nanobot/providers/litellm_provider.py
- Catch 429/RateLimitError from LiteLLM
- Exponential backoff: 1s, 2s, 4s (max 3 retries)
- Log retry attempts
- Return error message after max retries
```

**Tests:**
```python
# tests/providers/test_ratelimit.py
- Mock 429 then success → verify retry
- Mock 3x 429 → verify gives up
- Verify backoff timing
```

**Acceptance:**
- Rate-limited calls retry automatically
- Max 3 retries
- User informed on persistent failure

**Status:** :black_circle: Waiting
**Blocked By:** TASK-001
**Can Start:** After TASK-001

---

### TASK-013: Integration Checkpoint (Phase 2) `[ ]`

**REQ-ID:** FR-PERF-001, FR-PERF-002
**Owner:** Orchestrator
**Branch:** `main`
**Dependencies:** TASK-011, TASK-012

**Implementation:**
- Run full test suite
- Verify streaming + rate limiting work together
- Merge to main
- Tag: `phase-2-complete`

**Status:** :black_circle: Blocked
**Blocked By:** TASK-011, TASK-012

---

## Phase 3: Multi-Modal

### TASK-014: Enhanced Vision Support `[ ]`

**REQ-ID:** FR-MODAL-001
**Owner:** team-core
**Branch:** `feature/FR-MODAL-001-vision`
**Dependencies:** TASK-009 (integration tests)

**Context Budget:**
- Load: FR-MODAL-001, agent/context.py, channels/telegram.py
- Total: ~3k tokens

**Implementation:**
```python
# nanobot/agent/context.py
- Detect vision-capable models from config
- Include base64 images in message content array
- Support JPEG, PNG, GIF, WebP MIME types

# nanobot/channels/telegram.py
- Auto-download photos on receive
- Pass media paths to agent context

# nanobot/providers/litellm_provider.py
- Pass image content blocks to LiteLLM
- Fallback: describe "image attached" for non-vision models
```

**Tests:**
```python
# tests/agent/test_vision.py
- base64 encoding of test image
- context includes image content block
- fallback for non-vision model
```

**Acceptance:**
- Images analyzed by vision-capable LLMs
- Graceful fallback for text-only models

**Status:** :black_circle: Blocked
**Blocked By:** TASK-009

---

### TASK-015: Voice Message Transcription `[ ]`

**REQ-ID:** FR-MODAL-002
**Owner:** team-infra
**Branch:** `feature/FR-MODAL-002-voice`
**Dependencies:** TASK-008 (channel tests)

**Context Budget:**
- Load: FR-MODAL-002, channels/telegram.py
- Total: ~2.5k tokens

**Implementation:**
```python
# nanobot/agent/tools/transcribe.py (new tool)
- Whisper API integration via LiteLLM or httpx
- Accept audio file path → return text

# nanobot/channels/telegram.py
- Auto-transcribe voice messages before sending to agent
- Include transcription as message content

# nanobot/config/schema.py
- Add transcription provider config
```

**Tests:**
```python
# tests/agent/test_voice.py
- Mock Whisper API response
- Voice message → transcription → agent context
- Error handling for failed transcription
```

**Acceptance:**
- Voice messages transcribed to text
- Transcription included in agent context
- API errors handled gracefully

**Status:** :black_circle: Blocked
**Blocked By:** TASK-008

---

## Phase 4: New Channels

### TASK-016: Discord Channel `[ ]`

**REQ-ID:** FR-CHAN-003
**Owner:** team-infra
**Branch:** `feature/FR-CHAN-003-discord`
**Dependencies:** TASK-008 (channel tests)

**Context Budget:**
- Load: FR-CHAN-003, channels/base.py, channels/telegram.py (as reference)
- Total: ~3k tokens

**Implementation:**
```python
# nanobot/channels/discord.py (new file)
- DiscordChannel extends BaseChannel
- discord.py bot with on_message handler
- DM and designated channel support
- Allow-list filtering
- Markdown formatting

# nanobot/config/schema.py
- Add discord config section (enabled, token, allow_from, guild_id)

# nanobot/channels/manager.py
- Register Discord channel
```

**Tests:**
```python
# tests/channels/test_discord.py
- Mock discord.py client
- DM and channel message handling
- Allow-list filtering
- Outbound message send
```

**Acceptance:**
- Discord bot receives and sends messages
- Allow-list works
- Integrated with channel manager

**Status:** :black_circle: Blocked
**Blocked By:** TASK-008

---

### TASK-017: Slack Channel `[ ]`

**REQ-ID:** FR-CHAN-004
**Owner:** team-infra
**Branch:** `feature/FR-CHAN-004-slack`
**Dependencies:** TASK-008 (channel tests)

**Context Budget:**
- Load: FR-CHAN-004, channels/base.py, channels/telegram.py (as reference)
- Total: ~3k tokens

**Implementation:**
```python
# nanobot/channels/slack.py (new file)
- SlackChannel extends BaseChannel
- Socket Mode (no public URL needed)
- DM and @mention handling
- Allow-list filtering
- mrkdwn formatting

# nanobot/config/schema.py
- Add slack config section (enabled, app_token, bot_token, allow_from)

# nanobot/channels/manager.py
- Register Slack channel
```

**Tests:**
```python
# tests/channels/test_slack.py
- Mock Slack Socket Mode client
- DM and mention handling
- Allow-list filtering
- Outbound message send
```

**Acceptance:**
- Slack bot receives and sends messages
- Socket Mode (no webhook server)
- Integrated with channel manager

**Status:** :black_circle: Blocked
**Blocked By:** TASK-008

---

## Phase 5: Memory & Robustness

### TASK-018: Configurable Session Limits `[ ]`

**REQ-ID:** FR-MEM-001
**Owner:** team-core
**Branch:** `feature/FR-MEM-001-session-limits`
**Dependencies:** TASK-003 (session tests)

**Context Budget:**
- Load: FR-MEM-001, session/manager.py, config/schema.py
- Total: ~2.5k tokens

**Implementation:**
```python
# nanobot/config/schema.py
- Add max_messages to agents.defaults (default: 50)
- Add session_retention_days (default: 30)

# nanobot/session/manager.py
- get_history reads max_messages from config
- cleanup_old_sessions removes sessions older than retention

# nanobot/cli/commands.py
- Add session cleanup to status or as separate command
```

**Tests:**
```python
# tests/session/test_limits.py
- Configurable max_messages respected
- Default 50 when not set
- Old session cleanup
```

**Acceptance:**
- Session limits configurable
- Old sessions cleaned up

**Status:** :black_circle: Blocked
**Blocked By:** TASK-003

---

### TASK-019: Semantic Memory Search `[ ]`

**REQ-ID:** FR-MEM-002
**Owner:** team-core
**Branch:** `feature/FR-MEM-002-semantic-memory`
**Dependencies:** TASK-004 (memory tests)

**Context Budget:**
- Load: FR-MEM-002, agent/memory.py
- Total: ~3k tokens

**Implementation:**
```python
# nanobot/agent/memory.py
- Embedding generation for memory entries (via LiteLLM embedding API)
- Vector storage (simple JSON with numpy cosine similarity)
- search_memories(query, top_k=5) → list of relevant entries
- Auto-index new memories on append

# nanobot/agent/context.py
- Include semantically relevant memories in prompt
- Replace or supplement date-based recent memories

# nanobot/config/schema.py
- Add memory.embedding_model config
```

**Tests:**
```python
# tests/agent/test_semantic_memory.py
- Mock embedding API
- Index and search roundtrip
- Top-K filtering
- Context integration
```

**Acceptance:**
- Semantic search returns relevant memories
- Integrated into agent context building
- Works with mock embeddings in tests

**Status:** :black_circle: Blocked
**Blocked By:** TASK-004

---

### TASK-020: Cron Timezone Support `[ ]`

**REQ-ID:** FR-ROB-001
**Owner:** team-infra
**Branch:** `feature/FR-ROB-001-cron-timezone`
**Dependencies:** TASK-007 (cron tests)

**Context Budget:**
- Load: FR-ROB-001, cron/types.py, cron/service.py
- Total: ~2.5k tokens

**Implementation:**
```python
# nanobot/cron/types.py
- Add timezone field to CronSchedule (default: system TZ)

# nanobot/cron/service.py
- Pass timezone to croniter for expression evaluation
- Store and display times in user timezone

# nanobot/cli/commands.py
- Display cron times in configured timezone
- --timezone flag for cron add
```

**Tests:**
```python
# tests/cron/test_timezone.py
- Cron expression evaluated in specified TZ
- Default to system TZ
- Display formatting
```

**Acceptance:**
- Cron jobs respect timezone
- CLI shows correct local times

**Status:** :black_circle: Blocked
**Blocked By:** TASK-007

---

### TASK-021: Graceful Crash Recovery `[ ]`

**REQ-ID:** FR-ROB-002
**Owner:** team-infra
**Branch:** `feature/FR-ROB-002-crash-recovery`
**Dependencies:** TASK-007, TASK-010 (cron + bus tests)

**Context Budget:**
- Load: FR-ROB-002, bus/queue.py, cron/service.py, agent/loop.py
- Total: ~3.5k tokens

**Implementation:**
```python
# nanobot/bus/queue.py
- Persist outbound queue to disk (JSONL)
- Load undelivered messages on startup
- Mark messages as delivered after send

# nanobot/cron/service.py
- Persist job state atomically (write-then-rename)
- Resume from last known state on startup

# nanobot/agent/loop.py
- Wrap main loop in restart-on-exception handler
- Log crash details for debugging
```

**Tests:**
```python
# tests/test_recovery.py
- Simulate crash: undelivered messages recovered
- Cron state persists across restart
- Agent loop restarts after exception
```

**Acceptance:**
- No message loss on crash
- Cron jobs resume correctly
- Agent self-heals

**Status:** :black_circle: Blocked
**Blocked By:** TASK-007, TASK-010

---

## Agent Status Board (Milestone 1)

| Agent | Milestone 1 Tasks | Day | Status |
|-------|-------------------|-----|--------|
| team-test | TASK-001 | 1 | :large_green_circle: Ready |
| team-core | TASK-002, 004, 005 | 2 | :white_circle: Blocked (TASK-001) |
| team-test | TASK-003, 006 | 2 | :white_circle: Blocked (TASK-001) |
| team-infra | TASK-007, 008, 010 | 2 | :white_circle: Blocked (TASK-001) |
| team-core | TASK-009 | 3 | :white_circle: Blocked (TASK-002) |
| code-reviewer | PR review | 3 | :white_circle: Idle (waiting for PRs) |
| test-runner | Full suite validation | 4 | :white_circle: Idle (waiting for tests) |
| docs-agent | SRT² sync + changelog | 4 | :white_circle: Idle (waiting for completions) |
| sprint-coordinator | Merge + tag `milestone-1-complete` | 4 | :white_circle: Idle (waiting for sign-off) |

---

## Merge Schedule

```
Phase 1:
  [ ] TASK-001 → main (test infra)
  [ ] TASK-002 through TASK-010 → main (all tests)
  [ ] Integration checkpoint

Phase 2:
  [ ] TASK-011 → main (streaming)
  [ ] TASK-012 → main (rate limiting)
  [ ] TASK-013: Integration checkpoint

Phase 3:
  [ ] TASK-014 → main (vision)
  [ ] TASK-015 → main (voice)

Phase 4:
  [ ] TASK-016 → main (Discord)
  [ ] TASK-017 → main (Slack)

Phase 5:
  [ ] TASK-018 → main (session limits)
  [ ] TASK-019 → main (semantic memory)
  [ ] TASK-020 → main (timezone)
  [ ] TASK-021 → main (crash recovery)
```

---

## Integration Protocol

### Merge Readiness Checklist

**Before merge:**
- [ ] All tests pass (`pytest tests/`)
- [ ] No conflicts with main
- [ ] Ruff lint passes (`ruff check nanobot/`)
- [ ] Integration test with dependencies passes

**After merge:**
- [ ] Tag merge point
- [ ] Notify dependent agents
- [ ] Update tasks.md with merge timestamp
- [ ] Dependent agents rebase on latest main

---

## Risk Mitigation

### Risk: No existing tests to validate against
**Impact:** Changes may break existing functionality silently
**Mitigation:**
- Phase 1 establishes test baseline before any changes
- Mock LLM provider ensures deterministic tests
- Integration tests verify end-to-end flow

### Risk: LiteLLM streaming API differences across providers
**Impact:** Streaming may not work uniformly
**Mitigation:**
- Test with mock provider first
- Add provider-specific fallback logic
- Streaming is optional (non-streaming still works)

### Risk: Discord/Slack dependencies bloat install size
**Impact:** Lightweight philosophy compromised
**Mitigation:**
- Make discord.py and slack-sdk optional dependencies
- Only import when channel is enabled
- Document in pyproject.toml as extras

---

*Last updated: 2026-02-03 by Orchestrator*
