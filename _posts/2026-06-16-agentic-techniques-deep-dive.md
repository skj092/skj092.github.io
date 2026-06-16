---
layout: post
title: "Agentic Techniques Deep Dive: What I Learned from Reading Claude Code"
date: 2026-06-16
tags: [ai, agents, python, engineering]
---

I spent a few days reading through [openclaude](https://github.com/anthropics/claude-code) — the open-source TypeScript codebase behind Claude Code. My goal was selfish: extract every production-grade agentic technique I could find and understand them well enough to build them in Python.

This post is the result. Nine techniques, Python-ready pseudocode for each, and a build-order recommendation at the end. If you are building AI agents and want to go beyond the basic "call model → get response" loop, this is for you.

---

## 1. The Agent Loop (Core of Everything)

Every AI agent reduces to this pattern:

```
loop:
  1. Build context (messages + tools + system prompt)
  2. Call model → stream response
  3. If model emits tool_use → execute tools in parallel
  4. Append tool results to messages
  5. Go back to step 2
  6. If model emits stop_sequence or terminal condition → break
```

The key implementation detail: use **async generators** so every token flows to the UI in real time while the agent continues working.

```python
async def query_loop(messages, tools, model):
    while True:
        async for event in stream_from_model(messages, tools, model):
            yield event  # stream tokens to UI in real-time

            if event.type == "tool_use":
                pending_tools.append(event)

            if event.type == "message_stop":
                break

        if not pending_tools:
            break  # no tools called = agent is done

        tool_results = await execute_tools(pending_tools)
        messages.append(AssistantMessage(content=stream_content))
        messages.append(UserMessage(tool_results=tool_results))

        pending_tools = []
```

**Key insight:** The loop only terminates when the model makes a turn without calling any tools. Everything else is just deciding *when* to terminate and *what* to yield.

The codebase tracks six terminal conditions:

| Condition | Meaning |
|-----------|---------|
| `completed` | Normal finish |
| `max_turns` | Hit turn limit |
| `tool_failure_loop` | Same tool keeps failing |
| `prompt_too_long` | Context window exceeded |
| `model_error` | API error after retries |
| `blocking_limit` | Permission denied, needs human |

---

## 2. Streaming Tool Execution with Concurrency

This is the most underrated technique. Instead of waiting for the full model response before running tools, you can run tools **as the model is still streaming**:

```
Model streaming:   [text...] [tool_A_start] [text...] [tool_B_start] [stop]
Execution:                   [tool_A runs immediately]  [tool_B runs immediately]
Results:                     [tool_A done]              [tool_B done]
```

Two categories of tools:
- **Concurrency-safe** (read-only): Read, Glob, Grep — run in parallel
- **Exclusive tools**: Bash, Edit, Write — acquire a lock, run one at a time

```python
import asyncio

async def streaming_tool_executor(model_stream, tools):
    lock = asyncio.Lock()
    pending = []

    async for event in model_stream:
        if event.type == "tool_use":
            tool = get_tool(event.name)
            if tool.is_concurrency_safe:
                task = asyncio.create_task(tool.execute(event.input))
            else:
                async def with_lock():
                    async with lock:
                        return await tool.execute(event.input)
                task = asyncio.create_task(with_lock())
            pending.append((event.id, task))

    results = []
    for tool_id, task in pending:
        result = await task
        results.append(ToolResult(id=tool_id, content=result))

    return results
```

On a turn that calls 5 read-only tools, you save 4x latency because they all run simultaneously while the model is still writing its response.

---

## 3. Tool Failure Loop Guard

A common failure mode: the model keeps calling the same tool that keeps failing. Without a guard, you burn tokens forever.

```python
class ToolFailureLoopGuard:
    def __init__(self, max_consecutive=3):
        self.consecutive_failures = 0
        self.last_failed_tool = None
        self.max = max_consecutive

    def record(self, tool_name, success):
        if not success:
            if tool_name == self.last_failed_tool:
                self.consecutive_failures += 1
            else:
                self.consecutive_failures = 1
                self.last_failed_tool = tool_name
        else:
            self.consecutive_failures = 0
            self.last_failed_tool = None

    def should_abort(self):
        return self.consecutive_failures >= self.max
```

Simple but essential. This small class prevents a stuck agent from running indefinitely and surfaces the real error to the user.

---

## 4. Token Budget with Diminishing Returns Detection

When an agent is deep in a long loop, it may keep "continuing" but making less and less progress. This pattern detects that by watching how many new tokens are added each turn:

```python
class TokenBudgetTracker:
    THRESHOLD = 0.90          # 90% of context used
    MIN_DELTA = 500           # meaningful progress threshold
    DIMINISHING_RUNS = 3      # how many bad runs before giving up

    def __init__(self, context_window):
        self.context_window = context_window
        self.continuation_count = 0
        self.bad_run_count = 0

    def check(self, tokens_used, tokens_added_this_turn):
        usage_pct = tokens_used / self.context_window

        if usage_pct < self.THRESHOLD:
            return {"action": "continue"}

        if tokens_added_this_turn < self.MIN_DELTA:
            self.bad_run_count += 1
        else:
            self.bad_run_count = 0

        self.continuation_count += 1

        if self.bad_run_count >= self.DIMINISHING_RUNS:
            return {"action": "stop", "reason": "diminishing_returns"}

        return {
            "action": "warn_user",
            "message": f"Context {usage_pct:.0%} full. Continuing..."
        }
```

---

## 5. Context Compaction (Keeping Context Fresh)

When conversation history gets too long, you can't just truncate — you lose context. The answer is **semantic compaction**: summarize old messages while keeping recent ones verbatim.

Three modes in the codebase:

| Mode | Trigger | Use case |
|------|---------|----------|
| **Auto-compact** | `tokens_used > threshold` at turn start | Proactive |
| **Reactive-compact** | `prompt_too_long` API error mid-stream | Recovery |
| **Micro-compact** | Lightweight, within a single turn | Fine-grained |

```python
async def compact_messages(messages, model, keep_recent_n=10):
    if len(messages) <= keep_recent_n:
        return messages

    old_messages = messages[:-keep_recent_n]
    recent_messages = messages[-keep_recent_n:]

    summary = await model.complete(
        system="You are a context compaction assistant.",
        messages=[
            *old_messages,
            {"role": "user", "content": "Summarize the above conversation "
             "preserving all key decisions, facts, and current task state."}
        ]
    )

    return [
        {"role": "user", "content": f"[Previous context summary]: {summary}"},
        *recent_messages
    ]
```

The reactive variant is what makes this production-grade: if you get a `context_length_exceeded` error mid-turn, you compact *right then* and retry without losing the user's current request.

---

## 6. Prefetch While Streaming (Latency Hiding)

While the model is generating output, you can be doing useful work in the background. The codebase does this for loading relevant memories, discovering skill files, and prefetching MCP resources.

```python
async def query_with_prefetch(messages, tools, model, memory_store):
    # Start memory search BEFORE awaiting the model
    memory_task = asyncio.create_task(
        memory_store.find_relevant(messages[-1])
    )

    full_response = ""
    async for token in model.stream(messages, tools):
        full_response += token
        yield token  # stream to UI immediately

    # By the time model finishes, memory search is likely done
    relevant_memories = await memory_task

    if relevant_memories:
        messages.append(attach_memories(relevant_memories))
```

`asyncio.create_task()` is your friend here. Fire and forget, collect the result when you need it.

---

## 7. Multi-Agent Orchestration

This is what makes it a *multi-agent* system rather than a single agent. The model can spawn child agents with isolated query loops. There are five sub-patterns worth knowing:

**a) Agent routing — different agents on different models**

```python
AGENT_ROUTING = {
    "Explore": "claude-haiku-4-5",    # fast, cheap for search
    "Plan": "claude-opus-4-8",         # slow, smart for planning
    "code-reviewer": "claude-sonnet-4-6",
}

def get_model_for_agent(agent_type, default_model):
    return AGENT_ROUTING.get(agent_type, default_model)
```

**b) Worktree isolation — parallel agents that write files don't conflict**

```bash
# Each file-writing agent gets its own git worktree
git worktree add /tmp/agent-xyz-worktree HEAD
# Agent runs in isolation; changes merged back or discarded on completion
```

**c) Structured output — force parseable responses from subagents**

Instead of parsing free-form text, instruct the subagent to call a `StructuredOutput` tool with a JSON schema. The schema is validated at the tool-call layer, so the model retries on mismatch automatically.

```python
BUGS_SCHEMA = {
    "type": "object",
    "properties": {
        "bugs": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "file": {"type": "string"},
                    "line": {"type": "integer"},
                    "description": {"type": "string"},
                    "severity": {"type": "string", "enum": ["low", "medium", "high"]}
                }
            }
        }
    }
}

result = await run_agent(prompt, output_schema=BUGS_SCHEMA)
bugs = result["bugs"]  # guaranteed to match schema
```

**d) Background execution — auto-background long-running agents**

```python
async def run_agent_with_auto_background(prompt, timeout_s=120):
    task = asyncio.create_task(run_agent(prompt))
    try:
        result = await asyncio.wait_for(asyncio.shield(task), timeout=timeout_s)
        return result
    except asyncio.TimeoutError:
        task_id = generate_id()
        background_tasks[task_id] = task
        return TaskHandle(id=task_id, status="running")
```

**e) Pipeline orchestration — no barrier between stages**

```python
async def pipeline(items, *stages):
    """Run each item through all stages — no barrier between stages.
    Item A can be in stage 3 while item B is still in stage 1."""
    tasks = [run_item_through_stages(item, stages) for item in items]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if not isinstance(r, Exception)]

async def run_item_through_stages(item, stages):
    result = item
    for stage in stages:
        result = await stage(result)
    return result

# Review code across multiple dimensions in parallel
DIMENSIONS = ["bugs", "security", "performance"]
results = await pipeline(
    DIMENSIONS,
    lambda dim: run_agent(f"Find {dim} issues in this PR", schema=FINDINGS_SCHEMA),
    lambda findings: run_agent(f"Verify these findings: {findings}", schema=VERDICT_SCHEMA),
)
```

---

## 8. The Hooks System (Extensible Lifecycle Events)

Hooks let external processes intercept agent behavior at lifecycle points without touching the agent code.

```
UserPromptSubmit → PreToolUse → PostToolUse → Stop → SessionEnd
```

Each hook is a subprocess: configure a shell command, agent serializes event data as JSON, pipes it to stdin, reads stdout for the response.

```python
import subprocess, json, asyncio

async def fire_hook(hook_type, event_data, registered_hooks):
    hooks_for_type = registered_hooks.get(hook_type, [])
    results = []

    for hook_cmd in hooks_for_type:
        proc = await asyncio.create_subprocess_shell(
            hook_cmd,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
        )
        stdout, _ = await proc.communicate(
            input=json.dumps(event_data).encode()
        )
        if stdout:
            results.append(json.loads(stdout))

    return results

async def can_use_tool(tool_name, tool_input):
    results = await fire_hook("PreToolUse", {
        "tool_name": tool_name,
        "tool_input": tool_input
    }, hooks)

    for r in results:
        if r.get("action") == "block":
            return False, r.get("reason")
    return True, None
```

Configuration in `settings.json`:
```json
{
  "hooks": {
    "PreToolUse": ["python3 my_security_check.py"],
    "PostToolUse": ["./log_tool_usage.sh"],
    "Stop": ["python3 notify_slack.py"]
  }
}
```

---

## 9. MCP Integration (Model Context Protocol)

MCP is a standard for connecting external tools to agents as a plugin system. Instead of hardcoding tools, you connect to MCP servers that expose tools dynamically.

```
Agent ←→ MCP Client ←→ MCP Server (subprocess or HTTP)
                              ↓
                      tools, resources, prompts
```

| Transport | Description |
|-----------|-------------|
| `stdio` | Subprocess communicating over stdin/stdout (most common) |
| `sse` | HTTP Server-Sent Events |
| `http` | HTTP with streaming |

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def connect_mcp_server(command, args):
    server_params = StdioServerParameters(command=command, args=args)
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Discover tools dynamically
            tools = await session.list_tools()

            result = await session.call_tool("my_tool", {"arg": "value"})
            return result
```

**Key insight:** Tools from MCP servers are surfaced to the model exactly like built-in tools. The model cannot tell the difference. This means you can add capabilities at runtime without changing your agent code.

---

## How It All Fits Together

```
User Input
    │
    ▼
REPL (terminal UI)
    │
    ▼
query() ─────────────── prefetch memories (asyncio.create_task)
    │                            │
    │   Build context:           │ (runs in parallel with model call)
    │   system prompt            │
    │   + memories   ◄───────────┘
    │   + messages
    │
    ▼
Model API (streaming)
    │
    ├── token ──────────────────► yield to UI (real-time display)
    │
    ├── tool_use ───────────────► StreamingToolExecutor
    │                                ├── concurrent-safe → asyncio.gather()
    │                                └── exclusive → serialized with Lock
    │
    ▼
Tool results → append to messages → loop back to model
    │
    ├── ToolFailureLoopGuard checks → abort if stuck
    ├── TokenBudgetTracker checks → compact or stop if full
    │
    ▼ (no tools called OR terminal condition)
Final response
    │
    ▼
PostTurn hooks → done
```

---

## What to Build First (Priority Order)

| # | Technique | Python libs | Effort | Impact |
|---|-----------|-------------|--------|--------|
| 1 | Agent loop with streaming | `anthropic`, `asyncio` | Low | High |
| 2 | Parallel tool execution | `asyncio.gather()` | Low | High |
| 3 | Tool failure loop guard | Pure Python | Low | Medium |
| 4 | Token budget tracker | Pure Python | Low | Medium |
| 5 | Context compaction | `anthropic` | Medium | High |
| 6 | Prefetch while streaming | `asyncio.create_task` | Low | Medium |
| 7 | Multi-agent routing | `anthropic` | Medium | High |
| 8 | Hooks system | `subprocess`, `asyncio` | Medium | Medium |
| 9 | MCP integration | `mcp` (official SDK) | Medium | High |

**Recommended build order:**

- **Phase 1 (single agent):** Items 1 + 2 + 3 + 4 — production-quality single agent
- **Phase 2 (multi-agent):** Add items 5 + 7 — full multi-agent behavior
- **Phase 3 (extensible):** Add items 6 + 8 + 9 — plugin system and latency optimization

The first four items are low-effort and high-impact. If you only take one thing from this post: implement the streaming tool executor. The latency savings on tool-heavy turns are real and immediately visible.
