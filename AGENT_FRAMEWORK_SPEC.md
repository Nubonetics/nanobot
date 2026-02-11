# nanobot — Implementation-Level Specification

> **Generated from**: nanobot v0.1.0 source code audit
> **Codebase**: ~4,000 lines of Python across 42 source files
> **Framework type**: Lightweight ReAct-loop AI agent with multi-channel messaging, tool execution, subagent spawning, persistent memory, and scheduled tasks

---

## Purpose

This document specifies every behavioral mechanism of the nanobot agent framework at implementation-level precision. Given a YAML agent description and this specification, a code generator should be able to produce source code that is **behaviorally identical** to the framework's implementation.

---

## 1. Entry Point & Invocation Lifecycle

### 1.1 Public API Surface

There are **three** entry points into the agent processing pipeline:

#### 1.1.1 CLI Single-Message Mode
**File**: `nanobot/cli/commands.py:317–323`

```
nanobot agent -m "Hello" --session cli:default
```

1. `load_config()` loads `~/.nanobot/config.json` → `Config` object
2. API key resolved via `Config.get_api_key()` — priority order: OpenRouter > Anthropic > OpenAI > Gemini > Zhipu > Groq > vLLM (`config/schema.py:101–112`). If model starts with `"bedrock/"`, no API key is required.
3. `LiteLLMProvider(api_key, api_base, default_model)` is constructed
4. `AgentLoop(bus, provider, workspace, ...)` is constructed
5. `asyncio.run(agent_loop.process_direct(message, session_id))` is called
6. Result is printed to console via `rich.Console`

#### 1.1.2 CLI Interactive Mode
**File**: `nanobot/cli/commands.py:324–341`

When `--message` is omitted, enters a `while True` loop:
1. Reads user input via `console.input()`
2. Skips empty input (`if not user_input.strip(): continue`)
3. Calls `agent_loop.process_direct(user_input, session_id)` per input
4. Prints response, loops back
5. `KeyboardInterrupt` breaks the loop

#### 1.1.3 Gateway Mode (Multi-Channel Server)
**File**: `nanobot/cli/commands.py:155–270`

```
nanobot gateway --port 18790
```

1. Loads config, creates `MessageBus`, `LiteLLMProvider`, `AgentLoop`
2. Creates `CronService` with `on_cron_job` callback that calls `agent.process_direct()`
3. Creates `HeartbeatService` with `on_heartbeat` callback that calls `agent.process_direct()`
4. Creates `ChannelManager(config, bus)` which initializes enabled channels
5. Runs `asyncio.gather(agent.run(), channels.start_all())` concurrently
6. Agent loop polls the inbound queue; channels push messages to it

### 1.2 AgentLoop Construction

**File**: `nanobot/agent/loop.py:36–68`

**Constructor signature**:
```python
AgentLoop(
    bus: MessageBus,
    provider: LLMProvider,
    workspace: Path,
    model: str | None = None,        # defaults to provider.get_default_model()
    max_iterations: int = 20,
    brave_api_key: str | None = None,
    exec_config: ExecToolConfig | None = None,  # defaults to ExecToolConfig()
)
```

**Objects created during `__init__`**:

| Object | Type | Purpose |
|--------|------|---------|
| `self.context` | `ContextBuilder(workspace)` | System prompt assembly |
| `self.sessions` | `SessionManager(workspace)` | Conversation persistence |
| `self.tools` | `ToolRegistry()` | Tool registration & dispatch |
| `self.subagents` | `SubagentManager(provider, workspace, bus, model, brave_api_key, exec_config)` | Background task management |

**Default tools registered** (`_register_default_tools`, lines 70–95):
1. `ReadFileTool()` → name `"read_file"`
2. `WriteFileTool()` → name `"write_file"`
3. `EditFileTool()` → name `"edit_file"`
4. `ListDirTool()` → name `"list_dir"`
5. `ExecTool(working_dir, timeout, restrict_to_workspace)` → name `"exec"`
6. `WebSearchTool(api_key=brave_api_key)` → name `"web_search"`
7. `WebFetchTool()` → name `"web_fetch"`
8. `MessageTool(send_callback=bus.publish_outbound)` → name `"message"`
9. `SpawnTool(manager=subagents)` → name `"spawn"`

### 1.3 `process_direct()` — Direct Invocation Path

**File**: `nanobot/agent/loop.py:318–337`

```python
async def process_direct(self, content: str, session_key: str = "cli:direct") -> str:
```

Algorithm:
1. Constructs an `InboundMessage(channel="cli", sender_id="user", chat_id="direct", content=content)`
2. Delegates to `_process_message(msg)`
3. Returns `response.content if response else ""`

**Note**: The `session_key` parameter is accepted but **not used** — `_process_message` derives the session key from `msg.session_key` which is always `"cli:direct"`.

### 1.4 `run()` — Bus-Driven Event Loop

**File**: `nanobot/agent/loop.py:97–124`

```python
async def run(self) -> None:
```

Algorithm:
1. Sets `self._running = True`
2. Enters `while self._running:` loop
3. Calls `asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)`
4. On `TimeoutError`: `continue` (poll again)
5. On message received: calls `self._process_message(msg)`
6. If response is non-None: publishes via `self.bus.publish_outbound(response)`
7. On processing exception: publishes error `OutboundMessage` with `f"Sorry, I encountered an error: {str(e)}"`

### 1.5 How the User Message Enters the Session

**File**: `nanobot/agent/loop.py:148–218`

The user message is appended to the session **after** the agent loop completes, not before:

```python
# Line 216-218: AFTER the agent loop finishes
session.add_message("user", msg.content)
session.add_message("assistant", final_content)
self.sessions.save(session)
```

The user message is included in the LLM call via `context.build_messages()` (line 161) which appends it as the final `{"role": "user", "content": ...}` message. The session only stores the final user/assistant pair — intermediate tool calls are **not** persisted to the session.

---

## 2. The Core Loop (ReAct / Reason-Act)

### 2.1 Outer Loop Structure

**File**: `nanobot/agent/loop.py:167–210`

The core loop is a simple `while` loop with an iteration counter:

```python
iteration = 0
final_content = None

while iteration < self.max_iterations:   # default: 20
    iteration += 1

    response = await self.provider.chat(
        messages=messages,
        tools=self.tools.get_definitions(),
        model=self.model
    )

    if response.has_tool_calls:
        # ... execute tools, append results to messages
    else:
        final_content = response.content
        break
```

### 2.2 Per-Step Execution

Each iteration performs exactly:

1. **LLM Call**: `self.provider.chat(messages, tools, model)` → returns `LLMResponse`
2. **Branch on tool calls**:
   - **If `response.has_tool_calls` is True** (i.e., `len(response.tool_calls) > 0`):
     a. Constructs `tool_call_dicts` list — each entry has `{id, type:"function", function:{name, arguments:json.dumps(tc.arguments)}}`
     b. Appends assistant message via `context.add_assistant_message(messages, response.content, tool_call_dicts)`
     c. Iterates over `response.tool_calls` **sequentially** (NOT parallel):
        - Calls `self.tools.execute(tool_call.name, tool_call.arguments)`
        - Appends tool result via `context.add_tool_result(messages, tool_call.id, tool_call.name, result)`
     d. Loop continues to next iteration
   - **If `response.has_tool_calls` is False**:
     a. Sets `final_content = response.content`
     b. `break` — exits the loop

### 2.3 Termination Conditions

The loop terminates under exactly two conditions:

1. **No tool calls in response**: `response.has_tool_calls` returns `False` → `break`
2. **Max iterations reached**: `iteration >= self.max_iterations` (default 20) → loop condition fails

### 2.4 Post-Loop Fallback

**File**: `nanobot/agent/loop.py:212–214`

```python
if final_content is None:
    final_content = "I've completed processing but have no response to give."
```

This occurs when the loop exhausts `max_iterations` without the LLM producing a non-tool-call response.

### 2.5 "is_final_response" Predicate

The framework uses **no explicit `is_final_response` predicate**. The termination logic is:

```python
final_response = not response.has_tool_calls
# which resolves to:
final_response = len(response.tool_calls) == 0
```

### 2.6 Streaming

The framework does **not** implement streaming. All LLM calls are non-streaming — `litellm.acompletion()` is called without `stream=True`. Responses are received as complete objects.

### 2.7 Tool Execution Ordering

**Critical**: Tools are executed **sequentially**, not in parallel. When the LLM returns multiple tool calls in a single response, they are executed one at a time in order via a `for` loop (line 200–206). There is no `asyncio.gather()` for tool execution in the main agent loop.

---

## 3. Preprocessing Pipeline (Request Processors)

nanobot has **no formal request processor pipeline**. Instead, context assembly is performed by `ContextBuilder.build_messages()` which is called once before the agent loop begins.

### 3.1 Context Assembly — `build_messages()`

**File**: `nanobot/agent/context.py:115–147`

**Signature**:
```python
def build_messages(
    self,
    history: list[dict[str, Any]],
    current_message: str,
    skill_names: list[str] | None = None,
    media: list[str] | None = None,
) -> list[dict[str, Any]]:
```

**Algorithm** (executed in exact order):
1. **System prompt**: Calls `self.build_system_prompt(skill_names)`, appends as `{"role": "system", "content": system_prompt}`
2. **History**: Extends with `history` (list of `{"role": ..., "content": ...}` dicts from session)
3. **Current message**: Calls `_build_user_content(current_message, media)`, appends as `{"role": "user", "content": user_content}`

### 3.2 System Prompt Assembly — `build_system_prompt()`

**File**: `nanobot/agent/context.py:27–70`

**Algorithm** (executed in exact order):

1. **Core Identity** (`_get_identity()`, line 72–101):
   - Generates a header with current date/time (`datetime.now().strftime("%Y-%m-%d %H:%M (%A)")`)
   - Includes workspace path
   - Hardcoded instructions about tool usage behavior
   - Result is a multi-line string starting with `"# nanobot 🐈"`

2. **Bootstrap Files** (`_load_bootstrap_files()`, line 103–113):
   - Iterates over `BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]`
   - For each file, checks if `workspace / filename` exists
   - If exists, reads content and formats as `"## {filename}\n\n{content}"`
   - Joins all found files with `"\n\n"`

3. **Memory Context** (`memory.get_memory_context()`, line 48–50):
   - Calls `MemoryStore.get_memory_context()`
   - If non-empty, wraps in `"# Memory\n\n{memory}"`
   - Memory context includes:
     - Long-term memory from `memory/MEMORY.md` (prefixed with `"## Long-term Memory\n"`)
     - Today's notes from `memory/YYYY-MM-DD.md` (prefixed with `"## Today's Notes\n"`)

4. **Always-loaded Skills** (lines 53–58):
   - Calls `skills.get_always_skills()` → returns skill names where frontmatter has `always=true`
   - For each such skill, loads full SKILL.md content (with frontmatter stripped)
   - Wraps in `"# Active Skills\n\n{content}"`

5. **Available Skills Summary** (lines 61–68):
   - Calls `skills.build_skills_summary()` → returns XML-formatted `<skills>` block
   - Lists ALL skills (including unavailable ones), each with `available="true|false"`, name, description, location
   - Wraps in `"# Skills\n\n..."` with instructions for progressive loading

6. **Final Assembly**: All non-empty parts are joined with `"\n\n---\n\n"` separator

### 3.3 History Construction

**File**: `nanobot/session/manager.py:39–53`

`Session.get_history(max_messages=50)`:
1. Takes the last `max_messages` messages from `self.messages`
2. Strips all fields except `role` and `content` (removes `timestamp` and other metadata)
3. Returns `[{"role": m["role"], "content": m["content"]} for m in recent]`

**Critical limitation**: Only user/assistant text pairs are stored in the session. Tool calls and tool results from previous turns are **not** persisted or replayed.

### 3.4 User Content Construction with Media

**File**: `nanobot/agent/context.py:149–165`

`_build_user_content(text, media)`:
1. If `media` is `None` or empty: returns `text` as plain string
2. For each path in `media`:
   - Resolves to `Path`, guesses MIME type via `mimetypes.guess_type()`
   - Skips if file doesn't exist, MIME is None, or MIME doesn't start with `"image/"`
   - Base64-encodes the file, creates `{"type": "image_url", "image_url": {"url": "data:{mime};base64,{b64}"}}`
3. If no valid images found: returns `text` as plain string
4. If images found: returns `images + [{"type": "text", "text": text}]` (images **before** text)

### 3.5 Tool Context Updates

**File**: `nanobot/agent/loop.py:152–158`

Before building messages, the loop updates context on two tools:
1. `MessageTool.set_context(msg.channel, msg.chat_id)` — so the message tool knows which channel to reply to
2. `SpawnTool.set_context(msg.channel, msg.chat_id)` — so subagent announcements route back correctly

---

## 4. Contents Construction Algorithm

### 4.1 Message List Structure

The `messages` list passed to the LLM has the following structure:

```
[0] {"role": "system", "content": <system_prompt>}
[1..N] History messages: {"role": "user"|"assistant", "content": <text>}
[N+1] Current user message: {"role": "user", "content": <text_or_multipart>}
[N+2..] Tool interaction messages accumulated during the loop:
    {"role": "assistant", "content": <text>, "tool_calls": [...]}
    {"role": "tool", "tool_call_id": <id>, "name": <name>, "content": <result>}
```

### 4.2 No Branch-Based Context Segregation

nanobot does **not** implement branch-based context. All messages are in a single linear list. There is no concept of branches, context isolation between agent turns, or event filtering.

### 4.3 No Rewind or Transcription Events

nanobot does **not** implement rewind events, transcription aggregation, or event replay mechanisms. The conversation is a simple linear history.

### 4.4 Session History Filtering

The only filtering applied to history is the `max_messages=50` truncation in `Session.get_history()`. There are no predicates based on agent identity, event type, or branch ID.

### 4.5 "include_contents" Mode

There is no distinction between "full history" and "current turn" modes. All available history (up to 50 messages) is always included.

### 4.6 How Tool Interactions Are Accumulated

During the agent loop, tool interactions are appended to the `messages` list in-place:

1. `context.add_assistant_message(messages, content, tool_calls)` → appends `{"role": "assistant", "content": content or "", "tool_calls": tool_calls}` (line 211–217)
2. `context.add_tool_result(messages, tool_call_id, tool_name, result)` → appends `{"role": "tool", "tool_call_id": id, "name": name, "content": result}` (line 186–192)

These accumulate in the `messages` list across all loop iterations. When the loop re-invokes the LLM, the full accumulated message list is sent.

---

## 5. LLM Call Mechanics

### 5.1 Provider Architecture

**File**: `nanobot/providers/base.py`

Abstract base: `LLMProvider(api_key, api_base)`
- Abstract method: `async chat(messages, tools, model, max_tokens=4096, temperature=0.7) -> LLMResponse`
- Abstract method: `get_default_model() -> str`

Single implementation: `LiteLLMProvider` (`nanobot/providers/litellm_provider.py`)

### 5.2 LiteLLMProvider Construction

**File**: `nanobot/providers/litellm_provider.py:20–61`

```python
LiteLLMProvider(
    api_key: str | None = None,
    api_base: str | None = None,
    default_model: str = "anthropic/claude-opus-4-5"
)
```

**Provider detection algorithm** (lines 30–58):

1. `is_openrouter = (api_key starts with "sk-or-") or (api_base contains "openrouter")`
2. `is_vllm = bool(api_base) and not is_openrouter`
3. API key environment variable routing:
   - OpenRouter → `OPENROUTER_API_KEY`
   - vLLM → `OPENAI_API_KEY`
   - `"anthropic" in default_model` → `ANTHROPIC_API_KEY` (setdefault)
   - `"openai" or "gpt" in default_model` → `OPENAI_API_KEY` (setdefault)
   - `"gemini" in default_model.lower()` → `GEMINI_API_KEY` (setdefault)
   - `"zhipu" or "glm" or "zai" in default_model` → `ZHIPUAI_API_KEY` (setdefault)
   - `"groq" in default_model` → `GROQ_API_KEY` (setdefault)
4. If `api_base`: sets `litellm.api_base = api_base`
5. Disables LiteLLM debug info: `litellm.suppress_debug_info = True`

### 5.3 LLM Invocation — `chat()`

**File**: `nanobot/providers/litellm_provider.py:63–131`

**Algorithm**:

1. **Model resolution** (lines 84–106):
   - `model = model or self.default_model`
   - OpenRouter prefix: if `is_openrouter and not model.startswith("openrouter/")` → prepend `"openrouter/"`
   - Zhipu prefix: if `("glm" or "zhipu" in model.lower()) and not prefixed` → prepend `"zhipu/"`
   - vLLM prefix: if `is_vllm` → prepend `"hosted_vllm/"` (always, unconditionally)
   - Gemini prefix: if `"gemini" in model.lower() and not model.startswith("gemini/")` → prepend `"gemini/"`

2. **Kwargs construction** (lines 108–121):
   ```python
   kwargs = {
       "model": model,
       "messages": messages,
       "max_tokens": max_tokens,   # default 4096
       "temperature": temperature, # default 0.7
   }
   if api_base: kwargs["api_base"] = api_base
   if tools:
       kwargs["tools"] = tools
       kwargs["tool_choice"] = "auto"
   ```

3. **LLM call** (lines 123–131):
   ```python
   try:
       response = await acompletion(**kwargs)   # litellm.acompletion
       return self._parse_response(response)
   except Exception as e:
       return LLMResponse(content=f"Error calling LLM: {str(e)}", finish_reason="error")
   ```

**Critical**: On any exception, the error is returned as an `LLMResponse` with `content` set to the error string and `finish_reason="error"`. The agent loop will see `has_tool_calls=False` and treat this error message as the final response.

### 5.4 Response Parsing — `_parse_response()`

**File**: `nanobot/providers/litellm_provider.py:133–169`

1. Extracts `choice = response.choices[0]`, `message = choice.message`
2. **Tool calls**: If `message.tool_calls` exists and is truthy:
   - For each `tc` in `message.tool_calls`:
     - `args = tc.function.arguments`
     - If `args` is a string: `json.loads(args)` — on `JSONDecodeError`: `args = {"raw": args}`
     - Creates `ToolCallRequest(id=tc.id, name=tc.function.name, arguments=args)`
3. **Usage**: Extracts `prompt_tokens`, `completion_tokens`, `total_tokens` if available
4. Returns `LLMResponse(content=message.content, tool_calls=tool_calls, finish_reason=choice.finish_reason or "stop", usage=usage)`

### 5.5 No Callbacks

The framework implements **no** before_model_callback, after_model_callback, or on_model_error callbacks. The LLM call is a direct invocation without any hook points.

### 5.6 No LLM Call Count Tracking

There is no explicit tracking or enforcement of LLM call counts beyond the `max_iterations` loop guard. Usage data is captured in `LLMResponse.usage` but never inspected.

---

## 6. Tool Dispatch & Execution

### 6.1 Tool Registration

**File**: `nanobot/agent/tools/registry.py:15–20`

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.name] = tool
```

Tools are stored by name. Re-registering with the same name silently overwrites.

### 6.2 Tool Definition Export

**File**: `nanobot/agent/tools/registry.py:34–36`

```python
def get_definitions(self) -> list[dict[str, Any]]:
    return [tool.to_schema() for tool in self._tools.values()]
```

Each tool's `to_schema()` (`base.py:114–123`) returns:
```python
{
    "type": "function",
    "function": {
        "name": self.name,
        "description": self.description,
        "parameters": self.parameters,  # JSON Schema object
    }
}
```

### 6.3 Tool Dispatch Algorithm

**File**: `nanobot/agent/tools/registry.py:38–62`

```python
async def execute(self, name: str, params: dict[str, Any]) -> str:
    tool = self._tools.get(name)
    if not tool:
        return f"Error: Tool '{name}' not found"
    try:
        errors = tool.validate_params(params)
        if errors:
            return f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors)
        return await tool.execute(**params)
    except Exception as e:
        return f"Error executing {name}: {str(e)}"
```

**Exact sequence**:
1. Look up tool by name in `self._tools` dict
2. If not found: return error string (no exception raised)
3. Validate parameters via `tool.validate_params(params)`
4. If validation errors: return formatted error string
5. Call `await tool.execute(**params)` — params are unpacked as kwargs
6. On any exception: catch and return error string

**Critical**: All errors are returned as strings to the LLM, never raised. The agent loop never sees exceptions from tool execution.

### 6.4 Parameter Validation

**File**: `nanobot/agent/tools/base.py:55–112`

`Tool.validate_params(params)`:
1. Gets `schema = self.parameters or {}`
2. If `"type"` not in schema: defaults to `{"type": "object", ...}`
3. If schema type is not `"object"`: raises `ValueError`
4. Delegates to `_validate_schema(params, schema, path="")`

`_validate_schema(value, schema, path)` — recursive validation:
- **Type checking**: Uses `_TYPE_MAP = {"string": str, "integer": int, "number": (int, float), "boolean": bool, "array": list, "object": dict}`
- **Enum**: Checks `value in schema["enum"]`
- **Numeric ranges**: `minimum`, `maximum` for integer/number
- **String lengths**: `minLength`, `maxLength` for string
- **Object**: Checks `required` keys exist, recurses into known `properties`
- **Array**: Recurses into `items` schema for each element
- **Unknown fields are ignored** (no `additionalProperties` enforcement)

### 6.5 Tool Execution Lifecycle

There is **no** callback pipeline around tool execution. The lifecycle is simply:

```
ToolRegistry.execute(name, params)
  → tool.validate_params(params)
  → tool.execute(**params)
  → return result string
```

No before_tool_callback, after_tool_callback, plugin callbacks, confirmation, or authentication mechanisms exist.

### 6.6 Complete Tool Inventory

#### 6.6.1 `read_file` — ReadFileTool
**File**: `nanobot/agent/tools/filesystem.py:9–46`
- **Parameters**: `path` (string, required)
- **Algorithm**: `Path(path).expanduser()` → check exists → check is_file → `read_text("utf-8")`
- **Errors**: "File not found", "Not a file", "Permission denied"

#### 6.6.2 `write_file` — WriteFileTool
**File**: `nanobot/agent/tools/filesystem.py:49–86`
- **Parameters**: `path` (string, required), `content` (string, required)
- **Algorithm**: `Path(path).expanduser()` → `parent.mkdir(parents=True, exist_ok=True)` → `write_text(content, "utf-8")`
- **Returns**: `"Successfully wrote {len(content)} bytes to {path}"`

#### 6.6.3 `edit_file` — EditFileTool
**File**: `nanobot/agent/tools/filesystem.py:89–144`
- **Parameters**: `path`, `old_text`, `new_text` (all required strings)
- **Algorithm**:
  1. Read file, check `old_text in content`
  2. If `content.count(old_text) > 1`: return warning about ambiguity
  3. `content.replace(old_text, new_text, 1)` — replaces first occurrence only
  4. Write back

#### 6.6.4 `list_dir` — ListDirTool
**File**: `nanobot/agent/tools/filesystem.py:147–191`
- **Parameters**: `path` (string, required)
- **Algorithm**: `Path(path).expanduser()` → `sorted(dir_path.iterdir())` → prefix with `📁 ` or `📄 `

#### 6.6.5 `exec` — ExecTool
**File**: `nanobot/agent/tools/shell.py:12–141`
- **Parameters**: `command` (string, required), `working_dir` (string, optional)
- **Construction params**: `timeout=60`, `working_dir=None`, `deny_patterns=[...]`, `allow_patterns=[]`, `restrict_to_workspace=False`
- **Safety guard** (`_guard_command`, lines 111–141):
  1. Checks command against deny patterns (regex):
     - `rm -r/-rf/-fr`, `del /f /q`, `rmdir /s`, `format/mkfs/diskpart`, `dd if=`, `> /dev/sd`, `shutdown/reboot/poweroff`, fork bomb pattern
  2. If `allow_patterns` is non-empty: blocks commands not matching any allow pattern
  3. If `restrict_to_workspace`: blocks `../` path traversal and absolute paths outside `cwd`
- **Execution**: `asyncio.create_subprocess_shell(command, stdout=PIPE, stderr=PIPE, cwd=cwd)`
- **Timeout**: `asyncio.wait_for(process.communicate(), timeout=self.timeout)` — on timeout, kills process
- **Output truncation**: Caps at 10,000 chars
- **STDERR**: Appended with `"STDERR:\n"` prefix
- **Non-zero exit**: Appends `"\nExit code: {returncode}"`

#### 6.6.6 `web_search` — WebSearchTool
**File**: `nanobot/agent/tools/web.py:46–90`
- **Parameters**: `query` (string, required), `count` (integer, optional, 1–10)
- **API**: Brave Search API at `https://api.search.brave.com/res/v1/web/search`
- **Auth**: `X-Subscription-Token` header with API key
- **Returns**: Formatted text with numbered results (title, URL, description)

#### 6.6.7 `web_fetch` — WebFetchTool
**File**: `nanobot/agent/tools/web.py:93–163`
- **Parameters**: `url` (string, required), `extractMode` ("markdown"|"text", default "markdown"), `maxChars` (integer, optional)
- **URL validation**: Must be http/https with valid domain
- **HTTP client**: `httpx.AsyncClient(follow_redirects=True, max_redirects=5, timeout=30.0)`
- **Content extraction**:
  - JSON: `json.dumps(r.json(), indent=2)`
  - HTML: Uses `readability.Document` → extracts `summary()` → converts to markdown or strips tags
  - Other: raw text
- **Returns**: JSON object with `{url, finalUrl, status, extractor, truncated, length, text}`

#### 6.6.8 `message` — MessageTool
**File**: `nanobot/agent/tools/message.py:9–86`
- **Parameters**: `content` (string, required), `channel` (string, optional), `chat_id` (string, optional)
- **Context**: Uses `_default_channel` and `_default_chat_id` set by `set_context()`
- **Algorithm**: Constructs `OutboundMessage`, calls `self._send_callback(msg)` (which is `bus.publish_outbound`)

#### 6.6.9 `spawn` — SpawnTool
**File**: `nanobot/agent/tools/spawn.py:11–65`
- **Parameters**: `task` (string, required), `label` (string, optional)
- **Algorithm**: Calls `SubagentManager.spawn(task, label, origin_channel, origin_chat_id)`
- **Returns**: Immediate status string: `"Subagent [{label}] started (id: {task_id}). I'll notify you when it completes."`

---

## 7. Event System & Session Persistence

### 7.1 Message Events

nanobot does **not** use a formal event schema. Instead, it uses two simple dataclasses for message routing:

#### 7.1.1 InboundMessage
**File**: `nanobot/bus/events.py:8–23`

```python
@dataclass
class InboundMessage:
    channel: str          # "telegram", "whatsapp", "cli", "system"
    sender_id: str        # User identifier
    chat_id: str          # Chat/channel identifier
    content: str          # Message text
    timestamp: datetime   # default: datetime.now()
    media: list[str]      # default: [] — local file paths
    metadata: dict[str, Any]  # default: {} — channel-specific data

    @property
    def session_key(self) -> str:
        return f"{self.channel}:{self.chat_id}"
```

#### 7.1.2 OutboundMessage
**File**: `nanobot/bus/events.py:26–35`

```python
@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list[str]      # default: []
    metadata: dict[str, Any]  # default: {}
```

### 7.2 Session Schema

**File**: `nanobot/session/manager.py:14–58`

```python
@dataclass
class Session:
    key: str                        # "channel:chat_id"
    messages: list[dict[str, Any]]  # default: []
    created_at: datetime            # default: datetime.now()
    updated_at: datetime            # default: datetime.now()
    metadata: dict[str, Any]       # default: {}
```

**Message format** in session (added by `add_message`):
```python
{
    "role": str,           # "user" or "assistant"
    "content": str,        # message text
    "timestamp": str,      # datetime.now().isoformat()
    # + any additional **kwargs
}
```

### 7.3 Session Persistence Format (JSONL)

**File**: `nanobot/session/manager.py:136–153`

Sessions are stored at `~/.nanobot/sessions/{safe_key}.jsonl`:

```jsonl
{"_type": "metadata", "created_at": "...", "updated_at": "...", "metadata": {}}
{"role": "user", "content": "Hello", "timestamp": "2024-01-01T12:00:00"}
{"role": "assistant", "content": "Hi there!", "timestamp": "2024-01-01T12:00:01"}
```

- First line: metadata record with `_type: "metadata"`
- Subsequent lines: message records (one per line)
- File is rewritten entirely on each save (not appended)
- Key sanitization: `safe_filename(key.replace(":", "_"))` removes `<>:"/\|?*` chars

### 7.4 Session Loading

**File**: `nanobot/session/manager.py:100–134`

1. Check `_cache` dict first
2. Read JSONL file line by line
3. Lines with `_type == "metadata"` → extract metadata and created_at
4. All other lines → append to messages list
5. Construct `Session` object

### 7.5 Session Caching

Sessions are cached in `SessionManager._cache: dict[str, Session]`. Cache is populated on load and updated on save. No TTL or eviction policy.

### 7.6 No State Deltas or Artifact Deltas

nanobot does not implement state deltas, artifact deltas, or any incremental state management. Session state is simply the list of messages.

---

## 8. Agent Transfer Mechanism

### 8.1 No Agent Transfer

nanobot does **not** implement a `transfer_to_agent` tool or any agent transfer mechanism. There is no concept of:
- Sub-agent selection or handoff
- Agent hierarchies (parent/child)
- Peer agent transfers
- SingleFlow vs. AutoFlow

### 8.2 Subagent Spawning (Alternative)

Instead of agent transfer, nanobot implements **subagent spawning** via the `spawn` tool, which creates an independent background agent task.

**File**: `nanobot/agent/subagent.py:47–84`

`SubagentManager.spawn()`:
1. Generates `task_id = str(uuid.uuid4())[:8]`
2. Creates display label: `label or task[:30] + "..."`
3. Stores origin context: `{"channel": origin_channel, "chat_id": origin_chat_id}`
4. Creates `asyncio.Task` via `asyncio.create_task(self._run_subagent(...))`
5. Stores task in `self._running_tasks[task_id]`
6. Registers cleanup callback: `bg_task.add_done_callback(lambda _: self._running_tasks.pop(task_id, None))`
7. Returns status string immediately

### 8.3 Subagent Execution

**File**: `nanobot/agent/subagent.py:86–173`

`_run_subagent(task_id, task, label, origin)`:

1. **Tool setup**: Creates a new `ToolRegistry` with a **reduced** tool set:
   - `ReadFileTool`, `WriteFileTool`, `ListDirTool` (no EditFileTool)
   - `ExecTool` (with same config as parent)
   - `WebSearchTool`, `WebFetchTool`
   - **Excluded**: `MessageTool`, `SpawnTool` (subagents cannot message users or spawn further subagents)

2. **System prompt**: Built by `_build_subagent_prompt(task)` (lines 207–236) — focused prompt with:
   - Task description
   - Rules (stay focused, no side tasks)
   - Capability list (read/write, exec, web)
   - Restriction list (no messaging, no spawning, no history access)
   - Workspace path

3. **Messages**: `[system_prompt, user_message(task)]` — no history

4. **Agent loop**: Same structure as main loop but with `max_iterations = 15`

5. **Result announcement**: On completion or error, calls `_announce_result()` which:
   - Constructs an `InboundMessage` with `channel="system"`, `sender_id="subagent"`, `chat_id="{origin_channel}:{origin_chat_id}"`
   - Publishes to `bus.publish_inbound(msg)`
   - This triggers the main agent loop to process the result

### 8.4 System Message Handling

**File**: `nanobot/agent/loop.py:226–316`

When the main agent loop receives a message with `channel == "system"`:

1. Parses origin from `chat_id` (format `"channel:chat_id"`)
2. Loads the **origin** session (not a separate system session)
3. Runs the full agent loop with the announce content as the user message
4. Saves result to session with prefix: `"[System: {sender_id}] {content}"`
5. Returns `OutboundMessage` to the original channel/chat_id

---

## 9. Postprocessing Pipeline (Response Processors)

### 9.1 No Response Processors

nanobot has **no postprocessing pipeline**. The LLM response is used directly:

1. `response.content` becomes `final_content`
2. `final_content` is saved to session as-is
3. `final_content` is placed into `OutboundMessage.content` as-is

### 9.2 No Output Schema / Structured Response

There is no `output_schema` mechanism, no JSON mode enforcement, and no structured response parsing. All responses are treated as free-text strings.

### 9.3 Response Delivery

**File**: `nanobot/agent/loop.py:220–224`

```python
return OutboundMessage(
    channel=msg.channel,
    chat_id=msg.chat_id,
    content=final_content
)
```

For the CLI path (`process_direct`), the `OutboundMessage.content` is extracted and returned as a string.

For the gateway path, the `OutboundMessage` is published to the bus outbound queue, dispatched by `ChannelManager._dispatch_outbound()` to the appropriate channel's `send()` method.

### 9.4 Channel-Specific Formatting

Formatting is handled at the channel level, not in the agent core:

- **Telegram** (`channels/telegram.py:16–76`): `_markdown_to_telegram_html()` converts markdown to Telegram-compatible HTML. Handles code blocks, inline code, headers, blockquotes, bold, italic, strikethrough, links, and bullet lists. Falls back to plain text on parse error.
- **WhatsApp** (`channels/whatsapp.py:75–89`): Sends raw text via WebSocket JSON `{"type": "send", "to": chat_id, "text": content}`
- **CLI**: Prints directly via `rich.Console`

---

## 10. Callback System

### 10.1 No Formal Callback System

nanobot implements **no formal callback/hook system**. There are no:
- `before_model_callback` / `after_model_callback`
- `before_tool_callback` / `after_tool_callback`
- Plugin callbacks
- Event listeners

### 10.2 Functional Callbacks (Ad-hoc)

The following callback-style patterns exist:

| Callback | Type | Location | Purpose |
|----------|------|----------|---------|
| `MessageTool._send_callback` | `Callable[[OutboundMessage], Awaitable[None]]` | `tools/message.py:14` | Send messages to bus |
| `CronService.on_job` | `Callable[[CronJob], Coroutine[..., str \| None]]` | `cron/service.py:48` | Execute cron job through agent |
| `HeartbeatService.on_heartbeat` | `Callable[[str], Coroutine[..., str]]` | `heartbeat/service.py:49` | Execute heartbeat through agent |
| `MessageBus._outbound_subscribers` | `dict[str, list[Callable]]` | `bus/queue.py:22` | Channel-specific outbound dispatch |

These are injection points set during construction, not a general-purpose callback system.

---

## 11. Multi-Agent Orchestration

### 11.1 No Sequential Agent Workflow

nanobot does not implement sequential agent pipelines or workflow orchestration. There is a single `AgentLoop` instance that processes all messages.

### 11.2 Subagent Isolation Model

Subagents are isolated via:
- **Separate `ToolRegistry`**: No `MessageTool` or `SpawnTool`
- **No session access**: Subagents start with a clean message list (system prompt + task only)
- **Separate system prompt**: Focused on the specific task
- **Same LLM provider**: Shares the `LLMProvider` instance with the parent
- **Same workspace**: Has filesystem access to the same workspace directory
- **Lower iteration limit**: 15 vs. 20 for the main agent

### 11.3 Concurrency Model

- Main agent loop processes messages **sequentially** from the inbound queue (one at a time)
- Subagents run as `asyncio.Task` instances **concurrently** with the main loop
- Multiple subagents can run concurrently (no limit enforced)
- Subagent results are injected back into the inbound queue as `channel="system"` messages, processed in FIFO order

### 11.4 No Agent State Management

There is no persistent agent state beyond the session history. No state machines, no resumable invocations, no checkpoint/restore mechanism.

---

## 12. Message Bus

### 12.1 Architecture

**File**: `nanobot/bus/queue.py`

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]    # Channels → Agent
    outbound: asyncio.Queue[OutboundMessage]   # Agent → Channels
    _outbound_subscribers: dict[str, list[Callable]]  # Per-channel callbacks
```

### 12.2 Inbound Flow

```
Channel._handle_message()
  → InboundMessage constructed
  → bus.publish_inbound(msg)         # puts on asyncio.Queue
  → AgentLoop.run() polls via bus.consume_inbound()
  → AgentLoop._process_message(msg)
```

### 12.3 Outbound Flow (Gateway Mode)

```
AgentLoop._process_message()
  → returns OutboundMessage
  → bus.publish_outbound(response)   # puts on asyncio.Queue
  → ChannelManager._dispatch_outbound() polls via bus.consume_outbound()
  → channel.send(msg)
```

### 12.4 Outbound Subscriber Model

`MessageBus` supports two dispatch patterns:
1. **Queue-based**: `consume_outbound()` returns the next message (used by `ChannelManager`)
2. **Subscriber-based**: `subscribe_outbound(channel, callback)` registers per-channel callbacks, dispatched by `dispatch_outbound()` (available but not used in the gateway setup — `ChannelManager` uses the queue approach instead)

---

## 13. Scheduled Tasks (Cron Service)

### 13.1 Job Types

**File**: `nanobot/cron/types.py`

Three schedule kinds:
- `"at"`: One-shot execution at a specific timestamp (ms)
- `"every"`: Recurring at fixed interval (ms)
- `"cron"`: Recurring via cron expression (parsed by `croniter`)

### 13.2 Job Execution

**File**: `nanobot/cron/service.py:216–247`

1. Job callback invoked: `await self.on_job(job)` → calls `agent.process_direct(job.payload.message, session_key=f"cron:{job.id}")`
2. On success: `job.state.last_status = "ok"`
3. On exception: `job.state.last_status = "error"`, `job.state.last_error = str(e)`
4. For `"at"` jobs: disabled after run (or deleted if `delete_after_run=True`)
5. For recurring jobs: next run recomputed via `_compute_next_run()`

### 13.3 Timer Mechanism

Uses `asyncio.create_task(tick())` where `tick()` sleeps until the next earliest job's `next_run_at_ms`. After each tick, re-arms with the next soonest time.

---

## 14. Heartbeat Service

**File**: `nanobot/heartbeat/service.py`

### 14.1 Algorithm

1. Sleeps for `interval_s` (default 1800s = 30 min)
2. Reads `workspace/HEARTBEAT.md`
3. If file is empty/missing or contains only headers/empty checkboxes: skips
4. Otherwise: calls `on_heartbeat(HEARTBEAT_PROMPT)` which invokes `agent.process_direct()`
5. If response contains `"HEARTBEATOK"` (case-insensitive, underscores removed): logs "OK"
6. Otherwise: logs "completed task"

---

## 15. Channel System

### 15.1 BaseChannel Interface

**File**: `nanobot/channels/base.py`

```python
class BaseChannel(ABC):
    name: str = "base"

    async def start(self) -> None: ...      # Connect and listen
    async def stop(self) -> None: ...       # Disconnect
    async def send(self, msg: OutboundMessage) -> None: ...  # Send message

    def is_allowed(self, sender_id: str) -> bool:
        # If allow_from is empty: allow everyone
        # Otherwise: check sender_id in allow_from list
        # Supports pipe-delimited IDs (e.g., "123|username")

    async def _handle_message(self, sender_id, chat_id, content, media, metadata) -> None:
        # Checks is_allowed(), then publishes InboundMessage to bus
```

### 15.2 Telegram Channel

**File**: `nanobot/channels/telegram.py`

- Uses `python-telegram-bot` library with long polling
- Handles text, photos, voice, audio, and documents
- Voice/audio transcription via Groq Whisper API
- Downloads media to `~/.nanobot/media/`
- Converts markdown responses to Telegram HTML

### 15.3 WhatsApp Channel

**File**: `nanobot/channels/whatsapp.py`

- Connects to Node.js bridge via WebSocket
- Bridge uses `@whiskeysockets/baileys` for WhatsApp Web protocol
- Auto-reconnects on disconnection (5s delay)
- Handles message types: `message`, `status`, `qr`, `error`

---

## Appendix A: Complete Request Processor Chain

nanobot has no formal request processor chain. The equivalent processing happens in `ContextBuilder.build_messages()`:

| Step | Operation | Mutates |
|------|-----------|---------|
| 1 | Build system prompt (identity + bootstrap + memory + skills) | Creates `messages[0]` |
| 2 | Append session history (last 50 messages, role+content only) | Extends `messages` |
| 3 | Append current user message (with optional base64 images) | Appends to `messages` |

---

## Appendix B: Event Schema Reference

### InboundMessage
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `channel` | `str` | required | Source channel identifier |
| `sender_id` | `str` | required | User identifier |
| `chat_id` | `str` | required | Chat/conversation identifier |
| `content` | `str` | required | Message text |
| `timestamp` | `datetime` | `datetime.now()` | Message timestamp |
| `media` | `list[str]` | `[]` | Local file paths for attached media |
| `metadata` | `dict[str, Any]` | `{}` | Channel-specific metadata |

### OutboundMessage
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `channel` | `str` | required | Target channel |
| `chat_id` | `str` | required | Target chat |
| `content` | `str` | required | Message text |
| `reply_to` | `str \| None` | `None` | Reply reference |
| `media` | `list[str]` | `[]` | Media attachments |
| `metadata` | `dict[str, Any]` | `{}` | Channel-specific metadata |

### Session
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `key` | `str` | required | `"channel:chat_id"` |
| `messages` | `list[dict]` | `[]` | `[{role, content, timestamp, ...}]` |
| `created_at` | `datetime` | `datetime.now()` | Creation time |
| `updated_at` | `datetime` | `datetime.now()` | Last update time |
| `metadata` | `dict[str, Any]` | `{}` | Session metadata |

---

## Appendix C: LLM Request/Response Schema Reference

### LLM Request (kwargs to `litellm.acompletion`)
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | `str` | required | Provider-prefixed model ID |
| `messages` | `list[dict]` | required | Conversation messages |
| `max_tokens` | `int` | `4096` | Max response tokens |
| `temperature` | `float` | `0.7` | Sampling temperature |
| `tools` | `list[dict]` | omitted if empty | Tool definitions in OpenAI format |
| `tool_choice` | `str` | `"auto"` | Only set when tools are present |
| `api_base` | `str` | omitted if None | Custom endpoint URL |

### LLMResponse
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `content` | `str \| None` | required | Text response |
| `tool_calls` | `list[ToolCallRequest]` | `[]` | Requested tool invocations |
| `finish_reason` | `str` | `"stop"` | Completion reason |
| `usage` | `dict[str, int]` | `{}` | Token usage stats |

### ToolCallRequest
| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Unique call identifier |
| `name` | `str` | Tool function name |
| `arguments` | `dict[str, Any]` | Parsed arguments |

---

## Appendix D: Configuration Schema Reference

**File**: `nanobot/config/schema.py`

```
Config (BaseSettings, env_prefix="NANOBOT_", env_nested_delimiter="__")
├── agents: AgentsConfig
│   └── defaults: AgentDefaults
│       ├── workspace: str = "~/.nanobot/workspace"
│       ├── model: str = "anthropic/claude-opus-4-5"
│       ├── max_tokens: int = 8192
│       ├── temperature: float = 0.7
│       └── max_tool_iterations: int = 20
├── channels: ChannelsConfig
│   ├── whatsapp: WhatsAppConfig
│   │   ├── enabled: bool = False
│   │   ├── bridge_url: str = "ws://localhost:3001"
│   │   └── allow_from: list[str] = []
│   └── telegram: TelegramConfig
│       ├── enabled: bool = False
│       ├── token: str = ""
│       └── allow_from: list[str] = []
├── providers: ProvidersConfig
│   ├── anthropic: ProviderConfig {api_key, api_base}
│   ├── openai: ProviderConfig
│   ├── openrouter: ProviderConfig
│   ├── groq: ProviderConfig
│   ├── zhipu: ProviderConfig
│   ├── vllm: ProviderConfig
│   └── gemini: ProviderConfig
├── gateway: GatewayConfig
│   ├── host: str = "0.0.0.0"
│   └── port: int = 18790
└── tools: ToolsConfig
    ├── web: WebToolsConfig
    │   └── search: WebSearchConfig
    │       ├── api_key: str = ""
    │       └── max_results: int = 5
    └── exec: ExecToolConfig
        ├── timeout: int = 60
        └── restrict_to_workspace: bool = False
```

**Config file**: `~/.nanobot/config.json` (camelCase keys, converted to snake_case on load)

**API key priority** (`get_api_key()`): OpenRouter > Anthropic > OpenAI > Gemini > Zhipu > Groq > vLLM

**API base resolution** (`get_api_base()`):
1. If OpenRouter key set: return OpenRouter base (default `https://openrouter.ai/api/v1`)
2. If Zhipu key set: return Zhipu base
3. If vLLM base set: return vLLM base
4. Otherwise: `None`

---

## Appendix E: Skills System Reference

### Skill Discovery Order
1. **Workspace skills** (`{workspace}/skills/{name}/SKILL.md`) — highest priority
2. **Built-in skills** (`nanobot/skills/{name}/SKILL.md`) — lower priority, skipped if workspace has same name

### Skill Metadata (YAML Frontmatter)
```yaml
---
description: "What this skill does"
always: true          # Include in every system prompt
metadata: '{"nanobot": {"requires": {"bins": ["git"], "env": ["GITHUB_TOKEN"]}, "always": true}}'
---
```

### Requirement Checking
- `bins`: Checks `shutil.which(binary)` for each required binary
- `env`: Checks `os.environ.get(var)` for each required environment variable
- Skills with unmet requirements are marked `available="false"` in the summary

### Progressive Loading
- **Always-on skills**: Full content included in system prompt
- **Available skills**: Only summary (name, description, path) included; agent uses `read_file` to load on demand

---

## Appendix F: Complete File Index

| File | Lines | Primary Class/Function |
|------|-------|----------------------|
| `nanobot/__init__.py` | 7 | `__version__`, `__logo__` |
| `nanobot/__main__.py` | 8 | Module entry point |
| `nanobot/cli/commands.py` | 656 | `app` (Typer), `onboard`, `agent`, `gateway`, `status` |
| `nanobot/config/schema.py` | 127 | `Config`, `AgentDefaults`, `ProviderConfig`, etc. |
| `nanobot/config/loader.py` | 96 | `load_config`, `save_config`, key conversion |
| `nanobot/agent/loop.py` | 338 | `AgentLoop` |
| `nanobot/agent/context.py` | 218 | `ContextBuilder` |
| `nanobot/agent/memory.py` | 110 | `MemoryStore` |
| `nanobot/agent/skills.py` | 229 | `SkillsLoader` |
| `nanobot/agent/subagent.py` | 241 | `SubagentManager` |
| `nanobot/agent/tools/base.py` | 124 | `Tool` (ABC) |
| `nanobot/agent/tools/registry.py` | 74 | `ToolRegistry` |
| `nanobot/agent/tools/filesystem.py` | 192 | `ReadFileTool`, `WriteFileTool`, `EditFileTool`, `ListDirTool` |
| `nanobot/agent/tools/shell.py` | 142 | `ExecTool` |
| `nanobot/agent/tools/web.py` | 164 | `WebSearchTool`, `WebFetchTool` |
| `nanobot/agent/tools/message.py` | 87 | `MessageTool` |
| `nanobot/agent/tools/spawn.py` | 66 | `SpawnTool` |
| `nanobot/bus/events.py` | 38 | `InboundMessage`, `OutboundMessage` |
| `nanobot/bus/queue.py` | 82 | `MessageBus` |
| `nanobot/channels/base.py` | 122 | `BaseChannel` (ABC) |
| `nanobot/channels/manager.py` | 140 | `ChannelManager` |
| `nanobot/channels/telegram.py` | 303 | `TelegramChannel` |
| `nanobot/channels/whatsapp.py` | 142 | `WhatsAppChannel` |
| `nanobot/cron/types.py` | 60 | `CronJob`, `CronSchedule`, `CronPayload`, `CronJobState` |
| `nanobot/cron/service.py` | 347 | `CronService` |
| `nanobot/heartbeat/service.py` | 131 | `HeartbeatService` |
| `nanobot/providers/base.py` | 70 | `LLMProvider` (ABC), `LLMResponse`, `ToolCallRequest` |
| `nanobot/providers/litellm_provider.py` | 174 | `LiteLLMProvider` |
| `nanobot/providers/transcription.py` | 66 | `GroqTranscriptionProvider` |
| `nanobot/session/manager.py` | 203 | `Session`, `SessionManager` |
| `nanobot/utils/helpers.py` | 92 | Utility functions |
