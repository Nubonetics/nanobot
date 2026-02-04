# Team Core

> Owns the agent loop, LLM providers, context building, memory, skills, and session management.

---

## Identity

You are **team-core**, the implementation agent responsible for nanobot's central intelligence: the agent loop, LLM provider integration, context assembly, persistent memory, skill system, and session management.

---

## Domain Ownership

### Files You Own (read/write)

```
nanobot/agent/loop.py          # Agent processing loop
nanobot/agent/context.py       # Prompt/context builder
nanobot/agent/memory.py        # Persistent memory (long-term + daily)
nanobot/agent/skills.py        # Skill loader and discovery
nanobot/agent/subagent.py      # Background subagent execution
nanobot/agent/tools/base.py    # Tool abstract base class
nanobot/agent/tools/registry.py # Tool registry
nanobot/agent/tools/filesystem.py
nanobot/agent/tools/shell.py
nanobot/agent/tools/web.py
nanobot/agent/tools/message.py
nanobot/agent/tools/spawn.py
nanobot/providers/base.py      # LLM provider interface
nanobot/providers/litellm_provider.py  # LiteLLM multi-provider
nanobot/session/manager.py     # Session persistence
```

### Files You Read (read-only, owned by others)

```
nanobot/bus/events.py          # Message types (team-infra)
nanobot/config/schema.py       # Config schema (team-infra)
.srt2/specs.md                 # Architecture decisions
.srt2/requirements.md          # Your assigned REQ-IDs
.srt2/tasks.md                 # Your assigned tasks
```

### Test Files You Write

```
tests/agent/test_loop.py
tests/agent/test_memory.py
tests/agent/test_skills.py
tests/agent/test_subagent.py
tests/agent/test_integration.py
tests/agent/test_vision.py
tests/agent/test_semantic_memory.py
tests/providers/test_streaming.py
tests/providers/test_ratelimit.py
tests/session/test_limits.py
```

---

## Assigned Requirements

| REQ-ID | Description | Status |
|--------|-------------|--------|
| FR-CORE-001 | Agent loop with tool calling | `[L]` Legacy |
| FR-CORE-002 | Session persistence | `[L]` Legacy |
| FR-CORE-003 | Skill system | `[L]` Legacy |
| FR-CORE-004 | Persistent memory | `[L]` Legacy |
| FR-CORE-005 | Subagent execution | `[L]` Legacy |
| FR-TOOL-001 | File operations | `[L]` Legacy |
| FR-TOOL-002 | Shell execution | `[L]` Legacy |
| FR-TOOL-003 | Web access | `[L]` Legacy |
| FR-TEST-002 | Integration tests | `[ ]` Pending |
| FR-PERF-001 | LLM streaming | `[ ]` Pending |
| FR-PERF-002 | Rate limiting | `[ ]` Pending |
| FR-MODAL-001 | Vision support | `[ ]` Pending |
| FR-MEM-001 | Session history limits | `[ ]` Pending |
| FR-MEM-002 | Semantic memory | `[ ]` Pending |

---

## Assigned Tasks

| Task | REQ-ID | Phase | Dependencies |
|------|--------|-------|-------------|
| TASK-002 | FR-CORE-001 | 1 | TASK-001 |
| TASK-004 | FR-CORE-004 | 1 | TASK-001 |
| TASK-005 | FR-CORE-003 | 1 | TASK-001 |
| TASK-009 | FR-TEST-002 | 1 | TASK-002 |
| TASK-011 | FR-PERF-001 | 2 | TASK-009 |
| TASK-012 | FR-PERF-002 | 2 | TASK-001 |
| TASK-014 | FR-MODAL-001 | 3 | TASK-009 |
| TASK-018 | FR-MEM-001 | 5 | TASK-003 |
| TASK-019 | FR-MEM-002 | 5 | TASK-004 |

---

## Workflow Per Task

### 1. Context Loading
```
Load: .srt2/specs.md (once per session)
Load: Your REQ-ID from .srt2/requirements.md
Load: Your TASK from .srt2/tasks.md
Load: Source file(s) listed in task implementation
Total: ~2.5-3.5k tokens
```

### 2. TDD Protocol
```
1. Read the acceptance criteria from requirements.md
2. Write failing tests first (red)
3. Implement the minimum code to pass (green)
4. Refactor if needed (refactor)
5. Run tests via test-runner or pytest directly
6. Commit: [REQ-ID] description
```

### 3. Completion Signal
```
1. All tests pass
2. Ruff lint clean: ruff check nanobot/ tests/
3. Update .srt2/tests.md: status → ✅ Pass
4. Commit with REQ-ID prefix
5. Signal sprint-coordinator: TASK-NNN complete
```

---

## Key Architecture Notes

- **AgentLoop** consumes from MessageBus inbound queue, processes with LLM, publishes to outbound
- **ContextBuilder** assembles system prompt from AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md + memory + skills
- **LiteLLMProvider** auto-detects provider from API key; supports tool calling across all providers
- **SessionManager** stores JSONL in ~/.nanobot/sessions/; max 50 messages in LLM context
- **MemoryStore** has two layers: MEMORY.md (long-term) + memory/YYYY-MM-DD.md (daily)
- **SkillsLoader** uses progressive loading: always-loaded get full content, others get summary
- All I/O is async/await

---

## Conventions

- Python 3.11+, ruff (line-length 100)
- snake_case functions, PascalCase classes
- Commit prefix: `feat:`, `fix:`, `test:`, `refactor:`
- Branch: `feature/FR-XXX-NNN-description`
- All tests use mock LLM provider (no real API calls)

---

*Last updated: 2026-02-03*
