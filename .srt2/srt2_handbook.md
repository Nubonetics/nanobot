# SRT² Handbook v1.0 — Overview
## Specs. Requirements. Tests². Tasks.

> [!TIP]
> **TL;DR:** 4 files designed for parallel AI agent execution. No code without REQ-ID + test. Agent-enforced traceability.

> **Handbook Structure:**
> - **01-overview.md** ← You are here
> - [02-file-templates.md](#srt²-handbook-v10--file-templates) — Template specs for all 4 SRT² files
> - [03-workflow.md](#srt²-handbook-v10--workflow) — Multi-agent workflow, communication, context loading
> - [04-task-rules.md](#srt²-handbook-v10--task-rules) — Decomposition rules and constitutional protocol
> - [05-orchestration.md](#srt²-handbook-v10--orchestration) — Coordination, merging, risk mitigation

---

## What is SRT²?

**SRT²** (pronounced "SRT-squared") is a lightweight, AI-native development methodology optimized for **parallel execution by multiple AI agents**.

**Core Principle:**
> No code without a REQ-ID in requirements.md and a test in tests.md

**Key Innovation:**
> Tasks are decomposed for parallel AI agent execution with minimal context loading

**Philosophy:**
- AI agents work in parallel on independent workstreams
- Each agent loads only what it needs (~2-5k tokens)
- Fresh context per task prevents degradation
- Orchestration happens through task dependencies, not meetings
- Humans review and merge, agents implement

---

## The 4 Files

```
.srt/
├── specs.md          # Architecture decisions, conventions (agents read once)
├── requirements.md   # REQ-IDs with acceptance criteria (agents read subset)
├── tests.md         # Coverage matrix and test specs (agents update)
└── tasks.md         # Parallel execution plan (orchestrator writes, agents execute)
```

| File | Purpose | Who Reads | Update Frequency |
|------|---------|-----------|------------------|
| **specs.md** | Vision, decisions, conventions | All agents (once at start) | Per decision |
| **requirements.md** | What to build (REQ-IDs) | Agents read only their assigned REQ-IDs | Per feature |
| **tests.md** | Verification evidence | Agent updates after testing | Per test |
| **tasks.md** | Parallel execution coordination | Orchestrator + active agents | Continuously |

---

## Why SRT² Is AI-Native

### Traditional Development (Human-Paced)
```
Developer reads all docs → Plans work → Implements serially → Tests → Done
Context: 34k+ tokens loaded once, held in human brain
Speed: 1 feature at a time
```

### SRT² Development (AI-Paced)
```
Agent Alpha: REQ-001 → Test → Implement → Merge (2.5k tokens)
Agent Beta:  REQ-002 → Test → Implement → Merge (2.5k tokens)  [PARALLEL]
Agent Gamma: REQ-003 → Test → Implement → Merge (2.5k tokens)  [PARALLEL]
Orchestrator: Coordinates merges, resolves conflicts
```

**Result:** 3-4x faster through parallelization

---

## Core Concepts

### 1. Task Atomicity

Every task in tasks.md is:
- **Independent** - Can execute without waiting for other tasks
- **Atomic** - One logical unit (30-90 minutes of work)
- **Testable** - Has clear verification criteria
- **Traceable** - Links to specific REQ-ID

### 2. Context Minimization

Agents load only:
- specs.md (once, ~1k tokens)
- Their assigned REQ-ID from requirements.md (~500 tokens)
- Their task from tasks.md (~800 tokens)
- **Total: ~2.5k tokens vs 34k+ for full context**

### 3. Fresh Context Per Task

After each task completion:
- Agent context resets
- Next task loads fresh context
- Prevents context rot and degradation

### 4. Dependency-Based Orchestration

```
TASK-001 (Auth Utils) ─┬─→ TASK-002 (Login) ──→ TASK-004 (Session)
                       └─→ TASK-003 (Reset) ──┘
```

- Green: No dependencies (start immediately)
- Yellow: Waiting for one dependency
- Gray: Waiting for multiple dependencies

---

## Getting Started

### For AI Agent Teams (Default)

```bash
mkdir -p .srt
cd .srt

# Create the 4 files
touch specs.md requirements.md tests.md tasks.md

# Initialize with templates (see 02-file-templates.md)
```

### For Human-Only Teams

SRT² works for humans too, but consider full STRIVE if:
- Team > 5 people
- Need formal phase gates
- Require enterprise governance

---
---

# SRT² Handbook v1.0 — File Templates
## Template specifications for all 4 SRT² files

> **Handbook Structure:**
> - [01-overview.md](#srt²-handbook-v10--overview) — What SRT² is, core concepts, getting started
> - **02-file-templates.md** ← You are here
> - [03-workflow.md](#srt²-handbook-v10--workflow) — Multi-agent workflow, communication, context loading
> - [04-task-rules.md](#srt²-handbook-v10--task-rules) — Decomposition rules and constitutional protocol
> - [05-orchestration.md](#srt²-handbook-v10--orchestration) — Coordination, merging, risk mitigation

---

## specs.md Template

```markdown
# Project Specifications

> **Vision:** [2-3 sentence product vision]

---

## Architecture Decisions

### [YYYY-MM-DD] Decision Title

**REQ-IDs:** [Affected requirements]

**Context:** What problem we're solving

**Decision:** What we chose

**Alternatives:**
- Option A: Why not
- Option B: Why not

**Consequences:**
- ✅ Benefit
- ⚠️ Tradeoff

---

## Technical Stack

**Language:** TypeScript 5.3
**Frontend:** React 18, Next.js 14
**Backend:** Node.js 20, Express
**Database:** PostgreSQL 16, Prisma
**Deploy:** Vercel + Railway

---

## Conventions

### Code Style
- File naming: kebab-case
- Folder structure: Feature-based (`/features/auth/`)
- Functions: Verb-first (`getUserById`)

### Git Workflow
- Branches: `feature/REQ-ID-description`
- Commits: `[REQ-ID] Description`
- PRs: Squash and merge to main

### API Design
- REST principles, plural nouns
- Response: `{data: {...}, meta: {...}}`
- Errors: `{error: {code, message, field?}}`

---

## Agent Loading Strategy

**Agents should load:**
- This file once at session start
- Only relevant sections as needed
- Never load entire specs.md repeatedly

**Context budget:**
- Full specs.md: ~1k tokens
- Single decision: ~200 tokens
- Conventions section: ~300 tokens

---

*Last updated: [timestamp]*
```

---

## requirements.md Template

```markdown
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

## Authentication

**FR-AUTH-001** `[x]` User login with email/password

**Owner:** Agent Alpha
**Branch:** `feature/auth-login`
**Dependencies:** None

**Why:** Users need secure access to their accounts

**Acceptance Criteria:**
- POST /api/auth/login accepts {email, password}
- Valid credentials → 200 + JWT cookie
- Invalid credentials → 401 with generic error
- Rate limit: 5 attempts/15min

**Test:** `TEST-AUTH-001`

**Commits:** `a1b2c3d`, `e4f5g6h`

---

**FR-AUTH-002** `[~]` Password reset flow

**Owner:** Agent Beta
**Branch:** `feature/auth-reset`
**Dependencies:** FR-AUTH-001 (JWT utils)

**Why:** Users forget passwords and need recovery

**Acceptance Criteria:**
- Request reset → email with token link
- Token valid 1 hour
- Reset form validates password strength
- Success → redirect to login

**Test:** `TEST-AUTH-002`

**Commits:** -

---

**FR-AUTH-003** `[ ]` Session middleware

**Owner:** Agent Gamma
**Branch:** `feature/auth-session`
**Dependencies:** FR-AUTH-001 (JWT validation)

**Why:** Protected routes need authentication

**Acceptance Criteria:**
- Middleware verifies JWT on protected routes
- Invalid/expired JWT → 401
- Session refresh endpoint available

**Test:** `TEST-AUTH-003`

**Commits:** -

---

## Agent Notes

**Context Loading:**
- Agents load ONLY their assigned REQ-IDs
- Example: Agent Alpha loads only FR-AUTH-001 (~500 tokens)
- Never load all requirements (~34k tokens)

**Ownership:**
- Each REQ-ID assigned to one agent
- Prevents merge conflicts
- Parallel execution enabled

---

*Total: X | Done: Y | In Progress: Z | Pending: W*
```

---

## tests.md Template

```markdown
# Test Coverage Matrix

> **Rule:** Every REQ-ID must have test coverage

---

## Coverage Dashboard

**Overall:** X% (Y/Z passing)

| REQ-ID | Test ID | Type | Status | Owner | Location |
|--------|---------|------|--------|-------|----------|
| FR-AUTH-001 | TEST-AUTH-001 | Integration | ✅ Pass | Agent Alpha | `tests/auth/login.test.ts` |
| FR-AUTH-002 | TEST-AUTH-002 | Integration | 🟡 WIP | Agent Beta | `tests/auth/reset.test.ts` |
| FR-AUTH-003 | TEST-AUTH-003 | Integration | ⚫ Not Started | Agent Gamma | - |

**Status Icons:**
- ✅ Pass - All tests passing
- 🟡 WIP - Tests written, some failing
- 🔴 Fail - Tests exist, all failing
- ⚫ Not Started - No tests yet

---

## Test Suites

### TEST-AUTH-001: User Login

**REQ-ID:** FR-AUTH-001
**Owner:** Agent Alpha
**Type:** Integration
**Status:** ✅ Pass (4/4)
**Location:** `tests/auth/login.test.ts`

```typescript
describe('POST /api/auth/login', () => {
  ✅ returns 200 + JWT cookie with valid credentials
  ✅ returns 401 with invalid credentials
  ✅ returns 400 with missing fields
  ✅ enforces rate limit after 5 attempts
})
```

**Run:** `npm test -- login.test.ts`
**Last Run:** 2026-01-28 14:30 UTC by Agent Alpha

---

### TEST-AUTH-002: Password Reset

**REQ-ID:** FR-AUTH-002
**Owner:** Agent Beta
**Type:** Integration
**Status:** 🟡 WIP (2/4)
**Location:** `tests/auth/reset.test.ts`

```typescript
describe('Password Reset Flow', () => {
  ✅ sends reset email with valid token
  ✅ token expires after 1 hour
  ⚫ validates password strength on reset
  ⚫ redirects to login on success
})
```

**Run:** `npm test -- reset.test.ts`
**Last Run:** 2026-01-28 15:00 UTC by Agent Beta

---

## Agent Protocol

**Before claiming task:**
- Check test status for your REQ-ID
- Create test placeholder if ⚫ Not Started

**During implementation:**
- Update status: ⚫ → 🔴 → 🟡 → ✅
- Commit test file with implementation

**After completion:**
- Verify all tests ✅ Pass
- Update this file with final status
- Signal orchestrator for merge

---

*Last test run: [timestamp]*
```

---

## tasks.md Template (The Orchestration Core)

```markdown
# Multi-Agent Task Execution

> **Current Phase:** Authentication MVP
> **Integration Point:** 2026-02-10 16:00 UTC

---

## Execution Strategy

### Parallelization Plan

```
Day 1: TASK-001 (sequential, foundational)
Day 2: TASK-002, TASK-003 (parallel, depend on TASK-001)
Day 3: TASK-004 (depends on TASK-002)
Day 4: Integration tests + deploy
```

### Critical Path
```
TASK-001 → TASK-002 → TASK-004 (3 days)
```

---

## Active Tasks

### TASK-001: JWT Utility Functions `[x]`

**REQ-ID:** FR-AUTH-001
**Owner:** Agent Alpha
**Branch:** `feature/auth-login`
**Dependencies:** None (can start immediately)

**Context Budget:**
- Load: specs.md (conventions), FR-AUTH-001 only
- Total: ~2k tokens

**Implementation:**
```typescript
// src/utils/jwt.ts
- createToken(payload) → signed JWT
- verifyToken(token) → payload or throw
- Use JWT_SECRET from env
```

**Tests:**
```typescript
// tests/utils/jwt.test.ts
- Token creation
- Token verification
- Expiry handling
- Invalid token rejection
```

**Acceptance:**
- All tests pass
- No dependencies on other tasks
- Ready to merge

**Status:** ✅ Complete
**Completed:** 2026-02-10 09:00 UTC
**Commits:** `abc123`, `def456`

---

### TASK-002: Login Endpoint `[~]`

**REQ-ID:** FR-AUTH-001
**Owner:** Agent Alpha
**Branch:** `feature/auth-login`
**Dependencies:** TASK-001 (JWT utils) ✅

**Context Budget:**
- Load: FR-AUTH-001, TASK-001 results
- Total: ~2.5k tokens

**Implementation:**
```typescript
// src/api/auth/login.ts
POST /api/auth/login
1. Validate {email, password}
2. Find user, bcrypt compare
3. Generate JWT cookie (using utils from TASK-001)
4. Return 200 or 401
```

**Tests:**
```typescript
// tests/api/auth/login.test.ts
- Valid credentials → 200 + cookie
- Invalid credentials → 401
- Missing fields → 400
- Rate limiting works
```

**Acceptance:**
- Integration test passes
- Uses JWT utils from TASK-001
- No merge conflicts

**Status:** 🟡 In Progress
**Started:** 2026-02-10 09:30 UTC
**ETA:** 2026-02-10 11:00 UTC
**Progress:** 60% (endpoint done, tests in progress)

---

### TASK-003: Password Reset Request `[ ]`

**REQ-ID:** FR-AUTH-002
**Owner:** Agent Beta
**Branch:** `feature/auth-reset`
**Dependencies:** TASK-001 (JWT utils) ✅

**Context Budget:**
- Load: FR-AUTH-002, TASK-001 results
- Total: ~2.5k tokens

**Implementation:**
```typescript
// src/api/auth/reset-request.ts
POST /api/auth/reset-request
1. Validate {email}
2. Generate reset token (1hr expiry)
3. Send email with link
4. Return 200 (even if user not found)
```

**Tests:**
```typescript
// tests/api/auth/reset-request.test.ts
- Email sent with valid token
- Token expires in 1 hour
- No user enumeration
```

**Acceptance:**
- Integration test passes
- Email delivery verified
- Token generation works

**Status:** ⚫ Waiting
**Blocked By:** TASK-001 (needs JWT utils) ✅ UNBLOCKED
**Can Start:** Now
**ETA:** 2026-02-10 13:00 UTC

---

### TASK-004: Session Middleware `[ ]`

**REQ-ID:** FR-AUTH-003
**Owner:** Agent Gamma
**Branch:** `feature/auth-session`
**Dependencies:** TASK-002 (login endpoint) 🟡

**Context Budget:**
- Load: FR-AUTH-003, TASK-002 results
- Total: ~2.5k tokens

**Implementation:**
```typescript
// src/middleware/auth.ts
- Verify JWT from cookie
- Attach user to req.user
- Return 401 if invalid/expired
```

**Tests:**
```typescript
// tests/middleware/auth.test.ts
- Valid token → user attached
- Invalid token → 401
- Expired token → 401
- Missing token → 401
```

**Acceptance:**
- Middleware works with TASK-002 login
- Integration test passes
- Protected routes use middleware

**Status:** ⚫ Blocked
**Blocked By:** TASK-002 (login endpoint)
**Resume When:** TASK-002 merged to main
**ETA:** 2026-02-10 15:00 UTC

---

*See [05-orchestration.md](#srt²-handbook-v10--orchestration) for dependency graphs, merge schedules, and coordination protocols.*
```

---
---

# SRT² Handbook v1.0 — Workflow
## Multi-agent workflow, communication, and context loading

> **Handbook Structure:**
> - [01-overview.md](#srt²-handbook-v10--overview) — What SRT² is, core concepts, getting started
> - [02-file-templates.md](#srt²-handbook-v10--file-templates) — Template specs for all 4 SRT² files
> - **03-workflow.md** ← You are here
> - [04-task-rules.md](#srt²-handbook-v10--task-rules) — Decomposition rules and constitutional protocol
> - [05-orchestration.md](#srt²-handbook-v10--orchestration) — Coordination, merging, risk mitigation

---

## Multi-Agent Workflow

### Phase 1: Planning (Orchestrator)

```
1. Orchestrator reviews requirements.md
2. Identifies dependencies between REQ-IDs
3. Breaks into atomic tasks (30-90 min each)
4. Assigns to agents based on:
   - Dependencies (critical path first)
   - Agent expertise
   - Load balancing
5. Writes tasks.md with full execution plan
```

### Phase 2: Parallel Execution (Agents)

```
Agent Alpha:
  1. Read tasks.md → Find TASK-001
  2. Load minimal context (~2.5k tokens)
  3. Execute task (code + tests)
  4. Signal completion
  5. Context resets
  6. Repeat with next task

Agent Beta: [PARALLEL]
  1. Read tasks.md → Find TASK-003
  2. Check dependencies (blocked by TASK-001)
  3. Wait for signal
  4. Execute when unblocked

Agent Gamma: [PARALLEL]
  1. Read tasks.md → Find TASK-004
  2. Check dependencies (blocked by TASK-002)
  3. Wait for signal
  4. Execute when unblocked
```

### Phase 3: Integration (Orchestrator)

```
1. Monitor agent signals
2. Review PRs for merge readiness
3. Merge in dependency order:
   - TASK-001 → main
   - TASK-002 → main (rebase on TASK-001)
   - TASK-003 → main (rebase on TASK-001)
   - TASK-004 → main (rebase on TASK-002)
4. Run integration tests
5. Deploy if all pass
```

---

## Agent Communication Protocol

### Task Claiming
```
Agent: "Claiming TASK-003"
→ Update this file: Owner = Agent Beta, Status = [~]
→ Update requirements.md: FR-AUTH-002 = [~]
```

### Task Completion
```
Agent: "TASK-003 complete, ready for merge"
→ Update this file: Status = [x], Completion timestamp
→ Update tests.md: TEST-AUTH-002 = ✅ Pass
→ Update requirements.md: Append commit hashes
→ Create PR with title: [FR-AUTH-002] Reset request endpoint
→ Signal orchestrator for review
```

### Blocking Encountered
```
Agent: "TASK-004 blocked, waiting for TASK-002"
→ Update this file: Status = [!], Blocked By = TASK-002
→ Agent goes idle or picks another ready task
→ Orchestrator notifies when unblocked
```

---

## Context Loading Rules

**Per-Task Loading (Agents):**
1. Load specs.md once at session start (~1k tokens)
2. Load assigned REQ-ID from requirements.md (~500 tokens)
3. Load assigned TASK from tasks.md (~800 tokens)
4. Load dependency outputs if needed (~1k tokens)
5. **Total: ~2.5-3.5k tokens per task**

**Never Load:**
- Other agents' tasks
- Completed phase documentation
- Full requirements.md (~34k tokens)
- Irrelevant REQ-IDs

**Fresh Context Pattern:**
```
Agent completes TASK-001
→ Context resets
→ Agent starts TASK-002
→ Fresh context loads (TASK-002 only)
→ No context pollution from TASK-001
```

---
---

# SRT² Handbook v1.0 — Task Rules
## Decomposition rules and agent constitutional protocol

> **Handbook Structure:**
> - [01-overview.md](#srt²-handbook-v10--overview) — What SRT² is, core concepts, getting started
> - [02-file-templates.md](#srt²-handbook-v10--file-templates) — Template specs for all 4 SRT² files
> - [03-workflow.md](#srt²-handbook-v10--workflow) — Multi-agent workflow, communication, context loading
> - **04-task-rules.md** ← You are here
> - [05-orchestration.md](#srt²-handbook-v10--orchestration) — Coordination, merging, risk mitigation

---

## Task Decomposition Rules

### Rule 1: Atomic Tasks (30-90 minutes)

**Bad:**
```
TASK-001: Implement entire auth system
- 15 files
- 8 commits
- 4 hours
- Multiple REQ-IDs
```

**Good:**
```
TASK-001: JWT utility functions
- 2 files: jwt.ts, jwt.test.ts
- 1-2 commits
- 30 minutes
- One REQ-ID

TASK-002: Login endpoint
- 2 files: login.ts, login.test.ts
- 1-2 commits
- 45 minutes
- Same REQ-ID
```

### Rule 2: File Ownership (No Conflicts)

**Bad:**
```
TASK-002: Modifies src/utils/helpers.ts
TASK-003: Modifies src/utils/helpers.ts
→ Guaranteed conflict
```

**Good:**
```
TASK-002: Creates src/utils/jwt.ts (new file)
TASK-003: Creates src/utils/email.ts (new file)
→ No conflict, parallel safe
```

**If shared file needed:**
```
TASK-001: Creates src/utils/common.ts, merges first
TASK-002: Imports from common.ts (read-only)
→ Sequential dependency, no conflict
```

### Rule 3: Dependency Chains (Max 3 Levels)

**Bad (Too Sequential):**
```
T1 → T2 → T3 → T4 → T5 → T6
(6 days serial)
```

**Good (Balanced Parallelism):**
```
T1 → T2 → T4
  ↓
  T3 → T5
(3 days with 2 agents)
```

### Rule 4: Integration Checkpoints (Every 3-5 Tasks)

```
Phase 1 (5 tasks):
  Tasks 1-5 execute
  → Checkpoint: Integrate all
  → Run integration tests
  → Deploy to dev

Phase 2 (7 tasks):
  Tasks 6-12 execute
  → Checkpoint: Integrate all
  → Run integration tests
  → Deploy to staging

Phase 3 (Final):
  Tasks 13-15 execute
  → Final integration
  → Full regression
  → Deploy to production
```

---

## Agent Constitutional Protocol

> Constitutional principles enforced through agent behavior, not documents.

### Anti-Ghost Policy

```yaml
before_task:
  - CHECK: "Does my REQ-ID exist in requirements.md?"
  - IF NOT: "STOP. Create REQ-ID first."
  - CHECK: "Does my TEST-ID exist in tests.md?"
  - IF NOT: "STOP. Create test spec first."
```

---
---

# SRT² Handbook v1.0 — Orchestration
## Coordination, merging, risk mitigation, and session tracking

> **Handbook Structure:**
> - [01-overview.md](#srt²-handbook-v10--overview) — What SRT² is, core concepts, getting started
> - [02-file-templates.md](#srt²-handbook-v10--file-templates) — Template specs for all 4 SRT² files
> - [03-workflow.md](#srt²-handbook-v10--workflow) — Multi-agent workflow, communication, context loading
> - [04-task-rules.md](#srt²-handbook-v10--task-rules) — Decomposition rules and constitutional protocol
> - **05-orchestration.md** ← You are here

---

## Dependency Graph

```
TASK-001 (JWT Utils)
    ├─→ TASK-002 (Login)
    │       └─→ TASK-004 (Session)
    └─→ TASK-003 (Reset)
            └─→ Integration Point → Deploy
```

**Legend:**
- 🟢 Green: Ready (no blockers)
- 🟡 Yellow: In progress
- ⚪ Gray: Blocked
- 🔴 Red: Integration checkpoint

---

## Agent Status Board

| Agent | Current Task | Status | Progress | ETA |
|-------|-------------|--------|----------|-----|
| Agent Alpha | TASK-002 | 🟡 Active | 60% | 11:00 |
| Agent Beta | TASK-003 | 🟢 Ready | 0% | 13:00 |
| Agent Gamma | TASK-004 | ⚪ Blocked | 0% | 15:00 |

---

## Merge Schedule

```
09:00 ✅ TASK-001 → main (merged)
11:00 [ ] TASK-002 → main (pending)
13:00 [ ] TASK-003 → main (after TASK-002)
15:00 [ ] TASK-004 → main (after TASK-002)
16:00 [ ] Integration tests
17:00 [ ] Deploy to staging
```

---

## Integration Protocol

### Merge Readiness Checklist

**Before merge:**
- [ ] All tests ✅ Pass
- [ ] No conflicts with main
- [ ] Integration test with dependencies passes
- [ ] Code review approved (if human review required)

**After merge:**
- [ ] Tag merge point: `task-NNN-merged`
- [ ] Notify dependent agents
- [ ] Update this file with merge timestamp
- [ ] Dependent agents rebase on latest main

### Conflict Resolution

**If merge conflict:**
1. Later task rebases on earlier merge
2. Re-run tests
3. If tests fail → rollback, investigate
4. Orchestrator reviews resolution

**Prevention:**
- File ownership (agents don't touch same files)
- Small, atomic tasks
- Frequent integration checkpoints

---

## Risk Mitigation

### Risk: Agent Alpha delayed on TASK-002
**Impact:** Blocks TASK-004 (cascading delay)
**Mitigation:**
- TASK-002 is simplest (login endpoint)
- Agent Alpha = most experienced
- Buffer: estimated 1.5hrs, allocated 2hrs
- If delay >30min, orchestrator reassigns

### Risk: Merge conflicts between TASK-002 and TASK-003
**Impact:** Integration delay
**Mitigation:**
- TASK-002 and TASK-003 have no file overlap
- Verified during planning: different files
- If conflict emerges, TASK-003 rebases on TASK-002

### Risk: Integration tests fail at checkpoint
**Impact:** Deploy delay, missed deadline
**Mitigation:**
- Each task runs own tests before merge
- Checkpoint at day 2 (partial integration)
- Rollback plan: revert to last checkpoint
- Dedicated debug agent on standby

---

## Session Context

### Decisions Made Today

| Time | Decision | Impact |
|------|----------|--------|
| 09:00 | Use jose library for JWT | FR-AUTH-001, FR-AUTH-002 |
| 09:30 | Generic error for login failures | FR-AUTH-001 security |
| 10:00 | 1-hour token expiry for reset | FR-AUTH-002 |

### Blockers

| Issue | Owner | ETA | Workaround |
|-------|-------|-----|------------|
| SMTP credentials | @ops | 2026-02-12 | Console logging for dev |

### Agent Preferences Discovered

- Agent Alpha prefers verbose variable names
- Agent Beta writes tests first (strict TDD)
- Agent Gamma optimizes for performance

---

*Last updated: 2026-02-10 10:30 UTC by Orchestrator*
